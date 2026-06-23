# Kubernetes 네트워크 장애 분석 및 복구 실습 프로젝트

Kubernetes 운영 환경에서 자주 발생하는 DNS, Service, Pod, NetworkPolicy, Ingress 계층의 장애를 직접 재현하고, 증상 확인 → 원인 분석 → 복구 → 검증까지 전체 과정을 수행한 트러블슈팅 실습 프로젝트입니다.

- 기간: 2026.05.27
- 참여 인원: 1인 (개인 프로젝트)
- 환경: kind 기반 Kubernetes 클러스터 (control-plane x1, worker x1), CoreDNS, ingress-nginx

---

## 프로젝트 배경

Kubernetes를 학습할 때는 Pod, Service, DNS, NetworkPolicy를 각각 독립된 기능으로 배우지만, 실제 장애 상황에서는 여러 계층이 얽혀서 같은 증상으로 나타나는 경우가 많다.

예를 들어 서비스 이름이 조회되지 않는다고 해서 항상 DNS 문제인 것은 아니다. NetworkPolicy로 ingress 트래픽이 차단되어 있거나, Pod가 정상적으로 기동되지 않아 Endpoint가 생성되지 않은 경우에도 동일한 연결 실패 증상이 나타날 수 있다.

이 프로젝트는 이런 장애 상황 5가지를 의도적으로 재현하고, 동일한 절차로 원인을 좁혀가는 트러블슈팅 역량을 기르기 위해 진행하였다. 분석에 사용한 공통 절차는 [docs/troubleshooting-playbook.md](./docs/troubleshooting-playbook.md)에 정리해 두었다.

---

## 목표

- 운영 환경에서 자주 발생하는 DNS, Service, Pod, NetworkPolicy, Ingress 관련 장애를 직접 재현
- Kubernetes 네트워크 동작 원리 이해
- 증상만으로 원인을 단정하지 않고, 단계적으로 후보를 좁혀가는 트러블슈팅 프로세스 학습

---

## 장애 시나리오

| # | 시나리오 | 증상 | 원인 | 링크 |
|---|---|---|---|---|
| 01 | NetworkPolicy Timeout | Connection timed out | NetworkPolicy ingress 차단 | [scenarios/01-networkpolicy-timeout](./scenarios/01-networkpolicy-timeout) |
| 02 | DNS Failure | Could not resolve host | CoreDNS 중지 | [scenarios/02-dns-failure](./scenarios/02-dns-failure) |
| 03 | Service Port Mismatch | DNS는 정상이나 연결 실패 | Service targetPort 불일치 | [scenarios/03-service-port-mismatch](./scenarios/03-service-port-mismatch) |
| 04 | Ingress Upstream Failure | HTTP 503 | Ingress backend service name 오타 | [scenarios/04-ingress-502](./scenarios/04-ingress-502) |
| 05 | Pod CrashLoopBackOff | Connection refused / Failed to connect | Endpoint 미생성 (CrashLoopBackOff) | [scenarios/05-pod-crashloop](./scenarios/05-pod-crashloop) |

각 시나리오 폴더에는 장애 재현용 manifest, 트러블슈팅 과정, 스크린샷, Root Cause, 복구 방법이 정리되어 있다.

---

## 빠른 진단 가이드

| 증상 | 우선 확인 항목 | 주요 명령어 |
|---|---|---|
| Connection timed out | NetworkPolicy | `kubectl get networkpolicy` |
| Could not resolve host | CoreDNS 상태 | `kubectl get pods -n kube-system` |
| DNS 정상이나 연결 실패 | Service targetPort | `kubectl describe svc` |
| HTTP 503 | Ingress backend 설정 | `kubectl describe ingress` |
| Endpoint 없음 + 연결 실패 | Pod CrashLoopBackOff | `kubectl get pods`, `kubectl logs --previous` |

전체 진단 절차와 명령어 목록은 [docs/troubleshooting-playbook.md](./docs/troubleshooting-playbook.md)에서 확인할 수 있다.

---

## 디렉토리 구조

```text
.
├── README.md
├── docs/
│   ├── architecture.md
│   └── troubleshooting-playbook.md
└── scenarios/
    ├── 01-networkpolicy-timeout/
    ├── 02-dns-failure/
    ├── 03-service-port-mismatch/
    ├── 04-ingress-502/
    └── 05-pod-crashloop/
```
