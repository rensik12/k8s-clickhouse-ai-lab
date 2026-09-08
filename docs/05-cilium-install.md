# 05. Cilium Installation

## 목표

Kubernetes v1.36.4 클러스터에 Cilium v1.20.1을 설치하고 다음 기능을 동시에 활성화한다.

- kube-proxy replacement
- eBPF masquerading
- Egress Gateway
- Cluster Pool IPAM
- VXLAN tunnel mode

## 네트워크 값

```text
Node Network  : 192.168.184.0/24
Pod CIDR      : 10.200.0.0/16
Service CIDR  : 10.96.0.0/12
API Server    : 192.168.184.235:6443
```

Cilium은 Pod CIDR `10.200.0.0/16`을 노드별 `/24` 단위로 할당한다.

## 1. Worker / Egress Node Join

`lab-w1`, `lab-e`에서 `kubeadm init` 시 출력된 Worker Join 명령을 실행한다.

실제 token과 discovery hash는 공개 저장소에 기록하지 않는다.

```bash
kubeadm join 192.168.184.235:6443 \
  --token <REDACTED> \
  --discovery-token-ca-cert-hash sha256:<REDACTED>
```

`lab-m`에서 확인:

```bash
kubectl get nodes -o wide
```

CNI 설치 전에는 세 노드 모두 또는 Worker/Egress 노드가 `NotReady` 상태일 수 있다.

## 2. Helm 설치

`lab-m`에서 Helm이 없으면 설치한다.

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

Cilium Helm Repository 추가:

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
```

## 3. Cilium values.yaml

저장소의 `kubernetes/cilium/values.yaml`을 사용한다.

```yaml
kubeProxyReplacement: true

k8sServiceHost: 192.168.184.235
k8sServicePort: "6443"

routingMode: tunnel
tunnelProtocol: vxlan

bpf:
  masquerade: true

egressGateway:
  enabled: true

ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
      - 10.200.0.0/16
    clusterPoolIPv4MaskSize: 24
```

## 4. Cilium 설치

`lab-m`에서:

```bash
helm install cilium cilium/cilium \
  --version 1.20.1 \
  --namespace kube-system \
  -f kubernetes/cilium/values.yaml
```

서버에 저장소를 clone하지 않은 경우 동일한 values 파일을 `/root/cilium-values.yaml`로 작성한 뒤 다음처럼 실행할 수 있다.

```bash
helm install cilium cilium/cilium \
  --version 1.20.1 \
  --namespace kube-system \
  -f /root/cilium-values.yaml
```

## 5. 설치 상태 확인

```bash
kubectl -n kube-system get pods -o wide
kubectl get nodes -o wide
kubectl get ciliumnodes
```

기대 상태:

```text
lab-m    Ready
lab-w1   Ready
lab-e    Ready
```

CoreDNS도 `Running` 상태로 전환되어야 한다.

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns
```

## 6. Cilium 상태 확인

Cilium Agent 내부에서 상태를 확인한다.

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg status
```

kube-proxy replacement 확인:

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg status --verbose | grep -i kubeproxy
```

Egress Gateway 설정 확인:

```bash
kubectl -n kube-system get cm cilium-config -o yaml | \
  grep -E 'enable-egress-gateway|enable-bpf-masquerade|kube-proxy-replacement'
```

## 7. Pod CIDR 할당 확인

```bash
kubectl get ciliumnodes \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.ipam.podCIDRs}{"\n"}{end}'
```

각 노드에 `10.200.0.0/16` 범위의 `/24`가 할당되는지 확인한다.

예시:

```text
lab-m    ["10.200.0.0/24"]
lab-w1   ["10.200.1.0/24"]
lab-e    ["10.200.2.0/24"]
```

실제 할당 순서는 달라질 수 있다.

## 8. Egress Node 역할 지정

전체 노드와 Cilium이 정상 상태가 된 뒤 `lab-e`를 전용 Egress Node로 지정한다.

```bash
kubectl label node lab-e node-role=egress
kubectl taint node lab-e dedicated=egress:NoSchedule
```

확인:

```bash
kubectl get node lab-e --show-labels
kubectl describe node lab-e | grep -A2 Taints
```

Cilium DaemonSet은 모든 taint를 tolerate하도록 구성되어 있으므로 `lab-e`에도 Cilium Agent는 계속 실행된다.

## 9. Bootstrap Token 관리

Worker와 Egress Node Join이 끝난 뒤 불필요한 bootstrap token은 확인 후 삭제하거나 만료되도록 둔다.

```bash
kubeadm token list
```

필요 시:

```bash
kubeadm token delete <TOKEN_ID>
```

## 완료 기준

- [ ] `lab-w1` Join
- [ ] `lab-e` Join
- [ ] Helm 설치
- [ ] Cilium 1.20.1 설치
- [ ] Cilium Agent / Operator Running
- [ ] CoreDNS Running
- [ ] 전체 Node Ready
- [ ] kube-proxy replacement 확인
- [ ] Pod CIDR `10.200.0.0/16` 할당 확인
- [ ] Egress Gateway 활성화 확인
- [ ] `lab-e` label / taint 적용
