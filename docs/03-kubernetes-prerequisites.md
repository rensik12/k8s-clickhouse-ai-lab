# 03. Kubernetes Prerequisites

## 목표

Rocky Linux 9.8 기반 세 노드를 Kubernetes v1.36.4 설치가 가능한 상태로 준비한다.

```text
lab-m  : Control Plane
lab-w1 : Worker
lab-e  : Egress Gateway
```

선정 버전:

```text
Kubernetes : v1.36.4
Cilium     : v1.20.1
Runtime    : containerd v2.3.4
```

Cilium Egress Gateway를 이후 활성화하기 위해 Cilium은 `kubeProxyReplacement=true`와 BPF masquerading을 사용하는 방향으로 구축한다.

## 1. 노드 이름 해석

세 노드 모두 동일하게 설정한다.

```bash
cat >> /etc/hosts <<'EOF'
192.168.184.235  lab-m
192.168.184.163  lab-w1
192.168.184.179  lab-e
EOF
```

확인:

```bash
getent hosts lab-m
getent hosts lab-w1
getent hosts lab-e
```

## 2. Kernel Module

```bash
cat > /etc/modules-load.d/k8s.conf <<'EOF'
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
```

실제 확인 결과:

```text
br_netfilter  loaded
overlay       loaded
```

## 3. sysctl

```bash
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

실제 결과:

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

## 4. Swap

세 노드 모두 Swap 비활성화 상태다.

```text
Swap: 0B
swapon --show: no output
```

## 5. cgroup

세 노드 모두 cgroup v2를 사용한다.

```text
cgroup2fs
```

containerd cgroup driver:

```text
SystemdCgroup = true
```

## 6. containerd

세 노드 모두 설치 완료.

```text
containerd v2.3.4
service: active
```

설정:

```bash
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl enable --now containerd
```

## 7. Kubernetes v1.36 Repository

```bash
cat > /etc/yum.repos.d/kubernetes.repo <<'EOF'
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

설치:

```bash
dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable --now kubelet
```

실제 버전:

```text
kubeadm  v1.36.4
kubelet  v1.36.4
kubectl  v1.36.4
```

## 8. crictl

초기 검증 과정에서 세 노드 모두 다음 오류를 확인했다.

```text
-bash: crictl: 명령어를 찾을 수 없음
```

`crictl`은 `cri-tools` 패키지가 제공하므로 별도 설치한다.

```bash
dnf install -y cri-tools --disableexcludes=kubernetes
```

Runtime endpoint:

```bash
cat > /etc/crictl.yaml <<'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

확인:

```bash
crictl --version
crictl info
```

`crictl`은 kubeadm 자체의 필수 요소는 아니지만 CRI 및 런타임 문제를 직접 진단하기 위해 설치한다.

## 9. 최종 사전 점검 결과

| Item | Result |
|---|---|
| Kubernetes | v1.36.4 |
| containerd | v2.3.4 |
| cgroup | v2 (`cgroup2fs`) |
| containerd cgroup driver | systemd |
| overlay | loaded |
| br_netfilter | loaded |
| IPv4 forwarding | enabled |
| bridge iptables | enabled |
| Swap | disabled |
| containerd | active |
| crictl | 별도 설치 필요 확인 |

## 다음 단계

Control Plane인 `lab-m`에서 kube-proxy를 생성하지 않는 방식으로 클러스터를 초기화한다.

```text
lab-m
  kubeadm init --skip-phases=addon/kube-proxy
       |
       +--> kubectl 설정
       +--> Cilium 1.20.1 설치
       +--> lab-w1 join
       +--> lab-e join
       +--> lab-e label / taint
       +--> Cilium Egress Gateway 구성
```

Control Plane API endpoint는 `192.168.184.235:6443`을 사용한다.
