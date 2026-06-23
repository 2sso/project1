# Troubleshooting Playbook

이 문서는 본 프로젝트에서 Kubernetes 네트워크 장애를 분석할 때 공통적으로 사용한 진단 절차와 명령어를 정리한 문서이다. [scenarios/](../scenarios) 폴더의 5가지 장애 시나리오는 모두 아래 절차를 기반으로 분석하였다.

---

## 진단 절차

장애가 발생했을 때 임의로 접근하지 않고, 항상 아래 순서로 점검하였다.

1. **증상 확인** — 어떤 에러/타임아웃이 발생하는지 정확히 기록
2. **원인 후보 수립** — DNS 문제인가? Service 문제인가? Pod 문제인가? NetworkPolicy/Ingress 문제인가?
3. **원인 후보 검증** — Pod 상태, Endpoint, DNS 조회를 순서대로 확인
4. **상세 분석** — `describe`, `logs`, `events`로 세부 원인 확인
5. **원인 파악**
6. **복구 및 재검증** — 조치 후 동일한 절차로 정상 동작 확인
7. **결과 문서화**

---

## 계층별 점검 순서

같은 "연결 실패"라는 증상도 원인이 서로 다른 계층에 있을 수 있어, 아래 순서로 범위를 좁혀가며 점검하는 것이 효과적이었다.

```text
DNS → Pod → Service → Endpoint → NetworkPolicy / Ingress
```

- DNS가 정상이어도 실제 TCP 연결은 실패할 수 있다 (NetworkPolicy 차단)
- Pod, Service가 정상이어도 DNS 자체가 동작하지 않을 수 있다 (CoreDNS 중지)
- Pod가 Running 상태여도 Service 설정 오류로 통신이 안 될 수 있다 (targetPort 불일치)
- Pod, Service가 정상이어도 Ingress 설정 오류로 503이 발생할 수 있다 (backend service name 오타)
- DNS, Service가 정상이어도 Pod가 CrashLoopBackOff 상태면 Endpoint 자체가 생성되지 않는다

---

## 주요 점검 명령어

### Pod / Service / Endpoint

```bash
kubectl get pods
kubectl get svc
kubectl get endpoints

kubectl describe pod <pod-name>
kubectl describe svc <svc-name>
```

### 로그 / 이벤트

```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl get events
```

### DNS

```bash
nslookup <service-name>
```

### NetworkPolicy

```bash
kubectl get networkpolicy
kubectl describe networkpolicy <policy-name>
```

### Ingress

```bash
kubectl describe ingress <ingress-name>
```

### 연결 테스트

```bash
curl <service-name>
curl -m 5 <service-name>
ping <service-name>
```

---

## 증상별 빠른 진단 표

| 증상 | 우선 확인 항목 | 주요 명령어 | 해당 시나리오 |
|---|---|---|---|
| Connection timed out | NetworkPolicy | `kubectl get networkpolicy` | [01-networkpolicy-timeout](../scenarios/01-networkpolicy-timeout) |
| Could not resolve host | CoreDNS 상태 | `kubectl get pods -n kube-system` | [02-dns-failure](../scenarios/02-dns-failure) |
| DNS 정상이나 연결 실패 | Service targetPort | `kubectl describe svc` | [03-service-port-mismatch](../scenarios/03-service-port-mismatch) |
| HTTP 503 | Ingress backend 설정 | `kubectl describe ingress` | [04-ingress-502](../scenarios/04-ingress-502) |
| Endpoint 없음 + 연결 실패 | Pod CrashLoopBackOff | `kubectl get pods`, `kubectl logs --previous` | [05-pod-crashloop](../scenarios/05-pod-crashloop) |

---

## 정리

5가지 시나리오를 통해 확인한 핵심은, 동일한 "연결 실패" 증상이라도 원인이 DNS, Pod, Service, NetworkPolicy, Ingress 중 어디에 있는지는 케이스마다 다르다는 점이다.

특정 기술을 아는 것보다, 어떤 순서로 확인하고 검증하는지가 장애 해결 속도를 결정한다는 것을 이 프로젝트를 통해 배웠다.
