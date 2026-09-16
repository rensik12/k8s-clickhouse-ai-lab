# Egress Gateway 외부 통신 조사 - VXLAN ingress 가설 기각

## 상태

초기 가설 기각 / 조사 계속 진행

## 증상

`egress-test` Pod는 정책 적용 전 외부 통신이 정상이며 공인 IP `211.47.70.204`가 확인되었다.

`CiliumEgressGatewayPolicy` 적용 후 동일 테스트에서 응답이 지연되어 datapath를 추적했다.

## 확인 결과

테스트 Pod:

```text
Pod IP : 10.200.1.143
Node   : lab-w1
```

BPF Egress Map은 정상적으로 `lab-e(192.168.184.179)`를 Gateway로 선택했다.

`lab-w1`에서 실제 VXLAN 송신 확인:

```text
192.168.184.163:<ephemeral> > 192.168.184.179.8472
inner packet: 10.200.1.143:<ephemeral> > 1.1.1.1.443 SYN
```

이후 `lab-e`에서도 같은 패킷이 실제 ingress로 확인되었다.

```text
192.168.184.163:<ephemeral> > 192.168.184.179.8472
inner packet: 10.200.1.143:<ephemeral> > 1.1.1.1.443 SYN
```

따라서 OpenStack Security Group 또는 UDP/8472 차단 가설은 기각했다.

## SNAT 확인

`lab-e` tcpdump에서 decapsulation 이후 Cilium이 Pod Source IP를 Gateway Node IP로 SNAT한 것도 확인했다.

```text
Before SNAT:
10.200.1.143:<ephemeral> > 1.1.1.1.443 SYN

After SNAT:
192.168.184.179:<ephemeral> > 1.1.1.1.443 SYN
```

즉 다음 구간까지는 정상이다.

```text
egress-test Pod
  -> lab-w1 Cilium policy match
  -> VXLAN UDP/8472 encapsulation
  -> lab-e ingress
  -> VXLAN decapsulation
  -> SNAT 10.200.1.143 -> 192.168.184.179
  -> eth0 외부 송신
```

## 잘못된 테스트 조건

추적 과정에서 `https://1.1.1.1:443`를 연결 테스트 대상으로 사용했으나, 해당 환경에서는 원래 `1.1.1.1:443` 통신이 되지 않는 것으로 확인되었다.

따라서 SYN 응답이 없다는 사실만으로 Cilium Egress Gateway 또는 OpenStack NAT 장애라고 판단할 수 없다.

정책 적용 전 실제 성공했던 동일 endpoint인 `ifconfig.me`를 기준으로 최종 검증해야 한다.

## 다음 검증

```bash
# lab-e 자체 외부 Source IP
curl -4 -m 5 -s ifconfig.me ; echo

# 정책 대상 Pod 외부 Source IP
kubectl exec egress-test -- curl -4 -m 5 -s ifconfig.me ; echo
```

목표 결과:

```text
lab-e       -> 211.47.73.206
egress-test -> 211.47.73.206
```

## 배운 점

- 패킷 캡처 시 테스트 목적지 자체가 정상 통신 가능한 endpoint인지 먼저 검증해야 한다.
- `tcpdump`에서 VXLAN outer packet과 inner packet을 함께 확인하면 source node에서 gateway node까지의 datapath를 명확하게 검증할 수 있다.
- Gateway node에서 SNAT 전/후 패킷을 동시에 확인하면 Cilium Egress Gateway 동작 여부를 네트워크 상에서 직접 검증할 수 있다.
- 초기 가설이 틀렸다면 그대로 확정하지 않고 추가 캡처 결과에 따라 문서를 수정한다.
