# Architecture

이 문서는 5가지 장애 시나리오를 재현하는 데 사용한 클러스터 구성과, 정상 상태에서의 네트워크 흐름을 정리한 문서이다. 각 시나리오는 아래 구성 중 한 지점에 의도적으로 장애를 주입한 것이다.

---

## 클러스터 구성

```text
kind Cluster (Local)

├── CoreDNS
├── ClusterIP Service
├── Ingress (ingress-nginx)
└── frontend / backend Pod
```

- **kind**로 생성한 로컬 Kubernetes 클러스터에서 실습을 진행하였다.
- `frontend` Pod는 `netshoot` 이미지를 사용해 `curl`, `nslookup` 등으로 연결을 테스트하는 클라이언트 역할을 한다.
- `backend` Pod는 `nginx` 이미지를 사용해 실제 요청을 처리하는 서버 역할을 한다.
- `backend` Pod 앞에는 ClusterIP Service가 있고, 외부 HTTP 접근이 필요한 시나리오(04)에서는 Ingress(ingress-nginx)를 추가로 배치하였다.

## 구성 요소별 역할

| 구성 요소 | 역할 |
|---|---|
| CoreDNS | 클러스터 내부 Service 이름을 ClusterIP로 변환 (`backend` → `10.96.x.x`) |
| ClusterIP Service | `backend` Pod를 가리키는 고정 IP/이름 제공, Selector로 대상 Pod를 찾음 |
| Endpoint | Service가 실제로 트래픽을 전달할 Pod IP 목록. Ready 상태인 Pod만 등록됨 |
| Ingress (ingress-nginx) | 외부 HTTP 요청을 받아 지정된 backend Service로 라우팅 |
| frontend Pod (netshoot) | `curl`, `nslookup`으로 연결 테스트를 수행하는 클라이언트 |
| backend Pod (nginx) | 실제 요청을 처리하는 서버 |

---

## 정상 상태 네트워크 흐름

### 클러스터 내부 통신 (frontend → backend)

```text
frontend (netshoot)
        |
        | 1. nslookup backend  →  CoreDNS가 ClusterIP로 변환
        ↓
backend service (ClusterIP)
        |
        | 2. Service가 Endpoint 목록 확인
        ↓
backend pod (nginx)
        |
        | 3. curl backend  →  정상 응답
        ↓
     200 OK
```

### 외부 HTTP 접근 (Ingress 경로, Scenario 04 기준)

```text
Client
  |
  | curl http://app.local
  ↓
Ingress (ingress-nginx)
  |
  | backend service name으로 라우팅
  ↓
backend service (ClusterIP)
  ↓
backend pod (nginx)
```

정상 상태에서는 위 두 흐름 모두 막힘없이 끝까지 이어진다. 5가지 시나리오는 이 흐름 중 한 지점을 의도적으로 끊어서 장애를 재현한 것이다.

---

## 시나리오별 장애 주입 지점

```text
frontend (netshoot)
        |
        ↓
   [DNS 조회]  ← Case 2. CoreDNS 중지
        |
        ↓
backend service (ClusterIP)
        |
        ↓
   [Service → Endpoint]  ← Case 3. targetPort 불일치
        |
        ↓
   [NetworkPolicy]  ← Case 1. ingress 차단
        |
        ↓
backend pod (nginx)  ← Case 5. CrashLoopBackOff (Endpoint 미생성)


Client
  |
  ↓
Ingress (ingress-nginx)  ← Case 4. backend service name 오타
  |
  ↓
backend service (ClusterIP)
```

| 시나리오 | 장애 주입 지점 | 정상 흐름에서 끊어지는 구간 |
|---|---|---|
| 01. NetworkPolicy Timeout | NetworkPolicy | Service → Pod 사이 (ingress 트래픽 차단) |
| 02. DNS Failure | CoreDNS | frontend → DNS 조회 단계 자체가 실패 |
| 03. Service Port Mismatch | Service.targetPort | Service → Endpoint (잘못된 포트로 전달) |
| 04. Ingress Upstream Failure | Ingress backend name | Ingress → Service (존재하지 않는 Service 참조) |
| 05. Pod CrashLoopBackOff | backend Pod | Service → Endpoint (Ready Pod 없음 → Endpoint 미생성) |

---

## 사용 기술

- Kubernetes (kind)
- Docker
- Linux (Ubuntu)
- CoreDNS
- ingress-nginx
- kubectl
