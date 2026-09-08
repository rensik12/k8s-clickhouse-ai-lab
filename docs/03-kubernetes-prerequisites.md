# 03. Kubernetes Prerequisites

## 목표

Rocky Linux 9.8 기반 세 노드를 Kubernetes v1.36.4 설치가 가능한 상태로 준비한다.

대상 노드:

```text
lab-m  : Control Plane
lab-w1 : Worker
lab-e  : Egress Gateway
```

선정 버전:

```text
Kubernetes : v1.36.4
Cilium     : v1.20.1
Runtime    : containerd
```

Cilium Egress Gateway를 이후 활성화하기 위해 Cilium은 `kubeProxyReplacement=true`와 BPF masquerading을 사용하는 방향으로 구축한다.

---

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
gentent hosts lab-w1
gentent hosts lab-e
```

> 실행 시 `gentent`가 아니라 `getent`를 사용한다. 아래 정상 확인 명령을 사용한다.

```bash
getent hosts lab-m
getent hosts lab-w1
getent hosts lab-e
```

노드 간 통신도 확인한다.

```bash
ping -c 2 lab-m
ping -c 2 lab-w1
ping -c 2 lab-e
```

---

## 2. Kernel Module

세 노드 모두 적용한다.

```bash
cat > /etc/modules-load.d/k8s.conf <<'EOF'
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
```

확인:

```bash
lsmod | grep -E 'overlay|br_netfilter'
```

---

## 3. sysctl

세 노드 모두 적용한다.

```bash
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

확인:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

기대값:

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

---

## 4. Swap 재확인

현재 세 노드는 Swap이 이미 비활성화되어 있다.

재확인:

```bash
swapon --show
free -h
grep -nE '\bswap\b' /etc/fstab || true
```

`swapon --show`에 출력이 없어야 한다.

---

## 5. cgroup 확인

Rocky Linux 9 계열의 cgroup 구성을 확인한다.

```bash
stat -fc %T /sys/fs/cgroup
```

cgroup v2 환경이면 다음과 같이 출력된다.

```text
cgroup2fs
```

containerd와 kubelet은 `systemd` cgroup driver로 통일한다.

---

## 6. containerd 설치

Docker Engine 자체는 설치하지 않고 Kubernetes CRI Runtime으로 사용할 `containerd.io`만 설치한다.

```bash
dnf install -y dnf-plugins-core

dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

dnf install -y containerd.io
```

버전 확인:

```bash
containerd --version
runc --version
```

기본 설정 파일 생성:

```bash
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```

Systemd Cgroup 활성화:

```bash
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

확인:

```bash
grep -n 'SystemdCgroup' /etc/containerd/config.toml
grep -n 'disabled_plugins' /etc/containerd/config.toml || true
```

`disabled_plugins`에 `cri`가 포함되어 있으면 제거해야 한다.

containerd 시작:

```bash
systemctl enable --now containerd
systemctl status containerd --no-pager
```

---

## 7. Kubernetes v1.36 Repository

세 노드 모두 적용한다.

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
```

kubelet 활성화:

```bash
systemctl enable --now kubelet
```

> 아직 `kubeadm init` 또는 `kubeadm join`을 실행하지 않았기 때문에 kubelet이 정상적인 Pod Runtime 상태가 아니거나 반복 재시작하는 것은 이 단계에서는 이상 현상이 아니다.

---

## 8. crictl Runtime Endpoint

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
crictl info
```

---

## 9. 설치 결과 확인

세 노드에서 다음 결과를 수집한다.

```bash
echo '===== VERSION ====='
kubeadm version -o short
kubelet --version
kubectl version --client
containerd --version

echo '===== CGROUP ====='
stat -fc %T /sys/fs/cgroup
grep -n 'SystemdCgroup' /etc/containerd/config.toml

echo '===== MODULE ====='
lsmod | grep -E 'overlay|br_netfilter'

echo '===== SYSCTL ====='
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward

echo '===== RUNTIME ====='
systemctl is-active containerd
crictl info | grep -E 'RuntimeReady|NetworkReady' -A2 || true
```

---

## 다음 단계

사전 설정 완료 후 Control Plane인 `lab-m`에서 `kubeadm init`을 수행한다.

Cilium의 kube-proxy replacement를 사용하기 위해 일반적인 kubeadm 구성과 달리 kube-proxy를 설치하지 않는 방식으로 초기화한다.

예정 흐름:

```text
lab-m
  kubeadm init
  --skip-phases=addon/kube-proxy
       |
       +--> kubectl 설정
       +--> Cilium 설치
       +--> lab-w1 join
       +--> lab-e join
       +--> lab-e label / taint
       +--> Cilium Egress Gateway 구성
```

실제 `kubeadm init` 명령과 PodCIDR은 사전 설정 검증 후 확정한다.
