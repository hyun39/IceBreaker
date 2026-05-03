# SDD 성숙도 평가 — specs/edit/enterprise 기준

> 작성일: 2026-05-03  
> 대상: `/specs/edit/enterprise/` (22개 파일, 8,114줄)  
> 평가 기준: 2026 Spec-Driven Development 핵심 트렌드 6가지

---

## 질문

> **specs/edit/enterprise SDD trend에 맞춰볼때 어느정도의 수준일까?**

---

## 평가 기준 — 2026 SDD 핵심 트렌드 6가지

| # | 트렌드 | 정의 |
|---|--------|------|
| T1 | **Multi-modal Spec** | 텍스트 + 시각(목업) + 실행 가능(테스트) 스펙의 통합 |
| T2 | **AI-Assisted Spec** | LLM이 스펙 생성·검증·일관성 체크를 보조 |
| T3 | **Spec as Code (SaC)** | 스펙을 Git에서 버전 관리·리뷰·CI 검증 |
| T4 | **Living Specification** | 코드·배포·지표와 스펙이 자동 동기화 |
| T5 | **Executable Specification** | 스펙이 직접 테스트로 실행 (BDD/Gherkin runner) |
| T6 | **Continuous Discovery** | 프로덕션 지표 → 가설 검증 → 스펙 진화 루프 |

---

## 영역별 점수

```
T1  Multi-modal Spec          ████████████░░  85%  ★★★★☆
T2  AI-Assisted Spec          ████████████░░  80%  ★★★★☆
T3  Spec as Code              ████████░░░░░░  60%  ★★★☆☆
T4  Living Specification      ████░░░░░░░░░░  30%  ★★☆☆☆
T5  Executable Specification  ████░░░░░░░░░░  30%  ★★☆☆☆
T6  Continuous Discovery      ████████░░░░░░  55%  ★★★☆☆

Overall                       ██████████░░░░  57%  Level 3 / 5
```

---

## 상세 분석

### ✅ 잘 갖춰진 영역

**T1 Multi-modal (85%)** — 가장 강한 영역

- `01.02` L0~L3 목업 진화 사이클 (텍스트 + React 코드 + Storybook 시각)
- `03.01.xx` Design Token + Storybook + Chromatic (디자인 ↔ 코드 연결)
- `06.01` Pact Contract Test (서비스 간 계약이 코드로 실행)
- Gherkin 인수기준이 `01.01`에 명시됨

**T2 AI-Assisted (80%)**

- `01.02` `parse_spec.py` — 비즈니스 스펙 → 화면 JSON 자동 추출
- `01.02` `gen_l1_skeleton.py` → `gen_l2_hifi.py` — React 목업 자동 생성
- `01.02` MSW 핸들러 자동 생성 (api_contract.md 기반)
- **부족**: 스펙 자체의 완전성·일관성을 LLM이 검증하는 단계 없음

---

### ⚠️ 부분적으로 갖춰진 영역

**T3 Spec as Code (60%)**

| 있는 것 | 없는 것 |
|--------|--------|
| Git에 저장 (`.md` 파일) | 스펙 변경 PR 리뷰 프로세스 표준 |
| ADR 형식 정의 (`02.01`) | 스펙 드리프트 감지 (스펙과 코드 불일치 탐지) |
| 브랜치·버전 정책 (`02.02`, `02.03`) | 스펙 CI (스펙 자체를 lint하는 파이프라인) |

**T6 Continuous Discovery (55%)**

| 있는 것 | 없는 것 |
|--------|--------|
| Build-Measure-Learn 루프 (`01.01`) | 프로덕션 지표 → 스펙 자동 피드백 |
| SLO/Error Budget (`07.01`) | SLO 미달 → 요건 재검토 트리거 |
| UX KPI 정의 (`03.01`) | KPI 달성 여부가 스펙에 자동 반영되는 구조 없음 |

---

### ❌ 취약한 영역

**T4 Living Specification (30%)**

현재 스펙은 **정적 문서**다. 코드가 바뀌어도 스펙이 자동으로 갱신되지 않는다.

| 없는 것 | 의미 |
|--------|------|
| 코드 → 스펙 역동기화 | API 구현이 바뀌어도 OpenAPI 스펙은 수동 갱신 |
| 스펙-코드 드리프트 탐지 | "스펙에는 있는데 코드에는 없는" 상태 감지 불가 |
| 스펙 신선도(Freshness) 지표 | 마지막 갱신일 이후 코드 커밋 수 등 |

**T5 Executable Specification (30%)**

Gherkin이 `01.01`에 작성 기준으로 나오지만, 실제로 **실행되지 않는다.**

| 없는 것 | 의미 |
|--------|------|
| BDD 프레임워크 연동 | Cucumber(Java) / Behave(Python) 미통합 |
| Gherkin → 자동 테스트 | 인수기준이 CI에서 실행되지 않음 |
| 스펙 Coverage | "어느 스펙이 테스트로 커버되는가" 추적 불가 |

---

## 종합 판정: Level 3 / 5 (Advancing)

```
Level 1  Aware       ░░░░░  문서화 시작
Level 2  Structured  █████  체계적 문서화, 표준 정의
Level 3  Advancing   █████  ← 현재 위치
                             AI 보조, Multi-modal, 거버넌스 갖춤
                             코드-스펙 연결은 단방향(스펙→코드)
Level 4  Integrated  ░░░░░  Living Spec, Executable Spec, 양방향 연결
Level 5  Autonomous  ░░░░░  AI가 스펙 자체를 지속적으로 진화
```

---

## Level 4로 가기 위한 다음 단계

| 우선순위 | 항목 | 난이도 | 효과 |
|---------|------|--------|------|
| **P0** | Gherkin → Behave/Cucumber CI 연동 | 중 | T5 30%→80% |
| **P0** | OpenAPI 코드 우선 생성 (`fastapi-codegen` 등) | 중 | T4 일부 해소 |
| **P1** | 스펙 드리프트 CI 체크 (OpenAPI diff, Pact 강제) | 중 | T4 30%→60% |
| **P1** | 스펙 완전성 LLM 검증 (PR 시 "요건에 에러 상태 없음" 탐지) | 낮 | T2 80%→95% |
| **P2** | SLO 미달 → Jira 이슈 자동 생성 (지표→스펙 피드백) | 높 | T6 55%→75% |
| **P2** | 스펙 카탈로그 / Backstage 통합 | 높 | T3 60%→80% |

---

## 글로벌 벤치마크 대비

| 조직 유형 | 일반적 수준 | 현재 enterprise 스펙 |
|----------|-----------|---------------------|
| 스타트업 (< 50명) | Level 1~2 | **Level 3** (앞서 있음) |
| 중견 테크 기업 | Level 2~3 | **Level 3** (동급) |
| 빅테크 (Google, Netflix) | Level 4~5 | **Level 3** (한 단계 뒤) |
| 평균 엔터프라이즈 | Level 1~2 | **Level 3** (크게 앞서 있음) |

---

## 한 줄 요약

> 현재 수준은 **중견 테크 기업 동급(Level 3)** 으로 평균 엔터프라이즈보다 크게 앞서 있다.  
> Living Spec(T4)과 Executable Spec(T5)이 Level 4의 핵심 관문이며,  
> **Gherkin CI 연동**이 가장 투자 대비 효과가 큰 다음 단계다.
