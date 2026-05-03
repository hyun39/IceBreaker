# spec_new — BDD/SDD 통합 스펙 체계

> 레포 스펙을 **BDD/SDD 워크플로에 맞게 정제·재구성**한 실행 기준 디렉터리.  
> 구현 표준(`std/`)의 상세 원문은 **`std/detail/`** 에 두어 `specs/edit/` 를 열지 않고도 같은 레벨에서 읽을 수 있다.

---

## 디렉토리 구조와 역할

```
spec_new/
├── gov/          GOV — 강제 규칙·게이트  (무엇을 반드시 해야 하는가)
├── bdd/          BDD — 행동 명세 가이드  (어떻게 스펙을 실행 가능하게 만드는가)
└── std/          STD — 구현 표준        (어떻게 코드를 작성하는가)
    └── detail/   STD 상세 원문 (요약본 `0N_*.md` 와 짝)
```

| 섹션 | 독자 | 핵심 질문 | 위반 시 |
|------|------|----------|--------|
| `gov/` | 전체 팀 | "이것을 안 하면 안 되는가?" | CI 차단·리뷰 거부 |
| `bdd/` | PO·개발자 | "어떻게 스펙을 테스트로 만드는가?" | 누락 시 스펙 미완성 |
| `std/` | 개발자 | "어떻게 구현하는가?" | 코드 리뷰 피드백 |

---

## BDD/SDD 전체 흐름

```
┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 1 — 요건 발견 (GOV 주도)                                       │
│                                                                       │
│  [비즈니스 스펙] ──► [인수기준(Gherkin)] ──► [ADR] ──► [API 계약]     │
│   gov/01 형식           bdd/01 가이드       gov/02     gov/06         │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ 스펙 확정
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 2 — 스펙 실행화 (BDD 주도)                                     │
│                                                                       │
│  [.feature 파일 작성] ──► [Step 구현 (Red)] ──► [코드 작성 (Green)]   │
│   bdd/templates/          bdd/02 FastAPI          std/ 참조           │
│                           bdd/03 Spring                               │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ BDD 통과
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 3 — 품질 게이트 (GOV 강제)                                     │
│                                                                       │
│  Unit → Integration → Contract → E2E → Security → Performance       │
│                        gov/05 게이트 기준                              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## GOV 파일 목록

> 규칙·정책·강제 사항. 준수 여부는 CI 또는 리뷰에서 검증.

| 파일 | 강제 내용 | CI 게이트 |
|------|---------|---------|
| `gov/01_requirements.md` | 비즈니스 스펙 필수 항목·DoR | PR 템플릿 체크리스트 |
| `gov/02_adr.md` | 아키텍처 결정 기록 규칙 | ADR 없는 주요 변경 리뷰 거부 |
| `gov/03_git_workflow.md` | 브랜치·커밋·릴리스 규칙 | commitlint·브랜치 보호 |
| `gov/04_security.md` | OWASP·TLS·Secrets 필수 규칙 | SAST HIGH 이상 차단 |
| `gov/05_quality_gates.md` | 테스트 커버리지·BDD 필수 | Coverage 80%↑ 차단 |
| `gov/06_api_design.md` | API 설계·버전·에러 규칙 | Spectral 린트 차단 |
| `gov/07_data_policy.md` | 데이터 분류·PII·보존 규칙 | 마이그레이션 리뷰 |

---

## BDD 파일 목록

> Gherkin 작성부터 Step 구현까지 실천 가이드.

| 파일 | 내용 |
|------|------|
| `bdd/01_writing_guide.md` | Gherkin 원칙·좋은/나쁜 예시·시나리오 분류 |
| `bdd/02_fastapi_impl.md` | pytest-bdd step 구현·conftest·CI 연동 |
| `bdd/03_spring_impl.md` | Cucumber/Spring step 구현·설정·CI 연동 |
| `bdd/04_agent_impl.md` | LLM·ReAct·구조화 출력 BDD, Mock 전략, Pydantic 검증 |
| `bdd/templates/feature.feature` | 바로 복사해서 쓰는 .feature 템플릿 |
| `bdd/templates/business_spec.md` | Gherkin 연결 포함 비즈니스 스펙 템플릿 |

---

## STD 파일 목록

> 구현 방법. 요약은 `std/0N_*.md`, 전체 분량은 `std/detail/` 동일 파일명.

| 파일 | 핵심 내용 | 원본 참조 |
|------|---------|---------|
| `std/01_backend.md` | FastAPI·SpringBoot 계층·에러·비동기 | `std/detail/backend_*.md` |
| `std/02_frontend.md` | React 상태·API 클라이언트·에러 처리 | `std/detail/frontend.md` |
| `std/03_database.md` | 정규화·인덱스·마이그레이션·TTL | `std/detail/database.md` |
| `std/04_auth.md` | Keycloak PKCE·JWT 검증·역할 설계 | `std/detail/auth_keycloak.md` |
| `std/05_infra.md` | Docker·K8s·CI/CD·GitOps | `std/detail/infra_cicd.md` |
| `std/06_observability.md` | OTel·OpenSearch·구조화 로그 | `std/detail/observability_otel_opensearch.md` |
| `std/07_ai.md` | Agent·RAG·LLMOps 핵심 패턴 | `std/detail/agent.md`, `rag.md` |
| `std/08_data_pipeline.md` | Airflow·ODS→DW→MART 패턴 | `std/detail/data_pipeline_airflow.md` |

---

## 실행 가이드

> 환경 설정부터 첫 BDD Green까지 단계별 안내: **[EXECUTION_GUIDE.md](./EXECUTION_GUIDE.md)**

---

## 빠른 시작 (Quick Start)

### 새 기능 시작 시

```bash
# 1. 비즈니스 스펙 작성 (gov/01 규칙 준수)
cp bdd/templates/business_spec.md specs/{feature_name}/business_spec.md

# 2. .feature 파일 작성 (bdd/01 가이드 참조)
cp bdd/templates/feature.feature tests/bdd/features/{feature_name}.feature

# 3. ADR 등록 (기술 결정이 있을 때, gov/02 참조)
cp docs/adr/ADR-0000-template.md docs/adr/ADR-{NNNN}-{title}.md

# 4. Step 구현 (bdd/02 or bdd/03, 에이전트는 bdd/04 참조)
touch tests/bdd/steps/{feature_name}_steps.py

# 5. 구현 (std/ 참조)
# → 테스트 Red → 코드 작성 → 테스트 Green
```

### 파일 선택 기준

```
"이것이 규칙인가?"          → gov/ 확인
"Gherkin을 어떻게 쓰는가?"  → bdd/01 확인
"Step을 어떻게 구현하는가?" → bdd/02 or bdd/03 확인 (에이전트는 bdd/04)
"코드를 어떻게 짜는가?"     → std/ 확인
```

---

## `specs/edit/` 와의 관계

| | `specs/edit/` | `spec_new/` |
|---|---|---|
| 목적 | 조직·레포 전체 레퍼런스 (common, enterprise 등) | BDD/SDD 워크플로 실행 기준 |
| 구성 | 기술 도메인별·엔터프라이즈 확장 | 역할별 (gov, bdd, std + `std/detail`) |
| 분량 | 상세 | 요약(`std/0N`) + 상세 사본(`std/detail`) |
| 사용 시점 | 조직 표준 전체를 볼 때 | 기능 개발 시 `spec_new` 만 열어도 됨 |

> `std/detail/*.md` 는 `specs/edit/common/` 동일 파일의 사본이다. 내용을 맞추려면 원본 변경 후 `detail/` 에 다시 복사하면 된다.
