# 01. Architecture

## 프로젝트 개요

이 프로젝트는 Kubernetes 클러스터를 단순 설치하는 데서 끝내지 않고, 역할이 분리된 운영형 구조와 로그 분석 플랫폼을 직접 구현하는 것을 목표로 한다.

핵심 흐름은 다음과 같다.

```text
Application
   |
   +--> Internet traffic --> Cilium Egress Gateway --> Internet
   |
   +--> Container log --> Fluent Bit --> ClickHouse --> Grafana
                                                |
                                                +--> AI Log Analyzer
```

## 초기 노드 구성

### lab-m

- Role: Kubernetes Control Plane
- OS: Rocky Linux 9.5
- Disk: 300 GB
- Private IP: `192.168.184.235`
- 일반 Application Pod는 스케줄링하지 않음
- 주요 구성요소
  - kube-apiserver
  - kube-controller-manager
  - kube-scheduler
  - etcd

### lab-w1

- Role: Worker
- OS: Rocky Linux 9.5
- Disk: 300 GB
- Private IP: `192.168.184.163`
- 일반 Workload 실행 노드
- 초기 단계에서는 다음 서비스를 함께 배치
  - Sample Application
  - ClickHouse
  - Fluent Bit
  - Grafana
  - AI Log Analyzer

> 단일 Worker 구조이므로 초기 목표는 HA가 아니라 역할 분리, 네트워크 경로, 로그 파이프라인을 이해하고 검증하는 데 둔다.

### lab-e

- Role: Egress Gateway
- OS: Rocky Linux 9.5
- Disk: 300 GB
- Private IP: `192.168.184.179`
- Public IP: `211.47.73.206`
- Kubernetes Node로 클러스터에 Join
- 일반 Workload 스케줄링을 제한
- Cilium Egress Gateway 역할 수행
- 특정 Pod/Namespace의 외부 통신을 해당 노드로 경유시키고 공인 IP 기반 Egress 경로를 검증

예정 설정 개념:

```text
label: node-role=egress
taint: dedicated=egress:NoSchedule
```

## 네트워크 구조

```text
                  Internet
                     ^
                     |
               211.47.73.206
                   lab-e
              Egress Gateway
              192.168.184.179
                     ^
                     |
          Cilium EgressGatewayPolicy
                     |
         +-----------+-----------+
         |                       |
      lab-m                    lab-w1
192.168.184.235            192.168.184.163
 Control Plane                Workloads
```

`lab-e`는 내부 통신용 Private IP와 외부 통신에 사용할 Public IP를 모두 가진다. 이후 Egress Gateway 적용 전/후에 Pod에서 외부로 보이는 Source IP를 비교하여 실제 경로 변화를 검증한다.

## Network 설계 목표

1. 일반 Pod가 Worker Node의 기본 경로를 통해 외부 통신하는 상태 확인
2. Cilium Egress Gateway Policy 적용
3. 선택된 Pod의 외부 트래픽이 `lab-e`를 경유하는지 검증
4. 외부에서 보이는 Source IP가 `211.47.73.206`으로 고정되는지 확인
5. Egress Node 장애 시 영향 확인
6. NetworkPolicy와 Egress Policy의 역할 차이 정리

## Logging 설계 목표

### Phase 1

```text
Sample App
   |
stdout / file
   |
Fluent Bit
   |
ClickHouse
   |
Grafana
```

목표:

- Kubernetes 로그 수집
- 구조화된 로그 적재
- 서비스/Pod/Namespace 기준 조회
- Error 로그 검색 및 집계

### Phase 2

```text
Sample App
   |
Fluent Bit
   |
Kafka
   |
ClickHouse
```

Kafka를 중간 버퍼로 추가하여 로그 생산 계층과 저장 계층의 결합도를 낮추는 구조를 검토한다.

## AI Log Analyzer 목표

AI 기능은 단순 챗봇이 아니라 운영자가 실제 장애 상황에서 사용할 수 있는 기능을 목표로 한다.

예시:

```text
사용자 질문
"최근 10분간 payment-api 오류 원인을 분석해줘"

        |
        v
AI Analyzer API
        |
        +--> ClickHouse Query
        |
        +--> Error pattern grouping
        |
        +--> LLM summary
        v
장애 요약 / 주요 오류 / 발생 시각 / 관련 Pod / 확인 항목
```

## 향후 확장

VM 자원이 허용되면 다음 순서로 확장한다.

1. Worker Node 추가
2. Pod Anti-Affinity / NodeSelector / Taint / Toleration 테스트
3. Worker 장애 및 Pod 재스케줄링 테스트
4. Prometheus 추가
5. Kafka 추가
6. Argo CD 기반 GitOps 배포
7. ClickHouse와 기존 ELK 구조의 적재량/조회성능/자원사용량 비교

## 프로젝트에서 검증할 핵심 질문

- Kubernetes Node 역할을 왜 분리하는가?
- Egress Node를 별도로 두면 어떤 운영상 장단점이 있는가?
- Cilium Egress Gateway가 실제 패킷 경로를 어떻게 변경하는가?
- ClickHouse는 로그 저장소로 어떤 장점과 한계를 가지는가?
- 로그 분석에 AI를 붙일 때 단순 요약을 넘어 어떤 운영 가치를 만들 수 있는가?
