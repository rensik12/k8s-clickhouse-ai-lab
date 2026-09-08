# 02. VM / OS Setup

## 목적

Kubernetes 클러스터 구축 전 VM과 OS 레벨의 기본 조건을 확인하고 실제 Lab 환경 정보를 기록한다.

## 노드 구성

| Hostname | Role | vCPU | Memory | Disk | Private IP | Public IP |
|---|---|---:|---:|---:|---|---|
| `lab-m` | Control Plane | 4 | 7.5 GiB | 300 GB | `192.168.184.235` | - |
| `lab-w1` | Worker | 8 | 15 GiB | 300 GB | `192.168.184.163` | - |
| `lab-e` | Egress Gateway | 4 | 7.5 GiB | 300 GB | `192.168.184.179` | `211.47.73.206` |

## OS / Kernel

세 노드의 환경은 동일하다.

```text
OS           : Rocky Linux 9.8 (Blue Onyx)
Architecture : x86_64
Kernel       : 5.14.0-687.44.1.el9_8.x86_64
Hypervisor   : OpenStack Nova / KVM
```

> 최초 계획은 Rocky Linux 9.5였으나 실제 설치 후 확인 결과 Rocky Linux 9.8이 설치되어 있어 실제 환경 기준으로 문서를 정정했다.

## 네트워크

```text
Network CIDR : 192.168.184.0/24
Gateway      : 192.168.184.1

lab-m
  eth0       : 192.168.184.235/24

lab-w1
  eth0       : 192.168.184.163/24

lab-e
  eth0       : 192.168.184.179/24
  Public IP  : 211.47.73.206
```

세 노드 모두 동일한 L2/L3 Private Network에 연결되어 있고 기본 경로는 다음과 같다.

```text
default via 192.168.184.1 dev eth0
```

OpenStack metadata endpoint route도 존재한다.

```text
169.254.169.254 via 192.168.184.2 dev eth0
```

## Egress Public IP 구조

`lab-e`의 인터페이스에는 `192.168.184.179`만 설정되어 있다.

```text
eth0  192.168.184.179/24
```

하지만 외부에서 확인되는 Source IP는 다음과 같다.

```bash
curl -4 ifconfig.me
```

```text
211.47.73.206
```

따라서 `211.47.73.206`은 VM NIC에 직접 바인딩된 주소가 아니라 OpenStack 네트워크 계층의 Floating IP / NAT 형태로 매핑된 것으로 판단한다.

향후 Cilium Egress Gateway에서는 Pod 트래픽을 `lab-e`로 전달하고 `lab-e`의 Private IP인 `192.168.184.179`를 Egress Source로 사용한다. 이후 OpenStack 외부 NAT를 거치면 인터넷에서는 `211.47.73.206`으로 관측되는 구조를 목표로 한다.

검증 목표:

```text
Pod
 -> Cilium Egress Gateway
 -> lab-e (192.168.184.179)
 -> OpenStack NAT / Floating IP
 -> Internet (211.47.73.206)
```

## /etc/hosts

세 노드에 동일하게 구성한다.

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

## Swap

세 노드 모두 Swap이 비활성화되어 있다.

```text
Swap: 0B
```

`swapon --show` 출력도 없다.

따라서 kubelet 구성을 위해 추가적인 Swap 비활성화 작업은 필요하지 않다.

## 시간 동기화

세 노드 모두 다음 상태를 확인했다.

```text
Time zone                : Asia/Seoul (KST, +0900)
System clock synchronized: yes
NTP service              : active
RTC in local TZ          : no
```

Kubernetes 인증서 및 노드 간 시간 차이 관점에서 정상 상태다.

## Firewall / SELinux

세 노드 공통:

```text
firewalld : inactive
SELinux   : Disabled
```

Lab 환경에서는 현재 상태를 유지한다.

실서비스 환경에서는 방화벽과 SELinux를 무조건 비활성화하는 방식보다 필요한 Kubernetes/CNI 통신 정책을 명시적으로 적용하는 것이 바람직하다.

## Disk

세 노드의 Root Filesystem은 약 299 GB이며 현재 사용률은 약 2%다.

```text
/dev/vda4  299G  4.5G  295G  2% /
```

단, `/boot` 파티션은 약 936 MB 중 705 MB가 사용되어 초기 상태에서도 사용률이 76%다.

```text
/dev/vda3  936M  705M  232M  76% /boot
```

향후 Kernel Update가 반복될 경우 오래된 Kernel 패키지가 누적되는지 주기적으로 확인한다.

## Kubernetes / Cilium 버전 결정

구축 시점 기준 다음 조합을 사용한다.

```text
Kubernetes : v1.36.4
Cilium     : v1.20.1
Runtime    : containerd
```

선정 이유:

- Kubernetes v1.36.4는 2026-09-08 기준 v1.36 계열 최신 Patch Release다.
- Cilium v1.20.1은 Kubernetes 1.33 / 1.34 / 1.35 / 1.36을 공식 e2e 테스트 대상으로 명시한다.
- Cilium v1.20.1은 Linux Kernel 5.10 이상을 요구한다.
- 현재 Rocky Linux Kernel `5.14.0-687.44.1.el9_8.x86_64`는 해당 최소 요구사항을 충족한다.
- Kubernetes v1.37.0이 더 최신이지만 현재 Cilium v1.20.1의 공식 호환 테스트 목록에는 포함되지 않아 Lab의 안정성과 재현성을 위해 v1.36을 선택한다.

## 완료 기준

- [x] VM 3대 생성
- [x] Rocky Linux 설치
- [x] 실제 OS 버전 확인: Rocky Linux 9.8
- [x] 각 VM 300 GB 디스크 할당
- [x] Hostname 확정
- [x] CPU / Memory 실제 사양 기록
- [x] Kernel 정보 기록
- [x] Private IP 확정
- [x] Gateway 확인
- [x] Egress Public IP 확인
- [x] `lab-e` Public IP 구성 방식 확인
- [x] Internet 통신 확인
- [x] Swap 비활성화 확인
- [x] NTP 동기화 확인
- [x] Firewall / SELinux 상태 확인
- [ ] `/etc/hosts` 구성 및 이름 해석 확인
- [ ] 노드 간 ICMP 통신 확인
- [ ] DNS resolver 정보 기록

## 작업 기록

### 2026-09-08

```text
작업 내용:
- Kubernetes Lab용 OpenStack VM 3대 생성
- Control Plane / Worker / Egress 역할 분리
- 실제 OS / Kernel / CPU / Memory / Disk / Network 정보 수집
- Swap / NTP / Firewall / SELinux 상태 확인
- Egress Node 외부 Source IP 확인

결과:
- lab-m  : 4 vCPU / 7.5 GiB / 192.168.184.235
- lab-w1 : 8 vCPU / 15 GiB / 192.168.184.163
- lab-e  : 4 vCPU / 7.5 GiB / 192.168.184.179
- lab-e 외부 Source IP: 211.47.73.206
- OS: Rocky Linux 9.8
- Kernel: 5.14.0-687.44.1.el9_8.x86_64
- Swap 비활성화
- NTP 정상 동기화
- firewalld inactive
- SELinux Disabled

판단:
- lab-e Public IP는 VM NIC 직접 할당이 아니라 OpenStack Floating IP / NAT 구조로 판단
- Kubernetes / Cilium 구축을 진행하기 위한 기본 OS 조건 충족
- Kubernetes v1.36.4 + Cilium v1.20.1 조합으로 진행

다음 작업:
- 노드 간 이름 해석 / 통신 최종 확인
- Kernel module 및 sysctl 설정
- containerd 설치 및 SystemdCgroup 적용
- kubeadm / kubelet / kubectl 설치
```
