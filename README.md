# k8s-clickhouse-ai-lab

실서비스와 유사한 Kubernetes 인프라를 직접 구축하고, Egress 네트워크와 GitLab CI + Helm + ArgoCD 기반 GitOps 배포 환경을 재현하는 개인 프로젝트입니다.

ClickHouse 기반 로그/AI 분석은 후속 단계로 진행하며, 현재 우선순위는 Kubernetes 및 CI/CD 면접 대비에 둡니다.

## 현재 목표

1. Kubernetes Control Plane / Worker / Egress 역할 분리 구조를 직접 구축한다.
2. Cilium 기반 kube-proxy replacement 및 Egress Gateway를 구현하고 패킷 경로를 검증한다.
3. Sample Application을 Helm Chart로 표준화한다.
4. GitLab CI에서 소스 빌드 → Docker 이미지 빌드/Push까지 구성한다.
5. ArgoCD에서 Git 저장소의 배포 정의를 감시하고 Pull 방식으로 Kubernetes에 배포한다.
6. Sync / Drift / Rollback / 배포 실패 시나리오를 직접 재현한다.
7. 이후 ClickHouse 기반 로그 파이프라인과 AI 분석 기능으로 확장한다.

## 1차 인프라 구성

| Node | Role | OS | vCPU / Memory | Disk | Network |
|---|---|---|---|---:|---|
| `lab-m` | Control Plane | Rocky Linux 9.8 | 4 / 7.5 GiB | 300 GB | `192.168.184.235` |
| `lab-w1` | Worker | Rocky Linux 9.8 | 8 / 15 GiB | 300 GB | `192.168.184.163` |
| `lab-e` | Egress Gateway | Rocky Linux 9.8 | 4 / 7.5 GiB | 300 GB | Private `192.168.184.179` / Public `211.47.73.206` |

초기 구성은 VM 3대로 시작하며, 향후 Worker 노드를 추가해 Pod 재스케줄링과 Worker 장애 테스트까지 확장합니다.

## 목표 아키텍처

```text
                         Internet
                            ^
                            |
                    211.47.73.206
                OpenStack Floating IP
                            ^
                            |
                         lab-e
                  Cilium Egress GW
                    192.168.184.179
                            ^
                            |
               EgressGatewayPolicy
                            |
        +-------------------+-------------------+
        |                                       |
 lab-m                                     lab-w1
 Control Plane                             Workloads
192.168.184.235                         192.168.184.163
                                                |
                                         Sample App
                                                ^
                                                |
                                             ArgoCD
                                                ^
                                                |
                                        Deployment Repo
                                                ^
                                                |
                                    GitLab CI / Helm values
                                                ^
                                                |
                                   Build -> Image -> Registry
                                                ^
                                                |
                                           Source Repo
```

## CI/CD 목표 흐름

```text
Developer Commit
      |
      v
GitLab CI
  - Test / Build
  - Docker Image Build
  - Registry Push
      |
      v
Deployment Repository
  - Helm values image tag 변경
      |
      v
ArgoCD
  - Git 변경 감지
  - Desired / Live 상태 비교
  - Sync
      |
      v
Kubernetes
```

CI와 CD의 역할을 분리하여 GitLab CI가 Kubernetes API에 직접 배포하는 구조가 아니라, 배포 상태는 Git에 선언하고 ArgoCD가 Pull 방식으로 반영하도록 구성합니다.

## 기술 스택

### Phase 1 - Kubernetes Infrastructure

- OS: Rocky Linux 9.8
- Container Runtime: containerd v2.3.4
- Kubernetes: v1.36.4 / kubeadm
- CNI: Cilium v1.20.1
- Networking: eBPF, kube-proxy replacement, VXLAN, NetworkPolicy, Egress Gateway

### Phase 2 - CI/CD / GitOps

- GitLab CI
- Docker
- Container Registry
- Helm
- ArgoCD
- Sample Application: FastAPI 예정

### Phase 3 - Observability / Log AI

- Fluent Bit
- ClickHouse
- Grafana
- Kafka (선택)
- AI Log Analyzer

## 네트워크 계획

```text
Node Network : 192.168.184.0/24
Pod CIDR     : 10.200.0.0/16
Service CIDR : 10.96.0.0/12
API Server   : 192.168.184.235:6443
```

Cilium cluster-pool IPAM이 `10.200.0.0/16`에서 노드별 `/24` Pod CIDR을 할당합니다.

## 진행 순서

### Phase 1. Kubernetes / Network

- [x] VM 3대 구성
- [x] Rocky Linux 9.8 설치
- [x] containerd 2.3.4 설치
- [x] Kubernetes v1.36.4 설치
- [x] Control Plane `lab-m` 초기화
- [x] kube-proxy 미설치 확인
- [x] `lab-w1`, `lab-e` Join
- [x] Cilium 설치
- [x] 3개 Node Ready 확인
- [ ] Cilium Egress Gateway 기능 설정값 재검증
- [ ] `lab-e` 전용 Egress Node label / taint 적용
- [ ] EgressGatewayPolicy 적용
- [ ] Pod 외부 통신 Source IP `211.47.73.206` 검증
- [ ] Egress 장애 시나리오 및 복구 검증

### Phase 2. GitLab CI / Helm / ArgoCD

- [ ] Sample FastAPI Application 작성
- [ ] Dockerfile 작성
- [ ] Helm Chart 작성 및 수동 배포
- [ ] GitLab CI Runner / Pipeline 구성
- [ ] Image Registry Push
- [ ] Source Repo / Deployment Repo 역할 분리
- [ ] ArgoCD 설치
- [ ] ArgoCD Application 생성
- [ ] Git 변경 기반 자동 Sync 검증
- [ ] Live Manifest 임의 변경 후 Drift / Self-Heal 검증
- [ ] 이전 Git Commit / Image Tag 기반 Rollback 검증
- [ ] 실패 배포 및 복구 시나리오 문서화

### Phase 3. ClickHouse / AI

- [ ] Fluent Bit 로그 수집
- [ ] ClickHouse 로그 적재
- [ ] Grafana 조회
- [ ] Kafka 도입 검토
- [ ] AI Log Analyzer 구현

## 면접 대비 핵심 질문

프로젝트를 진행하면서 아래 질문에 명령어가 아니라 구조와 이유로 답할 수 있도록 정리합니다.

- Control Plane과 Worker Node를 왜 분리하는가?
- Egress Node를 별도로 두는 이유와 단점은 무엇인가?
- kube-proxy 없이 Cilium이 Service 트래픽을 어떻게 처리하는가?
- Pod CIDR / Service CIDR / Node Network의 차이는 무엇인가?
- GitLab CI에서 Kubernetes까지 직접 배포하지 않고 ArgoCD를 사용하는 이유는 무엇인가?
- CI와 CD를 왜 분리하는가?
- Helm과 ArgoCD의 역할은 어떻게 다른가?
- ArgoCD의 Desired State와 Live State는 무엇인가?
- Auto Sync / Self Heal / Prune의 차이는 무엇인가?
- 잘못된 배포가 발생했을 때 GitOps 환경에서는 어떻게 롤백하는가?
- Source Repository와 Deployment Repository를 분리하는 이유는 무엇인가?

## 진행 문서

- [01. Architecture](docs/01-architecture.md)
- [02. VM / OS Setup](docs/02-vm-os-setup.md)
- [03. Kubernetes Prerequisites](docs/03-kubernetes-prerequisites.md)
- [04. Control Plane Bootstrap](docs/04-control-plane-bootstrap.md)
- [05. Cilium Installation](docs/05-cilium-install.md)
- [06. Interview-oriented Roadmap](docs/06-interview-roadmap.md)

## 구성 파일

- [Cilium values.yaml](kubernetes/cilium/values.yaml)

## 프로젝트 원칙

1. 실제 수행한 작업만 기록한다.
2. 명령어만 나열하지 않고 선택 이유를 기록한다.
3. 정상 구축 과정뿐 아니라 실패와 트러블슈팅도 남긴다.
4. 장애 시나리오를 의도적으로 만들고 결과를 검증한다.
5. 면접에서 구조, 선택 이유, 장애 대응 과정을 자기 말로 설명할 수 있도록 정리한다.
6. ClickHouse/AI보다 Kubernetes와 GitOps의 기본기를 우선 완성한다.
