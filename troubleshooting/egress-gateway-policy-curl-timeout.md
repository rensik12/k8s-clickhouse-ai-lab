# Egress Gateway 정책 적용 후 외부 통신 Timeout

## 상태

조사 진행 중

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

`CiliumEgressGatewayPolicy` 적용 후 동일 명령이 응답 없이 대기한다.

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

### Pod / Gateway 선택

```text
egress-test
  Pod IP : 10.200.1.143
  Node   : lab-w1

lab-e
  egress-node=true
```

### lab-w1 Cilium BPF Egress Map

```text
Source IP      Destination CIDR   Egress IP   Gateway IP        Egress Ifindex
10.200.1.143   0.0.0.0/0          0.0.0.0     192.168.184.179   0
```

Source Node에서 Gateway가 `lab-e(192.168.184.179)`로 정상 선택되었다.

### lab-e Cilium BPF Egress Map

```text
Source IP      Destination CIDR   Egress IP         Gateway IP        Egress Ifindex
10.200.1.143   0.0.0.0/0          192.168.184.179   192.168.184.179   2
```

Gateway Node에서도 Egress IP와 Interface가 정상 선택되었다.

### lab-e Interface / Routing

```text
ifindex 2 = eth0
192.168.184.179 -> 1.1.1.1 via 192.168.184.1 dev eth0
```

### lab-e tcpdump

`lab-w1 -> lab-e` 방향의 Egress 테스트 패킷은 관찰되지 않았다.

관찰된 UDP/8472 패킷은 반대 방향인 `lab-e -> lab-w1` Cilium 트래픽이었다.

따라서 현재까지의 증거로는 다음 단계 중 하나에서 문제가 발생하는 것으로 범위를 좁혔다.

```text
Pod
  -> lab-w1 Cilium Egress Policy match      [정상]
  -> Gateway lab-e 선택                     [정상]
  -> lab-w1에서 Gateway 방향 패킷 송신      [확인 필요]
  -> OpenStack network
  -> lab-e 수신                             [현재 관찰 안 됨]
  -> lab-e SNAT / 외부 송신                 [아직 미검증]
```

## 다음 확인

`lab-w1`에서 테스트 요청을 발생시키면서 VXLAN 송신 여부와 Cilium Drop을 확인한다.

```bash
tcpdump -ni eth0 -nnvv 'udp port 8472 and host 192.168.184.179'
```

동시에 Cilium Agent에서 Drop monitor를 실행한다.

```bash
kubectl -n kube-system exec cilium-mjs56 -- cilium-dbg monitor --type drop
```

다른 터미널에서:

```bash
kubectl exec egress-test -- curl -k -m 5 https://1.1.1.1/cdn-cgi/trace
```

### 판단 기준

```text
1. lab-w1에서 UDP/8472 송신 자체가 없음
   -> Source Node Cilium datapath / BPF forwarding 확인

2. lab-w1에서는 lab-e 방향 UDP/8472 송신 확인
   + lab-e에는 도착하지 않음
   -> OpenStack Security Group / Port Security / Network ACL 확인

3. lab-e까지 도착
   + eth0 외부 패킷이 없음
   -> Gateway Node Egress/SNAT datapath 확인

4. lab-e eth0 외부 송신은 있으나 응답 없음
   -> OpenStack Floating IP / NAT / 외부 Routing 확인
```

## 현재 결론

정책 selector와 Gateway 선택은 정상이다. 아직 `lab-w1 -> lab-e` 실제 datapath 구간의 송신/전달 여부가 확인되지 않았으므로 OpenStack 또는 Cilium 중 어느 쪽이 원인이라고 확정하지 않는다.
