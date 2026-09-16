# Egress Gateway 정책 적용 후 외부 통신 Timeout

## 상태

해결 완료

## 환경

```text
Kubernetes : v1.36.4
Cilium     : v1.20.1
Pod        : egress-test / 10.200.1.143 / lab-w1
Source Node: lab-w1 / 192.168.184.163
Gateway    : lab-e / 192.168.184.179
Floating IP: 211.47.73.206
```

## 증상

정책 적용 전 테스트 Pod의 외부 Source IP는 정상적으로 `211.47.70.204`로 확인되었다.

```bash
kubectl exec egress-test -- curl -4 -s ifconfig.me
```

`CiliumEgressGatewayPolicy` 적용 직후 동일 명령이 응답 없이 대기했다.

## Egress Policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
metadata:
  name: egress-test-policy
spec:
  selectors:
  - podSelector:
      matchLabels:
        app: egress-test
  destinationCIDRs:
  - 0.0.0.0/0
  egressGateway:
    nodeSelector:
      matchLabels:
        egress-node: "true"
    interface: eth0
```

## 확인 결과

### BPF Egress Map

`lab-w1`:

```text
Source IP      Destination CIDR   Egress IP   Gateway IP        Egress Ifindex
10.200.1.143   0.0.0.0/0          0.0.0.0     192.168.184.179   0
```

`lab-e`:

```text
Source IP      Destination CIDR   Egress IP         Gateway IP        Egress Ifindex
10.200.1.143   0.0.0.0/0          192.168.184.179   192.168.184.179   2
```

정책 selector, Gateway 선택, Egress IP/Interface 선택은 모두 정상이다.

### lab-e Interface / Routing

```text
ifindex 2 = eth0
192.168.184.179 -> 1.1.1.1 via 192.168.184.1 dev eth0
```

### 실제 패킷 경로

`lab-w1`에서는 테스트 Pod의 외부 패킷이 VXLAN UDP/8472로 `lab-e`에 전달되는 것을 확인했다.

```text
192.168.184.163:<ephemeral> > 192.168.184.179.8472
inner packet: 10.200.1.143:<ephemeral> > 1.1.1.1.443 SYN
```

`lab-e`에서도 동일 패킷이 수신되고, Cilium이 Pod IP를 Egress Node IP로 SNAT한 뒤 외부로 송신하는 것을 확인했다.

```text
IN  : 10.200.1.143:<ephemeral> > 1.1.1.1.443
OUT : 192.168.184.179:<ephemeral> > 1.1.1.1.443
```

따라서 Cilium Egress Gateway datapath 자체는 정상 동작했다.

## Root Cause

OpenStack Security Group에서 Kubernetes 노드 간 VXLAN 통신에 필요한 `UDP/8472`가 충분히 허용되지 않은 상태였다.

`192.168.184.0/24` 구간의 UDP/8472 통신을 허용한 뒤 Egress Gateway 정책 대상 Pod의 외부 통신이 정상화됐다.

Cilium VXLAN overlay는 노드 간 캡슐화에 UDP/8472를 사용하므로, underlay firewall / Security Group에서 해당 포트를 허용해야 한다.

## 검증 결과

정책 적용 전:

```text
egress-test -> 211.47.70.204
```

정책 적용 후:

```bash
kubectl exec egress-test -- curl -4 -s ifconfig.me ; echo
```

결과:

```text
211.47.73.206
```

최종 경로:

```text
egress-test Pod (10.200.1.143)
  -> lab-w1 Cilium
  -> VXLAN UDP/8472
  -> lab-e (192.168.184.179)
  -> Cilium SNAT
  -> OpenStack Floating IP / NAT
  -> 211.47.73.206
```

## 조사 중 잘못된 가설

중간 조사에서 `lab-e`에서 테스트 VXLAN 패킷이 보이지 않아 OpenStack Security Group 또는 VXLAN ingress 차단을 의심했다. 이후 정밀 tcpdump를 통해 일반 Cilium health traffic과 실제 Egress 테스트 패킷을 구분했고, `lab-w1 -> lab-e` VXLAN 및 `lab-e` SNAT 모두 정상 동작하는 것을 확인했다.

또한 테스트 대상으로 사용한 `1.1.1.1:443`은 해당 환경에서 원래 통신이 되지 않는 목적지였으므로, 이 테스트의 timeout만으로 Egress Gateway 장애를 판단하면 안 됐다. 최종 검증에는 정책 적용 전 정상 통신을 확인했던 `ifconfig.me`를 사용했다.

## 배운 점

- Cilium VXLAN overlay 사용 시 노드 간 `UDP/8472` 허용 여부를 사전 점검해야 한다.
- BPF Egress Map은 정책 매칭과 Gateway 선택을 확인하는 데 유용하다.
- tcpdump로 outer VXLAN packet과 inner Pod packet을 함께 확인하면 실제 datapath를 단계별로 검증할 수 있다.
- 네트워크 장애 테스트는 반드시 '원래 정상 통신되는 목적지'를 기준으로 해야 한다.
- 특정 구간을 의심하더라도 패킷 캡처로 송신/수신/SNAT 단계를 순서대로 검증한 뒤 Root Cause를 확정해야 한다.
