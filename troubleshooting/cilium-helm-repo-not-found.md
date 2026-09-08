# Cilium Helm repository not found

## 증상

Cilium 설치 시 다음 오류가 발생했다.

```text
Error: INSTALLATION FAILED: repo cilium not found
```

## 원인

Helm에 `cilium` chart repository가 등록되지 않은 상태에서 `cilium/cilium` chart를 설치하려고 해서 발생했다.

## 해결

공식 Cilium Helm repository를 등록한다.

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
```

등록 확인:

```bash
helm repo list
helm search repo cilium/cilium --versions | grep 1.20.1
```

이후 기존 values 파일로 설치를 다시 수행한다.

```bash
helm install cilium cilium/cilium \
  --version 1.20.1 \
  --namespace kube-system \
  -f /root/cilium-values.yaml
```

## 대안

Cilium 공식 문서에서는 OCI Registry 설치도 지원한다. 이 방식은 Helm repository 등록이 필요 없다.

```bash
helm install cilium oci://quay.io/cilium/charts/cilium \
  --version 1.20.1 \
  --namespace kube-system \
  -f /root/cilium-values.yaml
```

## 검증

```bash
helm list -n kube-system
kubectl -n kube-system get pods -o wide
kubectl get nodes -o wide
```

CNI 설치가 완료되면 `lab-m`, `lab-w1`, `lab-e` 노드가 `Ready` 상태로 전환되고 CoreDNS Pod도 `Running` 상태가 되어야 한다.
