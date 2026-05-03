# Specs / Edit — 편집 관리용 스펙

이 디렉토리는 **편집·관리 목적**의 스펙을 영역별로 분리하여 관리한다.  
실행용 태스크 스펙(`specs/tasks/`)은 여기서 편집된 내용을 합산하여 구성한다.

---

## 디렉토리 구조

```
specs/edit/
├── business/               # 비즈니스 요건 (What & Why)
│   └── requirements.md     # 목표, 사용자 시나리오, 기능·비기능 요건
│
├── backend/                # 백엔드 기술 스펙 (How)
│   ├── fastapi_spec.md     # Flask → FastAPI 전환 설계안 (Python 유지)
│   └── springboot_spec.md  # 미구현 설계안 — Java/Spring Boot + LangChain4j
│
├── frontend/               # 프론트엔드 기술 스펙
│   ├── flask_ui_spec.md    # 현재 구현 — Flask Jinja2 + Vanilla JS
│   └── react_ui_spec.md    # 미구현 설계안 — React SPA 전환 시
│
├── agent/                  # Agentic Agent 스펙
│   └── agentic_agent_spec.md  # ReAct 에이전트 구조, 프롬프트, 툴 구성
│
├── rag/                    # RAG Agent 스펙
│   └── rag_agent_spec.md   # 벡터 스토어 설계, 캐시 전략 (미구현)
│
├── database/               # 테이블 설계 스펙
│   └── table_spec.md       # 테이블 정의, DDL, 인덱스, TTL 설계 (DA 관점)
│
└── common/                 # 기술 공통 레퍼런스 (비즈니스 무관)
    ├── agent.md            # ReAct 패턴, 툴 설계, Observability
    ├── backend_fastapi.md      # FastAPI — Pydantic, Depends, asyncio, httpx
    ├── backend_springboot.md   # Spring Boot — 생성자 주입, Virtual Threads, RestClient
    ├── database.md         # 정규화, 타입 선택, 인덱스 패턴, 마이그레이션
    ├── frontend.md                        # 렌더링 방식, 컴포넌트 설계, 상태 관리, 접근성
    ├── rag.md                             # 청킹, 임베딩, 벡터 스토어, 검색 전략, 평가
    ├── auth_keycloak.md                   # Keycloak 기반 통합 인증·인가 (PKCE, JWT, RBAC)
    ├── observability_otel_opensearch.md   # OTel + OpenSearch 통합 로그·트레이싱·메트릭
    ├── versions.md                        # common/ 전체 스펙에서 사용된 SW 버전 일람
    ├── business_format.md                 # 비즈니스 스펙 공통 템플릿 및 작성 가이드
    │
    │   # --- 1순위: 운영 필수 ---
    ├── infra_cicd.md                      # Docker 멀티스테이지 / K8s / GitHub Actions CI/CD
    ├── security.md                        # OWASP Top 10 / TLS / Secrets / 컨테이너 보안
    ├── testing_strategy.md                # 테스트 피라미드 통합 전략 (단위→통합→E2E→성능)
    │
    │   # --- 2순위: 스케일업 필수 ---
    ├── api_gateway.md                     # Kong / 라우팅 / Rate Limiting / 서킷 브레이커
    ├── caching_strategy.md                # Redis 멀티레이어 캐시 / TTL / Cache Stampede 방지
    ├── messaging.md                       # Kafka / DLQ / Outbox 패턴 / 멱등성
    │
    │   # --- 3순위: 팀·프로세스 성숙도 ---
    ├── adr_format.md                      # Architecture Decision Record 템플릿 및 예시
    ├── data_governance.md                 # 데이터 분류 / PII 마스킹 / 계보 / 마이그레이션
    └── data_pipeline_airflow.md           # 수집→ODS→DW→MART Airflow 파이프라인 기술 기준
```

---

## 각 스펙의 역할

| 파일 | 독자 | 핵심 질문 |
|------|------|-----------|
| `business/requirements.md` | PO, 기획자, 전체 팀 | 왜 만드는가? 무엇을 만드는가? |
| `backend/fastapi_spec.md` | 백엔드 개발자(Python) | Flask에서 FastAPI로 어떻게 전환하는가? |
| `backend/springboot_spec.md` | 백엔드 개발자(Java) | Spring Boot로 재구현 시 구조는 어떻게 되는가? |
| `frontend/flask_ui_spec.md` | 프론트엔드 개발자 | 현재 Flask 템플릿 화면은 어떻게 구성되는가? |
| `frontend/react_ui_spec.md` | 프론트엔드 개발자 | React 전환 시 컴포넌트 구조는 어떻게 되는가? |
| `agent/agentic_agent_spec.md` | AI/백엔드 개발자 | 에이전트는 어떻게 추론하는가? |
| `rag/rag_agent_spec.md` | AI 개발자 | 검색·캐싱을 어떻게 구현하는가? |
| `database/table_spec.md` | 백엔드·DBA | 어떤 테이블에 무엇을 저장하는가? |
| `common/agent.md` | AI 개발자 | 에이전트 패턴·툴 설계의 기술 기준은? |
| `common/backend_fastapi.md` | 백엔드 개발자(Python) | FastAPI 구조·비동기·테스트 기술 기준은? |
| `common/backend_springboot.md` | 백엔드 개발자(Java) | Spring Boot 계층·DI·동시성 기술 기준은? |
| `common/database.md` | 백엔드·DBA | 스키마·인덱스·마이그레이션의 기술 기준은? |
| `common/frontend.md` | 프론트엔드 개발자 | 컴포넌트·상태·빌드의 기술 기준은? |
| `common/rag.md` | AI 개발자 | 청킹·임베딩·검색·평가의 기술 기준은? |
| `common/auth_keycloak.md` | 백엔드·보안 | Keycloak 인증 흐름·토큰 검증·역할 설계는? |
| `common/observability_otel_opensearch.md` | DevOps·백엔드 | OTel 계측·로그 구조화·OpenSearch 인덱스 설계는? |
| `common/versions.md` | 전체 팀 | 각 스펙에서 사용된 SW 버전은 무엇인가? |
| `common/business_format.md` | PO·기획자·개발팀 | 비즈니스 스펙을 어떤 형식으로 써야 하는가? |
| `common/infra_cicd.md` | DevOps·백엔드 | Docker 이미지·K8s 배포·CI/CD 파이프라인 기준은? |
| `common/security.md` | 백엔드·DevOps·보안 | OWASP 대응·TLS·Secrets·컨테이너 보안 기준은? |
| `common/testing_strategy.md` | 전체 개발팀 | 어떤 테스트를 어느 레이어에서 어떻게 작성하는가? |
| `common/api_gateway.md` | 백엔드·DevOps | 라우팅·Rate Limiting·인증 통합 기준은? |
| `common/caching_strategy.md` | 백엔드 | Redis 캐시 패턴·TTL·무효화 전략은? |
| `common/messaging.md` | 백엔드 | Kafka 이벤트 설계·DLQ·Outbox 패턴은? |
| `common/adr_format.md` | 전체 팀 | 아키텍처 결정을 어떻게 기록하는가? |
| `common/data_governance.md` | 백엔드·DBA·보안 | 데이터 분류·PII 처리·보존 기간 기준은? |
| `common/data_pipeline_airflow.md` | 데이터 엔지니어 | 수집→ODS→DW→MART 파이프라인을 어떻게 구성하는가? |

---

## 편집 원칙

- **업무 요건(business)** 은 구현 방법을 기술하지 않는다
- **기술 스펙(나머지)** 은 "왜"보다 "어떻게"에 집중한다
- 변경 시 반드시 파일 하단 **변경 이력** 테이블을 업데이트한다
- 미결 과제는 `- [ ]` 체크박스로 관리한다
