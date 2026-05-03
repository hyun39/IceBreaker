# Versions — Common Spec SW 버전 정리

> 작성 기준일: 2026-05-03  
> 각 항목의 **출처**는 `common/` 디렉토리의 해당 스펙 파일을 가리킨다.  
> 버전 미확정 항목은 `TBD`로 표기하고, 명시된 최소 버전은 `+`로 표기한다.

---

## 언어·런타임

| SW | 버전 | 출처 |
|----|------|------|
| Python | 3.11+ | `backend_fastapi.md` |
| Java | 21 (LTS) | `backend_springboot.md` |

---

## 웹 프레임워크·서버

| SW | 버전 | 비고 | 출처 |
|----|------|------|------|
| FastAPI | TBD (latest) | ASGI 프레임워크 | `backend_fastapi.md` |
| Uvicorn | TBD (latest) | FastAPI ASGI 서버 | `backend_fastapi.md` |
| Spring Boot | 3.x | RestClient 사용 → 3.2+ 필요 | `backend_springboot.md` |
| Spring Framework | 6.1+ | RestClient 도입 버전 | `backend_springboot.md` |

---

## AI / LLM 오케스트레이션

| SW | 버전 | 비고 | 출처 |
|----|------|------|------|
| LangChain (Python) | TBD (latest) | ReAct 에이전트, 체인, 콜백 | `agent.md`, `backend_fastapi.md`, `rag.md` |
| LangChain4j (Java) | TBD — 미확정 | ReAct 에이전트 호환성 검증 필요 | `backend_springboot.md` |

---

## LLM 모델

| 모델 | 제공사 | 용도 | 출처 |
|------|--------|------|------|
| gpt-4o | OpenAI | 복잡한 다단계 추론 | `agent.md` |
| gpt-4o-mini | OpenAI | 단순 URL 탐색 에이전트 | `agent.md` |
| gpt-3.5-turbo | OpenAI | 포맷 변환 체인 (낮은 비용) | `agent.md`, `observability_otel_opensearch.md` |
| claude-opus | Anthropic | 복잡한 다단계 추론 | `agent.md` |
| claude-haiku | Anthropic | 단순 반복 검색 작업 | `agent.md` |

---

## 임베딩 모델

| 모델 | 제공사 | 차원 | 비고 | 출처 |
|------|--------|------|------|------|
| text-embedding-3-small | OpenAI | 1536 | 권장 — 비용↓, 다국어 지원 | `rag.md` |
| text-embedding-3-large | OpenAI | 3072 | 정확도↑, 비용↑ | `rag.md` |
| text-embedding-ada-002 | OpenAI | 1536 | 구형 — 레거시 호환용만 | `rag.md` |
| bge-m3 | BAAI (오픈소스) | 1024 | 로컬 실행, 한국어 강점 | `rag.md` |
| cross-encoder/ms-marco-MiniLM | Hugging Face | — | 리랭킹용 | `rag.md` |
| BGE-Reranker | BAAI | — | 리랭킹 — 한국어 성능 양호 | `rag.md` |

---

## 벡터 스토어

| SW | 버전 | 배포 형태 | 비고 | 출처 |
|----|------|----------|------|------|
| Chroma | TBD (latest) | 로컬 / 서버 | 개발 환경 권장 | `rag.md` |
| Pinecone | TBD (latest) | 클라우드 완전관리형 | 프로덕션 확장 | `rag.md` |
| Weaviate | TBD (latest) | 자체 호스팅 / 클라우드 | 하이브리드 검색 내장 | `rag.md` |
| pgvector | TBD (latest) | PostgreSQL 확장 | 기존 PG 환경 통합 | `rag.md` |
| Qdrant | TBD (latest) | 자체 호스팅 / 클라우드 | Rust 기반 고성능 | `rag.md` |

---

## 데이터베이스

| SW | 버전 | 비고 | 출처 |
|----|------|------|------|
| PostgreSQL | 15 | 기준 DB (JSONB, Partial Index) | `database.md`, `backend_springboot.md` |
| MySQL | 8.0+ | JSON 타입 지원 최소 버전 | `database.md` |
| SQLite | TBD | 개발·테스트 경량 환경 | `database.md` |
| MongoDB | TBD (latest) | profile_snapshot 비정형 저장 대안 | `database.md` |
| Redis | TBD (latest) | 응답 캐싱 (fastapi-cache2, Spring Cache) | `backend_fastapi.md`, `backend_springboot.md` |

---

## Python 패키지 (FastAPI 스택)

| 패키지 | 버전 | 용도 | 출처 |
|--------|------|------|------|
| pydantic | v2 | 요청·응답 검증·직렬화 | `backend_fastapi.md` |
| pydantic-settings | TBD (latest) | 환경변수 타입 안전 로드 | `backend_fastapi.md` |
| httpx | TBD (latest) | 비동기 HTTP 클라이언트 | `backend_fastapi.md`, `auth_keycloak.md` |
| tenacity | TBD (latest) | 지수 백오프 재시도 | `backend_fastapi.md` |
| structlog | TBD (latest) | 구조화 로그 (JSON) | `backend_fastapi.md`, `observability_otel_opensearch.md` |
| python-jose | TBD (latest) | JWT 검증 (Keycloak JWKS) | `auth_keycloak.md` |
| pytest | TBD (latest) | 단위·통합 테스트 | `backend_fastapi.md` |
| testcontainers-python | TBD (latest) | DB 통합 테스트 컨테이너 | `backend_fastapi.md` |
| fastapi-cache2 | TBD (latest) | Redis 기반 응답 캐싱 | `backend_fastapi.md` |
| ragas | TBD (latest) | RAG 평가 (Faithfulness, Relevancy) | `rag.md` |
| opentelemetry-sdk | TBD (latest) | OTel 트레이싱 코어 | `observability_otel_opensearch.md` |
| opentelemetry-instrumentation-fastapi | TBD (latest) | FastAPI 자동 계측 | `observability_otel_opensearch.md` |
| opentelemetry-instrumentation-httpx | TBD (latest) | httpx 외부 호출 자동 계측 | `observability_otel_opensearch.md` |
| opentelemetry-exporter-otlp | TBD (latest) | OTel Collector OTLP 익스포터 | `observability_otel_opensearch.md` |

---

## Java 라이브러리 (Spring Boot 스택)

| 라이브러리 | 버전 | 용도 | 출처 |
|-----------|------|------|------|
| spring-boot-starter-web | Spring Boot 3.x | MVC, RestController | `backend_springboot.md` |
| spring-boot-starter-oauth2-resource-server | Spring Boot 3.x | JWT 검증 (Keycloak) | `auth_keycloak.md` |
| spring-boot-starter-validation | Spring Boot 3.x | Bean Validation (`@Valid`) | `backend_springboot.md` |
| spring-retry | TBD (latest) | `@Retryable` 지수 백오프 | `backend_springboot.md` |
| Resilience4j | TBD (latest) | Circuit Breaker (미확정, 검토 중) | `backend_springboot.md` |
| Lombok | TBD (latest) | `@RequiredArgsConstructor` 등 | `backend_springboot.md` |
| JUnit | 5 | 단위·통합 테스트 | `backend_springboot.md` |
| Mockito | TBD (latest) | Mock 객체 | `backend_springboot.md` |
| Testcontainers | TBD (latest) | PostgreSQL 컨테이너 (`postgres:15`) | `backend_springboot.md` |
| SpringDoc OpenAPI | TBD (latest) | Swagger UI 자동 생성 | `backend_springboot.md` |
| logstash-logback-encoder | TBD (latest) | JSON 구조화 로그 | `backend_springboot.md`, `observability_otel_opensearch.md` |
| opentelemetry-spring-boot-starter | TBD (latest) | OTel 자동 계측 (traces·metrics·logs) | `observability_otel_opensearch.md` |
| LangChain4j | TBD — 미확정 | LLM 오케스트레이션, ReAct 에이전트 | `backend_springboot.md` |

---

## 프론트엔드

| SW | 버전 | 용도 | 출처 |
|----|------|------|------|
| React | 18 | UI 프레임워크 | `frontend.md` |
| TypeScript | TBD (latest) | 타입 안전성 | `frontend.md` |
| Vite | TBD (latest) | 번들러 (신규 프로젝트 권장) | `frontend.md` |
| Next.js | TBD (latest) | SSR/SSG, SEO 필요 시 | `frontend.md` |
| Tailwind CSS | TBD (latest) | 유틸리티 CSS | `frontend.md` |
| shadcn/ui | TBD (latest) | 컴포넌트 라이브러리 | `frontend.md` |
| Zustand | TBD (latest) | 중형 앱 전역 상태 관리 | `frontend.md` |
| Redux Toolkit | TBD (latest) | 대형 앱 전역 상태 관리 | `frontend.md` |
| Webpack | TBD (latest) | 레거시·고도 커스터마이징 번들러 | `frontend.md` |
| Vitest | TBD (latest) | 컴포넌트 단위 테스트 | `frontend.md` |
| Playwright | TBD (latest) | E2E 및 CT(Component Testing) | `frontend.md` |

---

## 인증·인가

| SW | 버전 | 비고 | 출처 |
|----|------|------|------|
| Keycloak | 22+ | Resource Owner Password 기본 비활성 버전 | `auth_keycloak.md` |

---

## 관찰성 (Observability)

| SW | 버전 | 역할 | 출처 |
|----|------|------|------|
| OpenTelemetry Collector | TBD (latest) | 수집·가공·라우팅 허브 | `observability_otel_opensearch.md` |
| OpenSearch | TBD (latest) | 로그 저장·검색 | `observability_otel_opensearch.md` |
| OpenSearch Dashboards | TBD (latest) | 로그 시각화·알림 | `observability_otel_opensearch.md` |
| Jaeger | TBD (latest) | 분산 트레이싱 | `observability_otel_opensearch.md` |
| Tempo | TBD (latest) | 분산 트레이싱 (Jaeger 대안) | `observability_otel_opensearch.md` |
| Prometheus | TBD (latest) | 메트릭 수집·저장 | `observability_otel_opensearch.md` |
| Grafana | TBD (latest) | 메트릭 시각화 | `observability_otel_opensearch.md` |

---

## 버전 확정 우선순위

스펙 구현 시작 전 아래 항목을 먼저 확정한다.

| 우선순위 | SW | 이유 |
|---------|-----|------|
| 1 | LangChain4j | Spring Boot 스택의 핵심 — ReAct 에이전트 호환성 검증 필요 |
| 2 | Keycloak | 운영 환경 HA 구성과 맞물린 인프라 결정 |
| 3 | OpenSearch | 샤드 수·ILM 정책이 버전별 기능 차이 영향 받음 |
| 4 | LangChain (Python) | 마이너 버전별 API 변경 잦음 — 핀닝 필요 |
| 5 | FastAPI | Pydantic v2 호환 버전 범위 확인 |

---

## 변경 이력

| 날짜 | 버전 | 내용 |
|------|------|------|
| 2026-05-03 | v1.0 | 초안 — common/ 전체 스펙에서 SW 버전 일괄 추출 |
