# S&P 500 Daily 주가 분석 — 소프트웨어 아키텍처 (Software Architecture)

> 기존 스펙(`data_pipeline.md`, `schema.md`, `agent_trend.md`, `frontend.md`)에서 **명시되지 않은**  
> 시스템 수준 설계 결정을 모두 이 문서에 기록한다.  
> 참조 공통 스펙: `common/backend_fastapi.md`, `common/auth_keycloak.md`, `common/caching_strategy.md`,  
> `common/security.md`, `common/infra_cicd.md`, `common/observability_otel_opensearch.md`

---

## 1. 컴포넌트 전체 맵

```
┌─────────────────────────────────────────────────────────────────┐
│  외부                                                            │
│  Alpha Vantage / yfinance API  │  OpenAI API                   │
└────────────────┬───────────────┴──────────┬────────────────────┘
                 │                           │
        (Airflow DAG)               (LangChain Agent)
                 │                           │
┌────────────────▼───────────────────────────▼────────────────────┐
│  Data Platform                                                   │
│  Airflow  ──►  Raw Storage (Parquet)                            │
│            ──►  PostgreSQL  (ODS / DW / Mart)                   │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                        (REST API, FastAPI)
                                 │
┌────────────────────────────────▼────────────────────────────────┐
│  Backend API Server                                              │
│  FastAPI  +  Redis (캐시)  +  Keycloak (인증)                   │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                        (HTTPS / JSON)
                                 │
┌────────────────────────────────▼────────────────────────────────┐
│  Frontend                                                        │
│  React SPA  (CSR, Zustand, Axios)                               │
└─────────────────────────────────────────────────────────────────┘

[횡단 관심사]
  OTel Collector ──► Tempo(트레이스), Prometheus(메트릭), OpenSearch(로그)
  Slack 알림 ──► Airflow 실패, API 에러율 초과, LLM 분석 실패
```

---

## 2. 백엔드 API 서버 (FastAPI)

### 2-1. 계층 구조

```
app/
├── main.py                  ← FastAPI 앱 생성, 미들웨어·라우터 등록
├── routers/
│   ├── ods.py               ← GET /api/v1/ods/prices
│   ├── dw.py                ← GET /api/v1/dw/sector
│   ├── mart.py              ← GET /api/v1/mart/key-index, trend-analysis
│   └── meta.py              ← GET /api/v1/meta/trading-dates
├── services/
│   ├── ods_service.py       ← ODS 조회 로직
│   ├── dw_service.py        ← DW 집계 조회 로직
│   ├── mart_service.py      ← Mart 지표·트렌드 조회 로직
│   └── calendar_service.py  ← 거래일 캘린더
├── repositories/
│   ├── ods_repo.py          ← SQL 직접 실행 (ODS 테이블)
│   ├── dw_repo.py
│   └── mart_repo.py
├── schemas/
│   ├── ods.py               ← Pydantic 요청·응답 모델
│   ├── dw.py
│   └── mart.py
├── core/
│   ├── config.py            ← pydantic-settings 환경변수 로드
│   ├── database.py          ← SQLAlchemy 비동기 엔진·세션
│   ├── cache.py             ← Redis 연결·헬퍼
│   ├── security.py          ← JWT 검증 (Keycloak JWKS)
│   └── exceptions.py        ← 공통 예외 클래스
└── dependencies.py          ← FastAPI Depends 의존성 함수
```

### 2-2. 라우터 패턴

```python
# routers/mart.py
router = APIRouter(prefix="/api/v1/mart", tags=["mart"])

@router.get("/key-index", response_model=KeyIndexResponse)
async def get_key_index(
    trade_date: date = Query(...),
    sector_code: str | None = Query(None),
    service: MartService = Depends(),
    _: TokenPayload = Depends(verify_token),   # 인증 필수
):
    return await service.get_key_index(trade_date, sector_code)
```

### 2-3. DB 접근 (비동기 SQLAlchemy)

```python
# core/database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

engine = create_async_engine(
    settings.database_url,          # postgresql+asyncpg://...
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,
    pool_recycle=3600,
)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSession(engine) as session:
        yield session
```

**트랜잭션 경계**:  
- 조회 전용 API는 트랜잭션 없이 `autocommit=True` 세션 사용  
- Airflow DAG의 적재 작업은 DAG 태스크 단위로 세션을 열고 닫음

---

## 3. 인증·인가

> 참조: `common/auth_keycloak.md`

### 3-1. 흐름

```
[Browser]
  1. Keycloak 로그인 (Auth Code + PKCE)
  2. Access Token 수신 (JWT, 만료 5분)
  3. API 요청마다 Authorization: Bearer <token>

[FastAPI]
  4. JWKS 엔드포인트에서 Keycloak 공개키 조회 (캐시 1시간)
  5. JWT 서명 · 만료 · issuer 검증
  6. 토큰의 realm_access.roles 로 API 접근 권한 판단
```

### 3-2. 역할 정의

| Keycloak Role | 접근 가능 레이어 |
|--------------|----------------|
| `viewer` | ODS, DW, Mart Key Index (읽기 전용) |
| `analyst` | 위 + Mart Trend Analysis |
| `admin` | 위 + 메타 관리 API |

### 3-3. FastAPI 검증 의존성

```python
# core/security.py
async def verify_token(
    credentials: HTTPAuthorizationCredentials = Depends(HTTPBearer()),
) -> TokenPayload:
    token = credentials.credentials
    jwks = await get_jwks_cached()          # Redis TTL 1시간
    payload = jwt.decode(token, jwks, algorithms=["RS256"],
                         audience=settings.keycloak_client_id)
    return TokenPayload(**payload)

def require_role(role: str):
    def checker(token: TokenPayload = Depends(verify_token)):
        if role not in token.realm_access.roles:
            raise HTTPException(status_code=403)
    return Depends(checker)
```

---

## 4. 캐싱 전략

> 참조: `common/caching_strategy.md`

### 4-1. 캐시 대상 및 TTL

| 캐시 대상 | Redis 키 패턴 | TTL | 갱신 트리거 |
|----------|-------------|-----|------------|
| ODS 종목 목록 (날짜별) | `sp500:v1:ods:prices:{date}:{sector}:{page}` | 6시간 | 파이프라인 완료 후 무효화 |
| DW Sector 집계 (날짜별) | `sp500:v1:dw:sector:{date}` | 6시간 | 파이프라인 완료 후 무효화 |
| Mart Key Index (날짜별) | `sp500:v1:mart:index:{date}` | 6시간 | 파이프라인 완료 후 무효화 |
| Mart Key Index 시계열 | `sp500:v1:mart:trend:{sector}:{days}` | 6시간 | 파이프라인 완료 후 무효화 |
| Mart Trend Analysis | `sp500:v1:mart:llm:{date}` | 12시간 | Agent 완료 후 무효화 |
| 거래일 목록 | `sp500:v1:meta:trading-dates` | 24시간 | 주간 배치 갱신 |
| JWKS (Keycloak 공개키) | `sp500:v1:auth:jwks` | 1시간 | TTL 만료 시 자동 갱신 |

### 4-2. Cache-Aside 패턴

```python
# core/cache.py
async def cached(key: str, ttl: int, fetch_fn):
    hit = await redis.get(key)
    if hit:
        return orjson.loads(hit)
    result = await fetch_fn()
    await redis.setex(key, ttl, orjson.dumps(result))
    return result

# Airflow DAG 완료 시 캐시 무효화
async def invalidate_pipeline_cache(trade_date: str):
    patterns = [
        f"sp500:v1:ods:prices:{trade_date}:*",
        f"sp500:v1:dw:sector:{trade_date}",
        f"sp500:v1:mart:index:{trade_date}",
    ]
    for pattern in patterns:
        keys = await redis.keys(pattern)
        if keys:
            await redis.delete(*keys)
```

---

## 5. 외부 API Rate Limit 처리 (주가 수집)

기존 `data_pipeline.md`에 "rate limit 준수"만 언급되어 있어 구체적 구현을 정의한다.

### 5-1. 처리 방식

```python
# Airflow Task: fetch_all_sp500_ohlcv
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

RATE_LIMIT_RPM = 5       # Alpha Vantage 무료 플랜: 5 req/min
SEMAPHORE = asyncio.Semaphore(5)

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=12, max=60))
async def fetch_single_ticker(session, ticker: str, trade_date: str) -> dict:
    async with SEMAPHORE:
        async with session.get(build_url(ticker, trade_date)) as resp:
            resp.raise_for_status()
            await asyncio.sleep(12)   # 60s / 5req = 12s 간격
            return await resp.json()

async def fetch_all_sp500_ohlcv(tickers: list[str], trade_date: str):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_single_ticker(session, t, trade_date) for t in tickers]
        results = await asyncio.gather(*tasks, return_exceptions=True)
    # 실패한 종목은 별도 quarantine 테이블에 기록
    return [r for r in results if not isinstance(r, Exception)]
```

### 5-2. API 공급사별 한도 비교 (미결 과제 연동)

| 공급사 | 무료 한도 | 유료 최소 플랜 | 500 종목 소요 시간 |
|--------|---------|-------------|-----------------|
| Alpha Vantage | 5 req/min, 500/day | $50/mo → 75 req/min | 무료: ~100min / 유료: ~7min |
| yfinance | 제한 없음 (비공식) | — | ~3min (병렬) |
| Polygon.io | 5 req/min (무료) | $29/mo → 무제한 | 무료: ~100min |

> API 공급사 확정 후 rate limit 수치를 이 문서에 반영한다.

---

## 6. 보안

> 참조: `common/security.md`

### 6-1. 전송 보안

- 모든 외부 통신: TLS 1.3 강제
- Ingress 수준에서 HTTP→HTTPS 리다이렉트
- OpenAI API 키, DB 접속 정보: Kubernetes Secret (Sealed Secrets 또는 Vault)

### 6-2. 보안 HTTP 헤더 (FastAPI 미들웨어)

```python
SECURITY_HEADERS = {
    "Strict-Transport-Security": "max-age=63072000; includeSubDomains",
    "X-Content-Type-Options": "nosniff",
    "X-Frame-Options": "DENY",
    "Content-Security-Policy": "default-src 'self'",
    "Referrer-Policy": "strict-origin-when-cross-origin",
}
```

### 6-3. 환경변수·시크릿 목록

| 변수 | 분류 | 주입 방법 |
|------|------|----------|
| `DATABASE_URL` | 시크릿 | K8s Secret |
| `REDIS_URL` | 시크릿 | K8s Secret |
| `OPENAI_API_KEY` | 시크릿 | K8s Secret |
| `STOCK_API_KEY` | 시크릿 | K8s Secret |
| `KEYCLOAK_URL` | 설정 | ConfigMap |
| `KEYCLOAK_REALM` | 설정 | ConfigMap |
| `KEYCLOAK_CLIENT_ID` | 설정 | ConfigMap |
| `ENV` | 설정 | ConfigMap |
| `LOG_LEVEL` | 설정 | ConfigMap |

### 6-4. 데이터 보안

- PostgreSQL: 테이블별 role 분리 (`reader`, `writer`, `admin`)
- Airflow Connection은 암호화 저장 (`AIRFLOW__CORE__FERNET_KEY`)
- LLM 입력 데이터에 개인정보 없음 (주가·Sector 지표만 포함)

---

## 7. 에러 처리 및 장애 복구

### 7-1. API 에러 응답 포맷 (공통)

```python
# core/exceptions.py
class AppException(Exception):
    def __init__(self, status_code: int, code: str, message: str):
        self.status_code = status_code
        self.code = code
        self.message = message

# 응답 포맷
{
  "error": {
    "code": "DATA_NOT_FOUND",
    "message": "해당 날짜의 데이터가 존재하지 않습니다.",
    "trade_date": "2026-05-01"
  }
}
```

### 7-2. 에러 코드 정의

| 코드 | HTTP | 상황 |
|------|------|------|
| `DATA_NOT_FOUND` | 404 | 해당 날짜 데이터 없음 |
| `NON_TRADING_DAY` | 404 | 비거래일 |
| `ANALYSIS_PENDING` | 202 | LLM Agent 분석 미완료 |
| `ANALYSIS_FAILED` | 503 | LLM Agent 분석 실패 |
| `UPSTREAM_TIMEOUT` | 504 | 외부 주가 API 타임아웃 |
| `UNAUTHORIZED` | 401 | 토큰 없음/만료 |
| `FORBIDDEN` | 403 | 역할 권한 부족 |

### 7-3. 파이프라인 장애 복구 절차

| 장애 상황 | 감지 방법 | 복구 방법 |
|----------|----------|----------|
| 주가 API 타임아웃 | DAG Task 실패 알림 | 자동 재시도 3회 → 수동 `airflow tasks run` 재실행 |
| ODS 적재 실패 | DQ 로그 + Slack | 원인 수정 후 날짜 파티션 단위 backfill |
| DW 집계 오류 | DQ 검증 fail + Slack | ODS 확인 후 `build_dw_sector` 수동 재실행 |
| LLM 분석 실패 | `mart_sector_trend_analysis` 미적재 | `run_agent_trend` 수동 재실행 또는 sentinel 값 유지 |
| DB 연결 장애 | API 헬스체크 fail | K8s Readiness probe → LB에서 제외, DB 복구 후 자동 복귀 |
| Redis 장애 | 캐시 miss 급증 / 알림 | Cache-Aside: Redis 없이 DB 직접 조회로 Degraded 운영 |

---

## 8. 관측성 (Observability)

> 참조: `common/observability_otel_opensearch.md`

### 8-1. 계측 포인트

| 대상 | 계측 항목 | 방법 |
|------|----------|------|
| FastAPI 전체 HTTP 요청 | latency, status code, path | FastAPIInstrumentor 자동 |
| DB 쿼리 | 쿼리 소요시간, 쿼리 문자열 | SQLAlchemy OTLP 자동 |
| Redis 조회 | 캐시 히트/미스율, latency | 수동 Span |
| 외부 주가 API 호출 | 응답시간, 상태코드, 종목 수 | httpx 자동 + 수동 속성 |
| LLM 호출 (Agent) | 토큰 수, 응답시간, Sector | LangChain 콜백 + 수동 Span |
| Airflow DAG/Task | 실행 시간, 성공/실패 | Airflow StatsD → Prometheus |

### 8-2. 핵심 메트릭

```
# Prometheus 메트릭 (예시)
sp500_api_request_duration_seconds_bucket{endpoint, method, status}
sp500_cache_hits_total{cache_key_prefix}
sp500_cache_misses_total{cache_key_prefix}
sp500_pipeline_dag_duration_seconds{dag_id}
sp500_llm_tokens_total{model, sector, type}   # type: input|output
sp500_stock_ingest_count{trade_date, status}   # status: success|failed
```

### 8-3. 알림 규칙

| 알림 | 조건 | 채널 |
|------|------|------|
| 파이프라인 DAG 실패 | DAG status = failed | Slack `#data-alerts` |
| API 에러율 급증 | 5xx 비율 > 5% (5분 기준) | Slack `#platform-alerts` |
| LLM 분석 미완료 | 09:30 UTC까지 11개 미만 적재 | Slack `#data-alerts` |
| DB 연결 부족 | pool 사용률 > 80% | Slack `#platform-alerts` |
| 종목 수집 부족 | 수집 종목 < 480 | Slack `#data-alerts` |

### 8-4. 로그 구조 (structlog)

```python
log.info("pipeline_task_complete",
    dag_id="load_raw_to_ods",
    trade_date="2026-05-01",
    records_loaded=503,
    duration_sec=42.3,
    batch_id="batch-abc123",
)
```

---

## 9. 환경 구성 (dev / staging / prod)

### 9-1. 환경별 차이

| 항목 | dev | staging | prod |
|------|-----|---------|------|
| 주가 API 호출 | yfinance(무료) 또는 mock 파일 | yfinance 또는 Alpha Vantage | Alpha Vantage (유료) |
| LLM 호출 | gpt-4o-mini (저비용) | gpt-4o-mini | gpt-4o |
| DB | 로컬 PostgreSQL | 공용 스테이징 DB | 프로덕션 DB |
| Redis | 로컬 | 공용 스테이징 Redis | 프로덕션 Redis |
| Airflow | 로컬 또는 Docker Compose | 스테이징 클러스터 | 프로덕션 클러스터 |
| 수집 대상 | 10개 종목 (빠른 반복) | 전체 500개 | 전체 500개 |
| 인증 | Keycloak dev realm | Keycloak staging realm | Keycloak prod realm |

### 9-2. 환경별 설정 파일

```
config/
├── .env.dev          ← 로컬 개발용
├── .env.staging      ← CI/CD 스테이징 시크릿 주입
└── .env.prod         ← K8s Secret 주입 (파일로 보관 금지)

helm/
├── values.yaml           ← 공통 기본값
├── values-dev.yaml
├── values-staging.yaml
└── values-prod.yaml
```

---

## 10. 배포 구조

> 참조: `common/infra_cicd.md`, `common/deployment_strategy.md`

### 10-1. 컨테이너 구성

| 컨테이너 | 이미지 기반 | 포트 |
|---------|-----------|------|
| `sp500-api` | `python:3.11-slim` (멀티스테이지) | 8000 |
| `sp500-frontend` | `node:20-alpine` → nginx | 80 |
| `sp500-airflow-webserver` | `apache/airflow:2.9` | 8080 |
| `sp500-airflow-worker` | `apache/airflow:2.9` | — |
| `postgres` | `postgres:15` | 5432 |
| `redis` | `redis:7-alpine` | 6379 |

### 10-2. K8s 리소스 요약

```yaml
# sp500-api Deployment (prod)
resources:
  requests: { cpu: "250m", memory: "256Mi" }
  limits:   { cpu: "1000m", memory: "1Gi" }
replicas: 2
strategy:
  type: RollingUpdate
  rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
```

### 10-3. CI/CD 흐름 (`branch_strategy.md` 연동)

```
feature/* → PR → develop
  └─ lint (ruff, black)
  └─ unit test
  └─ type check (mypy)

develop → PR → main
  └─ 위 + integration test (testcontainers)

main push
  └─ Docker build + push (tag: git-sha)
  └─ dev 자동 배포
  └─ E2E 테스트 (dev 환경)
  └─ staging 1인 승인 배포
  └─ prod 2인 승인 + Canary 배포
```

### 10-4. 헬스체크 엔드포인트

| 경로 | 역할 | 성공 조건 |
|------|------|----------|
| `GET /healthz` | Liveness | 항상 200 |
| `GET /ready` | Readiness | DB + Redis 연결 정상 |
| `GET /api/v1/meta/trading-dates` | 스모크 | 거래일 목록 반환 200 |

---

## 11. API 버전 관리

| 항목 | 기준 |
|------|------|
| 현재 버전 | `v1` (경로: `/api/v1/...`) |
| 버전 업 조건 | 응답 스키마 Breaking Change 발생 시 |
| 하위 호환 기간 | v2 출시 후 v1을 최소 3개월 유지 |
| 버전 공지 | `Deprecation` 헤더로 클라이언트에 사전 고지 |
| 프롬프트 버전 | `mart_sector_trend_analysis.prompt_version` 컬럼으로 관리 |

---

## 12. 미결 기술 과제

| 과제 | 우선순위 | 담당 |
|------|---------|------|
| 주가 API 공급사 확정 (비용·한도 기준) | P0 | 데이터 엔지니어링팀 |
| Raw 스토리지 방식 확정 (로컬 Parquet vs S3) | P0 | 인프라팀 |
| Redis Cluster vs Standalone 선택 | P1 | 인프라팀 |
| DB Connection Pool 크기 최적화 (부하 테스트 후) | P1 | 백엔드팀 |
| Airflow 메타 DB 분리 여부 (현재 같은 PostgreSQL) | P1 | 데이터 엔지니어링팀 |
| Sealed Secrets vs Vault 선택 | P1 | 인프라팀 |
| LLM 비용 초과 시 자동 차단 정책 | P1 | AI팀 |
| 캐시 무효화 이벤트 전달 방식 (HTTP 훅 vs 메시지) | P2 | 백엔드팀 |

---

## 관련 문서

| 파일 | 설명 |
|------|------|
| `overview.md` | 서비스 개요 및 전체 아키텍처 다이어그램 |
| `business_spec.md` | 비즈니스 요건, 시나리오, 성공 기준 |
| `schema.md` | 전체 레이어 테이블 DDL |
| `data_pipeline.md` | Airflow DAG 설계 및 수집 상세 |
| `agent_trend.md` | LLM Agent 프롬프트·실행 구조 |
| `frontend.md` | 화면 구성·API 계약·UI 상세 |
