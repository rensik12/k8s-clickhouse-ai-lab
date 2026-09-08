# 02. VM / OS Setup

## 목적

Kubernetes 클러스터 구축 전 VM과 OS 레벨의 기본 조건을 표준화한다.

이 문서는 실제 VM 생성 및 OS 설치를 진행하면서 값과 명령어를 채워간다.

## 노드 구성

| Hostname | Role | OS | Disk | Private IP | Public IP |
|---|---|---|---:|---|---|
| `lab-m` | Control Plane | Rocky Linux 9.5 | 300 GB | `192.168.184.235` | - |
| `lab-w1` | Worker | Rocky Linux 9.5 | 300 GB | `192.168.184.163` | - |
| `lab-e` | Egress Gateway | Rocky Linux 9.5 | 300 GB | `192.168.184.179` | `211.47.73.206` |

## OS

- Distribution: Rocky Linux
- Version: 9.5
- Architecture: x86_64 예정 / 실제 명령으로 확인
- Kernel: 확인 예정

확인 명령:

```bash
cat /etc/os-release
uname -r
uname -m
```

## Hostname

설치 완료 기준 Hostname:

```text
lab-m  : Control Plane
lab-w1 : Worker
lab-e  : Egress Gateway
```

확인:

```bash
hostnamectl
```

필요 시 설정:

```bash
hostnamectl set-hostname lab-m
hostnamectl set-hostname lab-w1
hostnamectl set-hostname lab-e
```

각 명령은 해당 노드에서 역할에 맞는 Hostname 하나만 적용한다.

## 네트워크 정보

현재 확정된 주소:

```text
lab-m
  Private: 192.168.184.235

lab-w1
  Private: 192.168.184.163

lab-e
  Private: 192.168.184.179
  Public : 211.47.73.206
```

Network CIDR, Gateway, DNS 및 실제 인터페이스 구성은 다음 명령으로 추가 확인한다.

```bash
ip -br addr
ip route
cat /etc/resolv.conf
```

### Egress Node 확인 포인트

`lab-e`의 `211.47.73.206`이 VM 인터페이스에 직접 설정된 주소인지, 상위 네트워크에서 1:1 NAT 등으로 매핑된 주소인지 확인한다.

이 차이는 이후 Cilium Egress Gateway에서 사용할 Egress IP와 SNAT 동작을 설계할 때 중요하다.

확인 예시:

```bash
ip -br addr
ip route
curl -4 ifconfig.me
```

## /etc/hosts

Lab 환경에서 노드 간 이름 해석을 보장하기 위해 세 노드에 동일하게 구성한다.

```text
192.168.184.235  lab-m
192.168.184.163  lab-w1
192.168.184.179  lab-e
```

확인:

```bash
getent hosts lab-m
getent hosts lab-w1
getent hosts lab-e
```

## 노드 간 통신 확인

각 노드에서 나머지 노드의 Private IP와 Hostname으로 통신 가능한지 확인한다.

예:

```bash
ping -c 3 lab-m
ping -c 3 lab-w1
ping -c 3 lab-e
```

## 시간 동기화

Kubernetes 및 인증서 관련 문제를 피하기 위해 각 노드의 시간 동기화 상태를 확인한다.

```bash
timedatectl
chronyc tracking
chronyc sources -v
```

## Swap

kubelet 구성 전 swap 사용 여부를 확인한다.

```bash
swapon --show
free -h
```

실제 비활성화 작업은 Kubernetes 사전 설정 단계에서 수행한다.

## Firewall / SELinux

Rocky Linux 9.5 기본 보안 설정 상태를 먼저 확인한다.

```bash
systemctl status firewalld --no-pager
getenforce
sestatus
```

초기부터 무조건 비활성화하지 않고 Kubernetes와 Cilium 구성에 필요한 정책을 검토한 뒤 Lab 구성 방식을 결정한다.

## 설치 완료 후 기본 정보 수집

세 노드에서 아래 명령 결과를 수집한다.

```bash
hostnamectl
cat /etc/os-release
uname -r
uname -m
ip -br addr
ip route
free -h
df -h
swapon --show
timedatectl
systemctl is-active firewalld
getenforce
```

## 완료 기준

- [x] VM 3대 생성
- [x] Rocky Linux 9.5 설치
- [x] 디스크 300 GB 할당
- [x] Hostname 확정
- [x] Private IP 확정
- [x] Egress Public IP 확보
- [ ] CPU / Memory 실제 사양 기록
- [ ] Kernel 정보 기록
- [ ] Gateway / DNS 확인
- [ ] `/etc/hosts` 구성
- [ ] 노드 간 이름 해석 확인
- [ ] 노드 간 ICMP 통신 확인
- [ ] Internet 통신 확인
- [ ] `lab-e` Public IP 구성 방식 확인
- [ ] 시간 동기화 확인
- [ ] Swap 상태 확인
- [ ] Firewall / SELinux 상태 확인

## 작업 기록

### 2026-09-08

```text
작업 내용:
- Kubernetes Lab용 VM 3대 생성
- Rocky Linux 9.5 설치
- Control Plane / Worker / Egress 역할 분리
- 각 VM 300 GB 디스크 구성
- Private IP 및 Egress Public IP 확정

결과:
- lab-m  : 192.168.184.235
- lab-w1 : 192.168.184.163
- lab-e  : 192.168.184.179 / 211.47.73.206

다음 작업:
- OS/Kernel/CPU/Memory/Network 기본 정보 수집
- 노드 간 통신 및 이름 해석 확인
- Kubernetes 사전 설정 진행
```
