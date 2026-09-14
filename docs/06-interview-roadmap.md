# 06. Interview-oriented Roadmap

## 목적

이 프로젝트의 우선순위를 면접 대비에 맞춘다.

핵심은 기술을 많이 붙이는 것이 아니라, 실제 업무에서 다뤄온 Kubernetes / CI/CD / GitOps 구조를 직접 재구축하고 다음 항목을 설명할 수 있게 만드는 것이다.

- 왜 이 구조를 선택했는가?
- 각 구성요소의 역할은 무엇인가?
- 장애가 나면 어디부터 확인하는가?
- 기존 방식과 비교해 어떤 장단점이 있는가?
- 운영 환경에서는 어떤 보완이 필요한가?

---

## Phase 1. Kubernetes Infrastructure

### 1-1. Control Plane / Worker / Egress 역할 분리

```text
lab-m  : Control Plane
lab-w1 : General Workload
lab-e  : Egress Gateway
```

확인할 내용:

- kubeadm 기반 클러스터 초기화 과정
- Static Pod 형태의 Control Plane 구성요소
- kubelet / containerd / CNI 관계
- Node NotReady가 발생하는 대표 원인
- Control Plane Taint 의미

### 1-2. Cilium / kube-proxy replacement

목표:

- kube-proxy 없이 Cilium eBPF로 Service 처리
- Cluster Pool IPAM 기반 Pod CIDR 할당
- VXLAN Overlay 통신 확인

직접 확인할 항목:

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get ciliumnodes
kubectl get svc -A
```

면접 포인트:

- kube-proxy의 역할
- iptables / IPVS 기반 Service 처리와 eBPF 방식 차이
- Overlay / Native Routing 차이
- Pod Network와 Service Network의 차이

### 1-3. Egress Gateway

목표 경로:

```text
Application Pod
      |
      v
lab-w1
      |
      v
Cilium Egress Gateway
      |
      v
lab-e (192.168.184.179)
      |
      v
OpenStack Floating IP
      |
      v
211.47.73.206
      |
      v
Internet
```

검증:

1. Egress 정책 적용 전 Source IP 확인
2. 정책 적용 후 Source IP 확인
3. `211.47.73.206`으로 고정되는지 확인
4. 특정 Namespace / Label Pod만 Egress 정책 적용
5. Egress Node 장애 시 영향 확인
6. 정책 제거 후 정상 경로 복구

면접 포인트:

- Egress Node 분리 이유
- 방화벽 ACL 관리 측면 장점
- 공인 IP 절감
- 단일 Egress Node의 SPOF 문제
- 다중 Egress Node 구성 시 고려사항

---

## Phase 2. Helm / GitLab CI / ArgoCD

### 2-1. Sample Application

단순한 FastAPI 애플리케이션을 만든다.

필수 Endpoint 예시:

```text
/health
/version
/error
```

`/version`은 배포된 이미지 버전을 확인하는 데 사용하고 `/error`는 향후 로그 테스트에도 활용한다.

### 2-2. Docker / Registry

```text
Source
  |
  v
GitLab CI
  |
  +--> Test
  +--> Docker Build
  +--> Tag
  +--> Registry Push
```

Image Tag는 `latest`만 사용하지 않고 Git Commit SHA 또는 명시적 버전을 사용한다.

예:

```text
sample-app:1.0.0
sample-app:a81e92c
```

면접 포인트:

- Immutable Image가 필요한 이유
- latest tag의 문제
- Registry 인증정보 관리 방법
- ImagePullSecret 역할

### 2-3. Helm Chart

최소 구성:

```text
helm/sample-app/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

검증:

```bash
helm lint
helm template
helm upgrade --install
```

면접 포인트:

- Helm을 사용하는 이유
- values와 template 역할
- 환경별 values 분리 방법
- Helm release와 Kubernetes resource 관계

### 2-4. CI와 CD 분리

목표 구조:

```text
[Application Repository]
        |
        | GitLab CI
        v
 Build / Test / Image Push
        |
        v
[Deployment Repository]
 Helm values image.tag 변경
        |
        v
      ArgoCD
        |
        v
    Kubernetes
```

GitLab CI에서 `kubectl apply` 또는 `helm upgrade`를 직접 실행하지 않는다.

CI 역할:

- 소스 검증
- Build
- Image 생성
- Registry Push
- 배포 Repository의 image tag 변경

CD 역할:

- ArgoCD가 Deployment Repository 감시
- Desired State 비교
- Kubernetes에 Sync

면접 포인트:

- Push CD와 Pull CD 차이
- ArgoCD 도입 이유
- CI Runner에 Kubernetes Credential을 직접 주지 않아도 되는 장점
- Git을 배포 이력 및 Desired State의 Source of Truth로 사용하는 이유

### 2-5. ArgoCD

검증할 기능:

- Application 생성
- Manual Sync
- Auto Sync
- Self Heal
- Prune
- Health / Sync Status
- History
- Rollback

의도적으로 만들 장애:

1. Deployment의 replica를 `kubectl edit`로 변경
2. ArgoCD가 OutOfSync를 감지하는지 확인
3. Self Heal 활성화 후 Git 기준 상태로 복구 확인
4. Git에서 Resource 삭제 후 Prune 동작 확인
5. 잘못된 Image Tag 배포
6. 이전 정상 버전으로 Rollback

면접 포인트:

- Synced와 Healthy 차이
- Drift가 무엇인지
- Self Heal이 항상 좋은가?
- Prune의 위험성
- ArgoCD 장애 시 기존 서비스에 미치는 영향
- ArgoCD가 Git 변경을 감지하는 방식

---

## Phase 3. Interview Failure Scenarios

정상 구축 후 일부러 장애를 만든다.

### Kubernetes

- Worker kubelet 중지
- containerd 중지
- 잘못된 Readiness Probe
- ImagePullBackOff
- CrashLoopBackOff
- Pending Pod

### Network

- Cilium Agent 문제
- NetworkPolicy로 통신 차단
- Egress Gateway Node 장애

### CI/CD

- Docker build 실패
- Registry 인증 실패
- 존재하지 않는 Image Tag
- Helm template 오류
- ArgoCD OutOfSync
- Sync 실패

각 장애는 다음 순서로 기록한다.

```text
증상
→ 확인 명령
→ 원인
→ 해결
→ 검증
→ 재발 방지
```

---

## Phase 4. ClickHouse / AI (후순위)

Kubernetes 및 GitOps 면접 준비가 완료된 이후 진행한다.

```text
Application
  |
Fluent Bit
  |
ClickHouse
  |
Grafana / AI Analyzer
```

이 단계에서는 로그 수집 자체보다 Kubernetes / GitOps에서 발생하는 운영 이벤트를 로그 분석과 연결하는 방향으로 확장한다.

---

## 완료 기준

이 프로젝트는 단순히 `kubectl get pods`가 정상이라고 끝내지 않는다.

다음 질문에 화이트보드로 구조를 그리고 설명할 수 있으면 완료로 본다.

1. kubeadm으로 클러스터가 어떻게 만들어지는가?
2. kubelet과 containerd는 어떤 관계인가?
3. CNI가 없으면 왜 Node가 NotReady인가?
4. kube-proxy의 역할과 Cilium replacement는 무엇이 다른가?
5. Egress Node를 왜 두었는가?
6. GitLab CI / Helm / ArgoCD의 역할을 각각 설명할 수 있는가?
7. ArgoCD 도입 전후 배포 흐름 차이는 무엇인가?
8. Sync / Drift / Self Heal / Prune을 설명할 수 있는가?
9. 장애 발생 시 어느 계층부터 확인할 것인가?
10. 이 구조를 실제 운영환경에 적용한다면 무엇을 추가하거나 변경할 것인가?
