# 04. Control Plane Bootstrap

## 목표

`lab-m`을 Kubernetes v1.36.4 Control Plane으로 초기화하고, kube-proxy 대신 Cilium 1.20.1의 eBPF kube-proxy replacement를 사용한다.

## 네트워크 설계

```text
Node Network  : 192.168.184.0/24
Pod CIDR      : 10.200.0.0/16
Service CIDR  : 10.96.0.0/12
API Server    : 192.168.184.235:6443
```

Pod CIDR은 kubeadm에서 할당하지 않고 Cilium cluster-pool IPAM에서 관리한다.

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
```

## 2. Control Plane 초기화

Cilium의 kube-proxy replacement를 사용하므로 kube-proxy 애드온 생성 단계를 생략했다.

실행 명령:

```bash
kubeadm init \
  --kubernetes-version v1.36.4 \
  --apiserver-advertise-address=192.168.184.235 \
  --control-plane-endpoint=192.168.184.235:6443 \
  --cri-socket unix:///run/containerd/containerd.sock \
  --skip-phases=addon/kube-proxy
```

> `--pod-network-cidr`은 지정하지 않았다. Pod 주소는 Cilium cluster-pool IPAM에서 `10.200.0.0/16`을 사용한다.

## 3. kubectl 설정

```bash
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

## 4. 초기화 결과

2026-09-08 기준 Control Plane 초기화 성공.

```text
Node             : lab-m
Kubernetes       : v1.36.4
Internal IP      : 192.168.184.235
Container Runtime: containerd 2.3.4
Status           : NotReady
```

`NotReady`는 CNI가 아직 설치되지 않았기 때문에 정상이다.

Control Plane 구성요소 상태:

```text
etcd-lab-m                      Running
kube-apiserver-lab-m            Running
kube-controller-manager-lab-m   Running
kube-scheduler-lab-m            Running
CoreDNS                         Pending
```

CoreDNS 역시 CNI와 일반 Worker가 아직 준비되지 않았으므로 Pending 상태이다.

## 5. kube-proxy 미설치 확인

```bash
kubectl -n kube-system get ds kube-proxy
```

확인 결과:

```text
Error from server (NotFound): daemonsets.apps "kube-proxy" not found
```

의도대로 kube-proxy가 생성되지 않았다.

## 6. Node Join

`kubeadm init`에서 생성된 Worker Join 명령을 `lab-w1`, `lab-e`에서 실행한다.

보안상 실제 bootstrap token과 discovery hash는 공개 저장소에 기록하지 않는다.

```bash
kubeadm join 192.168.184.235:6443 \
  --token <REDACTED> \
  --discovery-token-ca-cert-hash sha256:<REDACTED>
```

대상:

```text
lab-w1 -> Worker
lab-e  -> Egress Gateway Node
```

Join 직후 CNI가 설치되기 전까지 두 노드가 `NotReady`인 것은 정상이다.

## 7. Cilium 설치 계획

Cilium 1.20.1을 다음 값으로 설치한다.

```text
kubeProxyReplacement = true
BPF masquerading      = true
Egress Gateway        = true
Routing Mode          = tunnel
Tunnel Protocol       = VXLAN
Cluster Pool IPAM     = 10.200.0.0/16
Per-node Pod CIDR     = /24
API Server            = 192.168.184.235:6443
```

Cilium 구성은 `kubernetes/cilium/values.yaml`에 버전 관리한다.

## 8. Egress Node 역할 제한

Cilium 설치 및 전체 노드 Ready 확인 후 `lab-e`에 역할 label과 taint를 적용한다.

```bash
kubectl label node lab-e node-role=egress
kubectl taint node lab-e dedicated=egress:NoSchedule
```

일반 애플리케이션 Pod는 `lab-e`에 스케줄링하지 않고 Cilium Agent 등 DaemonSet 계열만 실행하도록 한다.

## 완료 기준

- [x] kubeadm image 준비
- [x] `kubeadm init` 성공
- [x] admin kubeconfig 설정
- [x] kube-proxy 미생성 확인
- [ ] `lab-w1` Join
- [ ] `lab-e` Join
- [ ] Cilium 설치
- [ ] 전체 Node Ready
- [ ] CoreDNS Running
- [ ] Egress Node label / taint 적용
