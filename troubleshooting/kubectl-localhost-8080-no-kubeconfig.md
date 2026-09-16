# kubectl localhost:8080 연결 실패

## 발생 상황

Egress Gateway 트러블슈팅 중 `lab-e` 노드에서 다음 명령을 실행했다.

```bash
kubectl exec egress-test -- \
  curl -k -m 5 https://1.1.1.1/cdn-cgi/trace
```

## 증상

```text
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

## 원인

`lab-e`에는 Kubernetes admin kubeconfig가 설정되어 있지 않았다.

`kubectl`은 사용할 kubeconfig를 찾지 못하면 기본 API Server 주소로 `http://localhost:8080`을 시도할 수 있으며, 해당 노드에는 Kubernetes API Server가 로컬 8080 포트에 존재하지 않으므로 연결에 실패했다.

이 오류는 Cilium Egress Gateway나 Pod 네트워크 장애가 아니다.

## 해결

클러스터 관리용 kubeconfig가 설정되어 있는 `lab-m`에서 `kubectl exec`를 실행한다.

```bash
kubectl exec egress-test -- \
  curl -k -m 5 https://1.1.1.1/cdn-cgi/trace
```

Worker/Egress 노드에 불필요하게 admin kubeconfig를 배포하지 않는다.

## 검증 포인트

테스트 트래픽을 실제로 발생시킨 뒤 다음을 동시에 확인한다.

- `lab-w1`: UDP/8472 VXLAN 송신 여부
- `lab-w1` Cilium: drop monitor
- `lab-e`: VXLAN 수신 및 eth0 외부 송신 여부

## 교훈

네트워크 트러블슈팅 중에는 테스트 명령 자체가 실제로 실행되었는지 먼저 확인해야 한다. 관리 명령은 Control Plane에서 실행하고, 데이터 플레인 패킷은 각 노드의 `tcpdump`/Cilium monitor로 관찰한다.
