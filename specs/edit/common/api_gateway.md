# Common Spec — API Gateway

---

## 역할과 책임

```
[클라이언트]
    │
    ▼
[API Gateway]          ← 단일 진입점
    ├─ TLS 종료
    ├─ 인증 토큰 검증 (Keycloak JWKS)
    ├─ Rate Limiting
    ├─ 라우팅
    ├─ 로드 밸런싱
    ├─ 요청/응답 변환
    └─ 서킷 브레이커
    │
    ├─► Backend API (FastAPI / Spring Boot)
    ├─► Auth Service (Keycloak)
    └─► Static Assets (CDN)
```

---

## 게이트웨이 선택 비교

| 항목 | Kong | nginx + Lua | AWS API Gateway | Traefik |
|------|------|------------|----------------|---------|
| 배포 형태 | 자체 호스팅 / 클라우드 | 자체 호스팅 | 완전 관리형 | 자체 호스팅 |
| 플러그인 생태계 | 풍부 | 제한적 | 제한적 | 중간 |
| K8s 통합 | Kong Ingress Controller | nginx Ingress | ALB Ingress | 네이티브 |
| Keycloak 연동 | OIDC 플러그인 | lua-resty-openidc | Lambda Authorizer | ForwardAuth |
| 학습 곡선 | 중간 | 높음 | 낮음 | 낮음 |
| 권장 상황 | 마이크로서비스, 풍부한 플러그인 필요 | 고성능 정적 규칙 | AWS 인프라 | K8s 단순 라우팅 |

---

## 라우팅 규칙

### Kong Ingress (K8s)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ice-breaker-ingress
  annotations:
    konghq.com/strip-path: "true"
    konghq.com/plugins: rate-limiting,oidc-auth
spec:
  ingressClassName: kong
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1/process
            pathType: Prefix
            backend:
              service: { name: api-service, port: { number: 8000 } }
          - path: /v1/admin
            pathType: Prefix
            backend:
              service: { name: admin-service, port: { number: 8001 } }
```

### 라우팅 우선순위

| 우선순위 | 경로 패턴 | 대상 서비스 |
|---------|----------|-----------|
| 1 | `/auth/*` | Keycloak |
| 2 | `/v1/admin/*` | Admin API (인증 필수) |
| 3 | `/v1/*` | Backend API (인증 필수) |
| 4 | `/healthz` | 헬스체크 (인증 불필요) |
| 5 | `/*` | Frontend 정적 파일 |

---

## Rate Limiting

### 전략

| 알고리즘 | 특징 | 적합한 상황 |
|---------|------|------------|
| Fixed Window | 단순, 구현 쉬움 | 트래픽 패턴 단순 |
| Sliding Window | 경계 버스트 방지 | 일반 API |
| Token Bucket | 순간 버스트 허용 | LLM API 등 비용 민감 |
| Leaky Bucket | 균일한 처리율 | 백엔드 보호 |

### Kong Rate Limiting 플러그인

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limiting
plugin: rate-limiting
config:
  minute: 20           # 분당 20회
  hour: 200            # 시간당 200회
  policy: redis        # 분산 환경 — Redis 기반 공유 카운터
  redis_host: redis
  redis_port: 6379
  hide_client_headers: false
  error_message: "요청이 너무 많습니다. 잠시 후 다시 시도해주세요."
```

### 사용자 등급별 제한

| 등급 | 분당 요청 | 시간당 요청 | Keycloak 역할 |
|------|----------|-----------|--------------|
| Anonymous | 5 | 20 | 없음 |
| Free | 20 | 200 | `user` |
| Pro | 100 | 1000 | `user-pro` |
| Admin | 무제한 | 무제한 | `admin` |

---

## 인증 통합 (Keycloak)

### Gateway 레벨 JWT 검증

```yaml
# Kong OIDC 플러그인
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: oidc-auth
plugin: openid-connect
config:
  issuer: https://keycloak.example.com/realms/ice-breaker
  client_id: backend-api
  auth_methods: [bearer]
  bearer_token_param_type: [header]
  # 검증 후 헤더로 사용자 정보 전달
  upstream_headers_claims:
    - sub          → X-User-Id
    - email        → X-User-Email
    - realm_access → X-User-Roles
```

### 인증 필요 / 불필요 경로 분리

```yaml
# 인증 불필요 경로에는 oidc-auth 플러그인 미적용
- path: /healthz       # 헬스체크
- path: /metrics       # Prometheus (내부망만 허용)
- path: /docs          # API 문서 (개발 환경만)
```

---

## 서킷 브레이커

```yaml
# Kong Circuit Breaker 플러그인
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: circuit-breaker
plugin: circuit-breaker
config:
  threshold: 50          # 오류율 50% 초과 시 Open
  min_calls: 10          # 최소 호출 수 (통계 유의미성)
  timeout: 30            # Open 상태 유지 시간 (초)
  half_open_calls: 3     # Half-Open 상태에서 허용 호출 수
```

| 상태 | 동작 |
|------|------|
| Closed | 정상 — 모든 요청 통과 |
| Open | 차단 — 즉시 503 반환 |
| Half-Open | 탐색 — 소수 요청 허용 후 상태 재평가 |

---

## 요청/응답 변환

```yaml
# Kong 요청 변환 — 헤더 추가·제거
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: request-transform
plugin: request-transformer
config:
  add:
    headers:
      - X-Request-Id:$(uuid)           # 요청 ID 주입
      - X-Gateway-Version:1.0
  remove:
    headers:
      - X-Internal-Debug               # 내부 디버그 헤더 제거 (외부 노출 방지)
```

---

## 로깅·관찰성

```yaml
# Kong → OpenTelemetry 익스포터
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: opentelemetry
plugin: opentelemetry
config:
  endpoint: http://otel-collector:4318/v1/traces
  resource_attributes:
    service.name: api-gateway
```

Gateway에서 생성한 `traceparent` 헤더가 백엔드까지 전파되어 전체 요청 추적 가능.

---

## 미결 기술 과제

- [ ] Kong vs Traefik 최종 선택 — K8s 환경 PoC 후 결정
- [ ] Rate Limiting Redis 클러스터 HA 구성
- [ ] API 버전 전환 전략 — `/v1` → `/v2` 동시 운영 기간 정의
- [ ] GraphQL 지원 필요 여부 결정
- [ ] 게이트웨이 자체 장애 시 Fallback — 직접 백엔드 접근 차단 여부
