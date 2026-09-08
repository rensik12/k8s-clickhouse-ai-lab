# 04. Control Plane Bootstrap

## 목표

`lab-m`을 Kubernetes v1.36.4 Control Plane으로 초기화하고, kube-proxy 대신 Cilium 1.20.1의 eBPF kube-proxy replacement를 사용한다.

## 네트워크 설계

```text
Node Network  : 192.168.184.0/24
Pod CIDR      : 10.10.0.0/16
Service CIDR  : 10.96.0.0/12
API Server    : 192.168.184.235:6443
```

Pod CIDR은 Cilium cluster-pool IPAM에서 명시적으로 관리한다.

## 1. cri-tools 보완

세 노드 공통:

```bash
dnf install -y cri-tools --disableexcludes=kubernetes

cat > /etc/crictl.yaml <<'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

crictl --version
crictl info
```

## 2. kubeadm 사전 점검

`lab-m`에서:

```bash
kubeadm config images list --kubernetes-version v1.36.4
```

선택적으로 이미지를 미리 Pull한다.

```bash
kubeadm config images pull \
  --kubernetes-version v1.36.4 \
  --cri-socket unix:///run/containerd/containerd.sock
```

## 3. Control Plane 초기화

Cilium의 kube-proxy replacement를 사용하므로 kube-proxy 애드온 생성 단계를 생략한다.

`lab-m`에서:

```bash
kubeadm init \
  --kubernetes-version v1.36.4 \
  --apiserver-advertise-address=192.168.184.235 \
  --control-plane-endpoint=192.168.184.235:6443 \
  --cri-socket unix:///run/containerd/containerd.sock \
  --skip-phases=addon/kube-proxy
```

> 이 단계에서는 `--pod-network-cidr`을 kubeadm에 지정하지 않는다. Cilium cluster-pool IPAM에서 `10.10.0.0/16`을 명시적으로 할당한다.

## 4. kubectl 설정

`kubeadm init` 성공 후 root 계정에서:

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

확인:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

CNI가 아직 없으므로 이 시점의 Control Plane은 `NotReady` 상태가 정상이다.

또한 kube-proxy가 생성되지 않았는지 확인한다.

```bash
kubectl -n kube-system get ds kube-proxy
```

예상 결과:

```text
Error from server (NotFound)
```

## 5. Cilium 설치 계획

Cilium 1.20.1 설치 시 다음 기능을 처음부터 활성화한다.

```text
kubeProxyReplacement = true
BPF masquerading      = true
Egress Gateway        = true
Cluster Pool IPAM     = 10.10.0.0/16
API Server            = 192.168.184.235:6443
```

예정 Helm 핵심 값:

```text
kubeProxyReplacement=true
bpf.masquerade=true
egressGateway.enabled=true
k8sServiceHost=192.168.184.235
k8sServicePort=6443
ipam.mode=cluster-pool
ipam.operator.clusterPoolIPv4PodCIDRList={10.10.0.0/16}
ipam.operator.clusterPoolIPv4MaskSize=24
```

Cilium 설치와 Helm 설치 명령은 `kubeadm init` 성공 후 실제 클러스터 상태를 확인한 다음 수행한다.

## 6. Worker / Egress Join

`kubeadm init` 출력의 join 명령을 보관한다.

이후:

```text
lab-w1 -> 일반 Worker로 Join
lab-e  -> Egress Gateway Node로 Join
```

Join 완료 후 `lab-e`에 label과 taint를 적용한다.

```bash
kubectl label node lab-e node-role=egress
kubectl taint node lab-e dedicated=egress:NoSchedule
```

## 완료 기준

- [ ] `cri-tools` 설치 및 `crictl info` 성공
- [ ] kubeadm image pull 성공
- [ ] `kubeadm init` 성공
- [ ] admin kubeconfig 설정
- [ ] kube-proxy 미생성 확인
- [ ] Cilium 설치
- [ ] Control Plane Ready
- [ ] `lab-w1` Join
- [ ] `lab-e` Join
- [ ] Egress Node label / taint 적용
