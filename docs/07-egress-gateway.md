# 07. Cilium Egress Gateway

## 목표

`lab-e`를 전용 Egress Gateway 노드로 지정하고, `lab-w1`에서 실행되는 테스트 Pod의 외부 트래픽이 `lab-e`를 경유한 뒤 OpenStack Floating IP `211.47.73.206`으로 외부에 노출되는지 검증한다.

## 현재 노드 상태

```text
lab-m   Ready   control-plane   192.168.184.235
lab-w1  Ready   worker          192.168.184.163
lab-e   Ready   egress          192.168.184.179
```

`lab-e`의 외부 통신은 OpenStack Floating IP/NAT를 통해 `211.47.73.206`으로 노출된다.

## 1. Cilium 기능 재검증

```bash
helm get values cilium -n kube-system

kubectl -n kube-system get cm cilium-config -o yaml | \
  egrep 'enable-egress-gateway|enable-bpf-masquerade|kube-proxy-replacement'

kubectl -n kube-system exec ds/cilium -- cilium-dbg status
```

확인 포인트:

```text
kube-proxy replacement: enabled
BPF masquerading      : enabled
Egress Gateway        : enabled
```

## 2. Egress 노드 역할 고정

```bash
kubectl label node lab-e egress-node=true --overwrite
kubectl taint node lab-e dedicated=egress:NoSchedule
```

확인:

```bash
kubectl get node lab-e --show-labels
kubectl describe node lab-e | grep -A3 Taints
```

## 3. 테스트 Pod를 Worker에 고정

테스트 Pod가 Egress 노드 자체에서 실행되면 경로 검증 의미가 줄어들기 때문에 `lab-w1`에 고정한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: egress-test
  namespace: default
  labels:
    app: egress-test
spec:
  nodeName: lab-w1
  containers:
  - name: curl
    image: curlimages/curl:latest
    command: ["sleep", "36000"]
```

## 4. 정책 적용 전 외부 Source IP 확인

```bash
kubectl exec egress-test -- curl -4 -s ifconfig.me ; echo
```

정책 적용 전에는 `lab-w1`의 외부 NAT 경로에 따른 공인 IP가 확인될 수 있다.

## 5. CiliumEgressGatewayPolicy

Cilium Egress Gateway는 정책에서 선택한 Pod의 외부 IPv4 트래픽을 지정된 Gateway Node로 전달하고 해당 노드의 Egress IP로 SNAT한다.

`lab-e`의 `eth0` 주소 `192.168.184.179`을 사용하고, 상위 OpenStack NAT를 통해 최종적으로 `211.47.73.206`으로 외부에 표시되는 구조를 검증한다.

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
  - "0.0.0.0/0"
  egressGateway:
    nodeSelector:
      matchLabels:
        egress-node: "true"
    interface: eth0
```

`interface`와 `egressIP`은 동시에 지정하지 않는다. `interface: eth0`을 지정하면 해당 인터페이스의 첫 IPv4 주소인 `192.168.184.179`이 Egress IP로 사용된다.

## 6. 정책 적용 후 검증

```bash
kubectl apply -f egress-policy.yaml

kubectl exec egress-test -- curl -4 -s ifconfig.me ; echo
```

목표 결과:

```text
211.47.73.206
```

Cilium BPF Egress Map 확인:

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg bpf egress list
```

확인 포인트:

```text
Source IP        : egress-test Pod IP
Destination CIDR : 0.0.0.0/0
Gateway IP       : 192.168.184.179
Egress IP        : 192.168.184.179 (Gateway Node에서 확인)
```

## 7. 장애 테스트

Egress Gateway 장애 시 정책 대상 트래픽의 영향도를 확인한다.

예정 시나리오:

1. 정상 상태에서 외부 IP `211.47.73.206` 확인
2. `lab-e` 또는 해당 노드의 Cilium Agent 중지
3. 테스트 Pod에서 외부 연결 실패/지연 확인
4. 복구 후 연결 정상화 확인
5. 단일 Egress 노드 구조의 SPOF를 정리하고 다중 Gateway 확장 방안 검토

## 면접 포인트

- Egress Node를 분리하는 이유
- Worker 증설 시 외부 ACL IP를 반복 등록하지 않아도 되는 이유
- Pod IP가 아니라 고정 Egress IP를 사용하는 이유
- Cilium Egress Gateway가 Pod 트래픽을 어떤 기준으로 Gateway Node에 전달하는지
- 단일 Egress Gateway의 장애 영향과 다중 Gateway 설계 시 고려사항
