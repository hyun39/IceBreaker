# Enterprise Common Spec — 개요 및 아키텍처 거버넌스

> 이 디렉터리는 **2026년 엔터프라이즈 기술 트렌드**를 반영해  
> 조직 전체에 적용할 수 있는 표준 기술 스펙 모음이다.  
> 서비스 단위 구현 기준은 개별 서비스 스펙에서 정의하며, 이 디렉터리는 **조직·플랫폼 수준 아키텍처 표준**을 담는다.

---

## 목적

| 목표 | 내용 |
|------|------|
| 기술 일관성 | 전사 서비스가 동일한 패턴·도구를 사용해 운영·보안 부담을 줄인다 |
| 개발자 경험(DX) | Golden Path 제공으로 팀이 "잘 닦인 길"을 빠르게 따를 수 있게 한다 |
| 규정 준수 | 보안·데이터·비용 정책을 코드로 강제해 감사 부담을 줄인다 |
| 혁신 가속 | 플랫폼 팀이 공통 기반을 제공하고 제품 팀은 비즈니스에 집중한다 |

---

## 2026 Enterprise 기술 지형

```
┌─────────────────────────────────────────────────────────────────────┐
│  PLATFORM LAYER (Internal Developer Platform)                        │
│  ├─ Golden Path Template (Backstage / 자체 포털)                     │
│  ├─ GitOps (ArgoCD) — 선언적 배포 표준                               │
│  ├─ Service Mesh (Istio) — mTLS, 트래픽 제어                         │
│  └─ FinOps Dashboard — 팀별 클라우드 비용 가시화                      │
├─────────────────────────────────────────────────────────────────────┤
│  SECURITY LAYER (Zero Trust)                                         │
│  ├─ Zero Trust Network — 암묵적 신뢰 없음, 지속 검증                  │
│  ├─ Supply Chain Security — SLSA, SBOM, Sigstore Cosign              │
│  ├─ Compliance as Code — OPA/Kyverno Policy                          │
│  └─ Secrets — Vault + 동적 시크릿, 최소 수명                         │
├─────────────────────────────────────────────────────────────────────┤
│  AI/ML LAYER (LLMOps)                                                │
│  ├─ Prompt Registry — 버전·실험·롤백                                  │
│  ├─ LLM Gateway — 멀티모델, Rate limit, 비용 분담                     │
│  ├─ Evaluation Pipeline — 품질 지표 자동 측정                         │
│  └─ RAG Knowledge Base — 엔터프라이즈 공용 벡터 스토어                │
├─────────────────────────────────────────────────────────────────────┤
│  DATA LAYER (Data Mesh)                                              │
│  ├─ Domain Data Product — 도메인 팀의 데이터 소유권                   │
│  ├─ Data Contract — 생산자·소비자 계약 기반 통합                      │
│  └─ Data Catalog — 전사 데이터 계보·품질·접근 관리                    │
├─────────────────────────────────────────────────────────────────────┤
│  RELIABILITY LAYER (SRE)                                             │
│  ├─ SLO/SLI/Error Budget — 신뢰성 정량화                             │
│  ├─ Chaos Engineering — 계획된 장애 주입                              │
│  └─ Toil Automation — 반복 수작업의 지속적 제거                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 산출물 기반 스펙 구조 (Spec-Driven Development)

> 파일명 규칙: `NN.MM_gov_XXX.md` = 정책·규칙 / `NN.MM_std_XXX.md` = 구현 표준  
> **NN** = Phase 번호 / **MM** = Phase 내 순서 / 각 Phase는 독립된 **산출물 게이트**를 갖는다

---

### Phase 01 — 요건 발견·검증

> **진입 조건**: 새 프로젝트·기능 시작  
> **산출물 게이트**: 승인된 비즈니스 스펙 ✔ · DoR 체크 통과 ✔ · 이해관계자 서명 ✔  
> *요구사항이 확정되어야 어떤 기술 결정이 필요한지(ADR), 어떤 협업 방식이 맞는지(브랜치·버전)가 명확해진다.*

| # | 유형 | 파일 | 핵심 산출물 |
|---|------|------|-----------|
| 01.01 | **GOV** | `01.01_gov_business_requirements.md` | 비즈니스 스펙 문서 · 유저스토리 · Gherkin 인수기준 |
| 01.02 | **STD** | `01.02_std_prototype_driven_requirements.md` | L1~L3 React 목업 · Storybook PR Preview · 요건 변경 이력 |
| 01.03 | **STD** | `01.03_std_executable_specification.md` | Gherkin → Behave/Cucumber CI 실행 · Spec Coverage 측정 **(Level 4)** |

---

### Phase 02 — 거버넌스 기반 수립

> **진입 조건**: Phase 01 요건 초안 확보 후 (병행 가능)  
> **산출물 게이트**: ADR 레지스트리 생성 ✔ · 브랜치 보호 규칙 적용 ✔ · 버전 정책 합의 ✔  
> *요건을 알아야 어떤 기술 결정을 ADR로 남길지, 어떤 릴리스 주기가 맞는지 설정할 수 있다.*

| # | 유형 | 파일 | 핵심 산출물 |
|---|------|------|-----------|
| 02.01 | **GOV** | `02.01_gov_adr_format.md` | `docs/adr/README.md` + 첫 기술 결정 ADR |
| 02.02 | **GOV** | `02.02_gov_branch_strategy.md` | 브랜치 보호 Ruleset · CODEOWNERS · commitlint 설정 |
| 02.03 | **GOV** | `02.03_gov_version_release_management.md` | `.releaserc.json` · SemVer 정책 문서 |
| 02.04 | **GOV** | `02.04_gov_repository_structure.md` | 레포 전략 · 디렉토리 구조 표준 · 레포 간 의존성 |

---

### Phase 03 — 설계

> **진입 조건**: Phase 02 게이트 통과  
> **산출물 게이트**: Design Token PR 머지 ✔ · 컴포넌트 Storybook 등록 ✔ · 접근성 axe 통과 ✔

| # | 유형 | 파일 | 핵심 산출물 |
|---|------|------|-----------|
| 03.01 | **STD** | `03.01_std_ux_design.md` | 워크플로우 선택 가이드 · Design Token · 접근성 · 컴포넌트 상태 · UX 테스트 (공통) |
| 03.01.01 | **STD** | `03.01.01_std_ux_design_react.md` | React-only 코드 우선 워크플로우 · Storybook 설계 · Chromatic 승인 |
| 03.01.02 | **STD** | `03.01.02_std_ux_design_figma.md` | Figma 디자이너 주도 워크플로우 · Variables→토큰 자동화 · Dev Mode 핸드오프 |

---

### Phase 04 — 플랫폼 기반 구축

> **진입 조건**: Phase 01 게이트 통과 (Phase 02·03 병행 가능)  
> **산출물 게이트**: Golden Path 템플릿 배포 ✔ · Kyverno 정책 적용 ✔ · ArgoCD dev 클러스터 동작 ✔

| # | 유형 | 파일 | 핵심 산출물 |
|---|------|------|-----------|
| 04.01 | **STD** | `04.01_std_platform_engineering.md` | Backstage 템플릿 · Tech Radar · 플랫폼 SLO 정의서 |
| 04.02 | **GOV** | `04.02_gov_zero_trust_security.md` | Zero Trust 정책 문서 · Kyverno 정책 YAML · SLSA 레벨 확인서 |
| 04.03 | **STD** | `04.03_std_gitops.md` | ArgoCD Application 매니페스트 · Helm Chart · 배포 승인 절차서 |
| 04.04 | **STD** | `04.04_std_spec_code_bootstrap.md` | Greenfield/Brownfield 초기 동기화 · Baseline 확립 · BDD 점진적 도입 |

---

### Phase 05 — 개발 표준 적용

> **진입 조건**: Phase 04 게이트 통과  
> **산출물 게이트**: 내부 SDK 첫 버전 배포 ✔ · 첫 서비스 Golden Path로 생성 ✔

| # | 유형 | 파일 | 핵심 산출물 |
|---|------|------|-----------|
| 05.01 | **STD** | `05.01_std_application_standards.md` | `@corp/platform-sdk` 패키지 · 언어별 템플릿 · 빌드 설정 |
| 05.02 | **STD** | `05.02_std_data_platform_standards.md` | Data Contract YAML · `corp-airflow-sdk` · dbt 프로젝트 구조 |
| 05.03 | **STD** | `05.03_std_caching_strategy.md` | Redis Cluster 설정 · 캐시 키 네이밍 정책 · TTL 테이블 |
| 05.04 | **STD** | `05.04_std_event_driven_architecture.md` | Kafka 토픽 설계서 · Schema Registry 설정 · DLQ 정책 |
| 05.05 | **STD** | `05.05_std_ai_application_standards.md` | 멀티에이전트 패턴 · HiTL 워크플로 · AI 감사 로그 스키마 |
| 05.06 | **STD** | `05.06_std_llmops.md` | LLM Gateway 설정 · Prompt Registry · Eval 파이프라인 설정 |

---

### Phase 06 — 품질 보증

> **진입 조건**: Phase 05 첫 서비스 배포  
> **산출물 게이트**: CI 품질 게이트 적용 ✔ · API 카탈로그 등록 ✔ · 계약 테스트 Pact Broker 동작 ✔

| # | 유형 | 파일 | 핵심 산출물 |
|---|------|------|-----------|
| 06.01 | **GOV** | `06.01_gov_testing_strategy.md` | 커버리지 게이트 설정 · Pact Broker · Chromatic 시각적 회귀 |
| 06.02 | **GOV** | `06.02_gov_api_governance.md` | OpenAPI 스펙 · Spectral 규칙 · API Registry 등록 |

---

### Phase 07 — 운영

> **진입 조건**: Phase 06 게이트 통과, 프로덕션 배포  
> **산출물 게이트**: SLO 정의서 승인 ✔ · 알림 규칙 동작 ✔ · Runbook 작성 ✔

| # | 유형 | 파일 | 핵심 산출물 |
|---|------|------|-----------|
| 07.01 | **GOV** | `07.01_gov_sre_practices.md` | SLO YAML · Error Budget 정책 · Incident Runbook · Chaos 실험 계획 |
| 07.02 | **STD** | `07.02_std_living_specification.md` | Code→Spec 역동기화 · Drift 감지 · SLO→스펙 피드백 루프 **(Level 4)** |

---

### Phase 08 — 지속 최적화

> **진입 조건**: Phase 07 운영 안정화 (최소 1개월)  
> **산출물 게이트**: Showback 리포트 첫 발행 ✔ · 태그 적용률 100% ✔

| # | 유형 | 파일 | 핵심 산출물 |
|---|------|------|-----------|
| 08.01 | **GOV** | `08.01_gov_finops.md` | 비용 태그 정책 · 예산 할당표 · Showback 대시보드 · LLM 비용 리포트 |

---

### Phase 09 — 자율 스펙 진화 ⚡ Level 5

> **진입 조건**: Phase 01.03 + 07.02 (Level 4) 안정화 후  
> **산출물 게이트**: AI Validator PR 자동화 ✔ · 주간 Evolution Report 발행 ✔ · 인간 승인 게이트 유지 ✔

| # | 유형 | 파일 | 핵심 산출물 |
|---|------|------|-----------|
| 09.01 | **STD** | `09.01_std_autonomous_spec.md` | AI Validator · AI Generator · AI Evolutor 자율 루프 |

---

## 적용 원칙 (PRISM)

| 원칙 | 설명 |
|------|------|
| **P**latform-first | 공통 기반을 플랫폼으로 제공, 제품 팀은 이를 소비 |
| **R**eliability | 신뢰성을 SLO로 정의하고 Error Budget으로 혁신과 안정의 균형 유지 |
| **I**mmutability | 인프라와 배포 아티팩트는 불변. 변경은 코드로만 |
| **S**ecurity-left | 보안 검증은 파이프라인 최초 단계에서 수행 |
| **M**easure-everything | 비용·품질·신뢰성을 수치로 관리. 측정 불가능한 요건은 요건이 아님 |

---

## SDD 진행 흐름 (Spec-Driven Development)

```
Phase 01 (거버넌스 기반)
  └─ 산출물 게이트 통과
        ├─► Phase 02 (요건 발견·검증)   ─► Phase 03 (설계)
        │         └─ 게이트 통과 ──────────────────────────┐
        └─► Phase 04 (플랫폼 기반)                         │
                  └─ 게이트 통과 ──────────────────────────┤
                                                           ▼
                                              Phase 05 (개발 표준 적용)
                                                    └─ 게이트 통과
                                                           ▼
                                              Phase 06 (품질 보증)
                                                    └─ 게이트 통과
                                                           ▼
                                              Phase 07 (운영)
                                                    └─ 안정화
                                                           ▼
                                              Phase 08 (지속 최적화)
```

| Phase | 기간 | 핵심 게이트 |
|-------|------|-----------|
| 01 요건 발견 | 1~4주 | 이해관계자 서명 · DoR 통과 |
| 02 거버넌스 기반 | Day 0~1주 (01과 병행) | ADR 레지스트리 · 브랜치 보호 · 버전 정책 |
| 03 설계 | 1~2주 (02와 병행) | Design Token PR · 접근성 통과 |
| 04 플랫폼 구축 | 1~3개월 (01 후 병행) | Golden Path · ArgoCD dev 동작 |
| 05 개발 표준 | 2~5개월 | SDK 배포 · 첫 서비스 템플릿 생성 |
| 06 품질 보증 | 개발과 동시 | CI 게이트 · API 카탈로그 |
| 07 운영 | 프로덕션 배포 후 | SLO 정의 · Runbook |
| 08 최적화 | 1개월 운영 후 | Showback 리포트 발행 |

---

## 거버넌스 기구

| 역할 | 책임 |
|------|------|
| Architecture Review Board (ARB) | 새 기술 도입 ADR 검토·승인 |
| Platform Engineering Team | Golden Path 제공·유지 |
| SRE Team | SLO 정의·Error Budget 관리·Incident 대응 |
| Security Champions | 팀별 보안 정책 적용·준수 |
| FinOps Practitioner | 비용 최적화·Showback 운영 |

---

## 주요 참조 스펙

- `01.01_gov_adr_format.md` — 모든 아키텍처 결정 기록 표준 (ARB 연동)
- `01.02_gov_branch_strategy.md` + `01.03_gov_version_release_management.md` — Git·릴리스 정책
- `05.02_std_data_platform_standards.md` — 데이터 분류·보존·거버넌스 기준

## ADR (Architecture Decision Record)

모든 아키텍처 결정은 ADR로 기록한다.  
템플릿·상태 정의·작성 가이드·ARB 연동 절차는 **`01.01_gov_adr_format.md`** 참조.
