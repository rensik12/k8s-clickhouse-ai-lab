# 08. ArgoCD Installation

## 목표

Helm으로 ArgoCD를 설치하고 GitOps 실습을 위한 기본 CD 컴포넌트가 정상 기동되는지 확인한다.

## 설치 방식

ArgoCD는 Helm Chart를 사용해 설치했다.

학습 단계에서는 핵심 GitOps 흐름에 집중하기 위해 Dex와 Notifications는 비활성화했고, ArgoCD 자체 NetworkPolicy도 생성하지 않았다.

```yaml
global:
  networkPolicy:
    create: false

dex:
  enabled: false

notifications:
  enabled: false
```

설치 명령 예시:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd \
  -n argocd \
  --create-namespace \
  -f /root/argocd-values.yaml \
  --wait \
  --timeout 10m
```

## 실제 기동 확인

2026-09-16 기준 다음 핵심 컴포넌트가 모두 `Running / Ready` 상태임을 확인했다.

```text
argocd-application-controller-0                     1/1 Running  lab-w1
argocd-applicationset-controller-5d5f67bb9d-jn6xm   1/1 Running  lab-w1
argocd-redis-7f9487d4fd-4l2zn                       1/1 Running  lab-w1
argocd-repo-server-86f77468fd-h9ksk                 1/1 Running  lab-w1
argocd-server-855c68cf89-8g2dj                      1/1 Running  lab-w1
```

서비스도 정상 생성되었다.

```text
argocd-applicationset-controller   ClusterIP   7000/TCP
argocd-redis                       ClusterIP   6379/TCP
argocd-repo-server                 ClusterIP   8081/TCP
argocd-server                      ClusterIP   80/TCP,443/TCP
```

현재 `argocd-server`는 `ClusterIP` 타입이므로 클러스터 외부에서 바로 접속할 수 없다.

## 다음 단계

1. 초기 admin password 확인
2. Port Forward로 ArgoCD UI 접속
3. Sample FastAPI Application 준비
4. Helm Chart 작성 및 수동 배포
5. Source Repository / Deployment Repository 분리
6. ArgoCD Application 생성
7. Auto Sync / Self Heal / Prune 검증
8. Git 기반 Rollback 및 실패 배포 복구 실습

## 면접 포인트

ArgoCD 설치 자체보다 중요한 것은 아래 흐름을 설명할 수 있는 것이다.

```text
Developer Commit
      ↓
CI Build / Image Push
      ↓
Deployment Repository 변경
      ↓
ArgoCD가 Git Desired State 감지
      ↓
Kubernetes Live State와 비교
      ↓
Sync
```

GitLab CI가 Kubernetes API에 직접 배포하지 않고 Git에는 원하는 배포 상태만 기록하며, ArgoCD가 Pull 방식으로 이를 Kubernetes에 반영하는 구조를 목표로 한다.
