# 02. VM / OS Setup

## 목적

Kubernetes 클러스터 구축 전 VM과 OS 레벨의 기본 조건을 표준화한다.

이 문서는 실제 VM 생성 및 OS 설치를 진행하면서 값과 명령어를 채워간다.

## 노드 계획

| Hostname | Role | vCPU | Memory | Disk | IP |
|---|---|---:|---:|---:|---|
| `k8s-master01` | Control Plane | 4 | 8 GB | 50 GB | TBD |
| `k8s-worker01` | Worker | 8 | 16 GB+ | 150 GB+ | TBD |
| `k8s-egress01` | Egress Gateway | 4 | 4 GB | 30 GB | TBD |

> 실제 할당 자원은 사용 가능한 하이퍼바이저 자원에 맞춰 조정한다.

## OS

- Distribution: TBD
- Version: TBD
- Kernel: TBD
- Architecture: x86_64

확인 명령:

```bash
cat /etc/os-release
uname -r
uname -m
```

## Hostname 설정

각 노드에서 역할에 맞게 설정한다.

```bash
hostnamectl set-hostname k8s-master01
hostnamectl set-hostname k8s-worker01
hostnamectl set-hostname k8s-egress01
```

설정 확인:

```bash
hostnamectl
```

## 네트워크 정보 기록

각 노드에서 아래 정보를 확인하고 결과를 기록한다.

```bash
ip -br addr
ip route
cat /etc/resolv.conf
```

### 확정 후 기록할 항목

```text
Network CIDR : TBD
Gateway      : TBD
DNS          : TBD

k8s-master01 : TBD
k8s-worker01 : TBD
k8s-egress01 : TBD
```

## /etc/hosts

DNS가 별도로 없는 Lab 환경에서는 노드 간 이름 확인을 위해 `/etc/hosts`를 동일하게 구성한다.

```text
<MASTER_IP>  k8s-master01
<WORKER_IP>  k8s-worker01
<EGRESS_IP>  k8s-egress01
```

확인:

```bash
getent hosts k8s-master01
getent hosts k8s-worker01
getent hosts k8s-egress01
```

## 시간 동기화

Kubernetes 및 인증서 관련 문제를 피하기 위해 각 노드의 시간 동기화 상태를 확인한다.

```bash
timedatectl
```

NTP 동기화 여부와 시간대 설정은 실제 OS 설치 후 기록한다.

## Swap

kubelet 구성 전 swap 사용 여부를 확인하고 프로젝트 구성에 맞게 처리한다.

```bash
swapon --show
free -h
```

실제 변경 작업은 Kubernetes 설치 단계에서 기록한다.

## Firewall / Security

초기에는 방화벽을 무조건 비활성화하지 않고 다음 원칙으로 접근한다.

1. 현재 상태 확인
2. Kubernetes 및 Cilium에 필요한 포트 정리
3. Lab 환경에서 단순화를 위해 비활성화할 경우 그 이유를 기록

확인 예시:

```bash
systemctl status firewalld
# 또는
ufw status
```

## 완료 기준

- [ ] VM 3대 생성
- [ ] OS 설치 완료
- [ ] Hostname 설정
- [ ] 고정 IP 설정
- [ ] Gateway / DNS 확인
- [ ] 노드 간 이름 해석 확인
- [ ] 노드 간 ICMP 통신 확인
- [ ] Internet 통신 확인
- [ ] 시간 동기화 확인
- [ ] OS / Kernel 정보 기록

## 작업 기록

### YYYY-MM-DD

```text
작업 내용:

결과:

이슈:

해결:
```
