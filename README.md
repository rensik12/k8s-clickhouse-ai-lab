# k8s-clickhouse-ai-lab

Production-like Kubernetes 환경을 직접 구축하고, ClickHouse 기반 로그 파이프라인과 AI 장애 분석 기능까지 확장하는 개인 프로젝트입니다.

## 목표

- Kubernetes 클러스터를 직접 구축하며 Control Plane / Worker / Egress 역할 분리 구조 이해
- Cilium 기반 네트워크 및 Egress Gateway 구조 구현
- Kubernetes 애플리케이션 로그를 ClickHouse에 적재
- Grafana를 통한 로그 조회 및 시각화
- AI를 활용한 오류 로그 요약 및 장애 원인 분석 기능 구현
- 구축 과정, 장애 테스트, 트러블슈팅을 재현 가능한 형태로 문서화

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
                              +-----------------+-----------------+
                              |                 |                 |
                          Sample App       Fluent Bit         Grafana
                                                |
                                            ClickHouse
                                                |
                                         AI Log Analyzer
```

## 기술 스택

- OS: Rocky Linux 9.8
- Container Runtime: containerd v2.3.4
- Kubernetes: v1.36.4 / kubeadm
- CNI: Cilium v1.20.1
- Networking: eBPF, kube-proxy replacement, VXLAN, NetworkPolicy, Egress Gateway
- Logging: Fluent Bit, ClickHouse
- Visualization: Grafana
- Application: Python FastAPI
- AI Analyzer: Python API + LLM API
- 2차 확장: Kafka, Argo CD, Prometheus

## 네트워크 계획

```text
Node Network : 192.168.184.0/24
Pod CIDR     : 10.200.0.0/16
Service CIDR : 10.96.0.0/12
API Server   : 192.168.184.235:6443
```

Cilium cluster-pool IPAM이 `10.200.0.0/16`에서 노드별 `/24` Pod CIDR을 할당한다.

## 현재 진행 상태

- [x] VM 3대 구성
- [x] Rocky Linux 9.8 설치
- [x] containerd 2.3.4 설치
- [x] Kubernetes v1.36.4 설치
- [x] Control Plane `lab-m` 초기화
- [x] kube-proxy 미설치 확인
- [ ] `lab-w1`, `lab-e` Join
- [ ] Cilium 1.20.1 설치
- [ ] Egress Gateway 검증
- [ ] ClickHouse 로그 파이프라인 구축
- [ ] AI Log Analyzer 구축

## 진행 문서

- [01. Architecture](docs/01-architecture.md)
- [02. VM / OS Setup](docs/02-vm-os-setup.md)
- [03. Kubernetes Prerequisites](docs/03-kubernetes-prerequisites.md)
- [04. Control Plane Bootstrap](docs/04-control-plane-bootstrap.md)
- [05. Cilium Installation](docs/05-cilium-install.md)

## 구성 파일

- [Cilium values.yaml](kubernetes/cilium/values.yaml)

## 프로젝트 원칙

1. 실제 수행한 작업만 기록한다.
2. 명령어만 나열하지 않고 선택 이유를 기록한다.
3. 정상 구축 과정뿐 아니라 실패와 트러블슈팅도 남긴다.
4. 장애 시나리오를 의도적으로 만들고 결과를 검증한다.
5. 최종적으로 다른 환경에서도 재현 가능한 문서를 목표로 한다.
