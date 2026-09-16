# Egress Gateway 적용 후 외부 통신 timeout

## 증상

`egress-test` Pod는 정책 적용 전 외부 통신이 정상이며 공인 IP `211.47.70.204`가 확인되었다.

`CiliumEgressGatewayPolicy` 적용 후 `curl`이 timeout 상태가 되었다.

## 확인 결과

테스트 Pod:

```text
Pod IP : 10.200.1.143
Node   : lab-w1
```

Egress Gateway 정책은 정상적으로 `lab-e`를 선택했다.

`lab-w1` Cilium BPF egress map:

```text
Source IP      Destination CIDR   Egress IP   Gateway IP        Egress Ifindex
10.200.1.143   0.0.0.0/0          0.0.0.0     192.168.184.179   0
```

`lab-e` Cilium BPF egress map:

```text
Source IP      Destination CIDR   Egress IP         Gateway IP        Egress Ifindex
10.200.1.143   0.0.0.0/0          192.168.184.179   192.168.184.179   2
```

`lab-e`의 ifindex 2는 `eth0`이며 외부 기본 경로도 정상이다.

```text
1.1.1.1 from 192.168.184.179 via 192.168.184.1 dev eth0
```

## 패킷 캡처

`lab-w1`에서 정책 대상 Pod가 외부 `1.1.1.1:443`으로 요청했을 때 다음 VXLAN 패킷이 실제로 송신되는 것을 확인했다.

```text
192.168.184.163:<ephemeral> > 192.168.184.179.8472
inner packet: 10.200.1.143:<ephemeral> > 1.1.1.1.443 SYN
```

즉 Cilium source datapath는 정책에 따라 `lab-e`를 Gateway로 선택하고, 외부 패킷을 VXLAN UDP/8472로 캡슐화하여 `lab-e`로 전송하고 있다.

반면 같은 시점 `lab-e` tcpdump에서는 `lab-w1 -> lab-e` 방향 UDP/8472 패킷이 확인되지 않았다.

## 현재 판단

현재까지 확인된 사실 기준으로 문제 구간은 다음과 같이 좁혀졌다.

```text
egress-test Pod
      |
      v
lab-w1 Cilium
      |
      | VXLAN UDP/8472 송신 확인
      v
OpenStack Underlay / Security Group
      |
      X  lab-e에서 ingress 미확인
      v
lab-e Cilium Egress Gateway
```

따라서 1순위 확인 대상은 OpenStack Security Group 또는 노드 간 UDP/8472 ingress 정책이다.

Cilium VXLAN overlay 사용 시 모든 Cilium 노드 사이에 UDP/8472 통신이 허용되어야 한다.

## 다음 확인

- lab-e가 사용하는 OpenStack Security Group의 ingress rule 확인
- `192.168.184.0/24 -> UDP/8472` 또는 Kubernetes Node SG self-reference 허용 여부 확인
- 세 노드 상호 간 UDP/8472 허용 여부 확인
- 필요 시 TCP/4240 및 ICMP health check rule도 함께 점검
- Rule 보완 후 lab-e tcpdump에서 `192.168.184.163 -> 192.168.184.179:8472` 확인
- 이후 Pod 외부 Source IP가 `211.47.73.206`으로 변경되는지 재검증

## 비고

원인은 아직 최종 확정하지 않는다. OpenStack Security Group 설정을 확인한 뒤 확정한다.
