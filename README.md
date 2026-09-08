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

| Node | Role | 주요 용도 |
|---|---|---|
| `k8s-master01` | Control Plane | kube-apiserver, scheduler, controller-manager, etcd |
| `k8s-worker01` | Worker | Application, ClickHouse, Grafana, Logging workloads |
| `k8s-egress01` | Egress Gateway | 외부 통신 경로 분리 및 SNAT |

초기 구성은 VM 3대로 시작하며, 향후 `k8s-worker02`를 추가해 Pod 재스케줄링과 Worker 장애 테스트까지 확장합니다.

## 목표 아키텍처

```text
                        Internet
                           ^
                           |
                    k8s-egress01
                  Cilium Egress GW
                           ^
                           |
              EgressGatewayPolicy
                           |
        +------------------+------------------+
        |                                     |
 k8s-master01                         k8s-worker01
 Control Plane                         Workloads
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

- OS: Linux
- Container Runtime: containerd
- Kubernetes: kubeadm
- CNI: Cilium
- Networking: eBPF, NetworkPolicy, Egress Gateway
- Logging: Fluent Bit, ClickHouse
- Visualization: Grafana
- Application: Python FastAPI
- AI Analyzer: Python API + LLM API
- 2차 확장: Kafka, Argo CD, Prometheus

## 진행 문서

- [01. Architecture](docs/01-architecture.md)
- [02. VM / OS Setup](docs/02-vm-os-setup.md)

추가 문서는 실제 구축 진행에 맞춰 순차적으로 작성합니다.

## 프로젝트 원칙

1. 실제 수행한 작업만 기록한다.
2. 명령어만 나열하지 않고 선택 이유를 기록한다.
3. 정상 구축 과정뿐 아니라 실패와 트러블슈팅도 남긴다.
4. 장애 시나리오를 의도적으로 만들고 결과를 검증한다.
5. 최종적으로 다른 환경에서도 재현 가능한 문서를 목표로 한다.
