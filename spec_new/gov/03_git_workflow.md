# GOV-03 — Git 워크플로 규칙

> **강제 대상**: 모든 코드 변경  
> **게이트**: commitlint(커밋), 브랜치 보호 규칙(머지)  
> **원본 참조**: `enterprise/02.02_gov_branch_strategy.md`, `enterprise/02.03_gov_version_release_management.md`

---

## 브랜치 전략

```
main          ← 프로덕션. 직접 push 금지. PR + 2인 승인만
  │
  ├─ develop  ← 통합 브랜치 (팀 규모 작을 시 생략 가능)
  │
  ├─ feature/{ticket-id}-{short-desc}    예: feature/ICE-42-add-rag-cache
  ├─ fix/{ticket-id}-{short-desc}        예: fix/ICE-99-null-linkedin-url
  ├─ chore/{desc}                        예: chore/upgrade-langchain-0.3
  └─ hotfix/{desc}                       예: hotfix/token-expiry-bug
```

| 브랜치 | 생성 기준 | 머지 대상 | 조건 |
|--------|---------|---------|------|
| `feature/*` | 기능 단위 | develop / main | PR + CI 통과 + 1인 승인 |
| `fix/*` | 버그 수정 | develop / main | PR + CI 통과 |
| `hotfix/*` | 프로덕션 긴급 | main + develop | PR + CI 통과 + 2인 승인 |

---

## 커밋 메시지 규칙 (Conventional Commits)

```
{type}({scope}): {subject}

type:
  feat     새 기능
  fix      버그 수정
  test     테스트 추가·수정
  refactor 동작 변경 없는 코드 개선
  docs     문서만 변경
  chore    빌드·설정 변경
  perf     성능 개선
  ci       CI/CD 변경

예:
  feat(agent): add linkedin lookup retry with backoff
  fix(api): handle null twitter username in ice_break_with
  test(bdd): add scenario for non-trading day 404
  docs(adr): add ADR-0003 keycloak decision
```

**금지**: `fix: 수정`, `update stuff`, `WIP` — PR 머지 전 정리 필수

---

## PR 규칙

- 제목: Conventional Commits 형식
- 본문: What(무엇을) + Why(왜) — How(어떻게)는 코드가 설명
- PR 크기: 300줄 이하 권장 (초과 시 분리 검토)
- 리뷰어: 최소 1인 (main 머지는 2인)
- BDD feature 변경 시 `.feature` 파일과 step 동시 포함

---

## 버전 관리 (Semantic Versioning)

```
MAJOR.MINOR.PATCH  예: 2.1.3

MAJOR → 하위 호환 불가 변경 (API Breaking Change)
MINOR → 하위 호환 가능 기능 추가
PATCH → 버그 수정
```

릴리스는 `git tag v{version}` + GitHub Release 자동 생성 (semantic-release).

---

## 브랜치 보호 규칙 (GitHub Ruleset)

`main` 브랜치 필수 설정:
- `Require pull request` — 직접 push 차단
- `Require status checks` — CI 전체 통과 필수
- `Require conversation resolution` — 모든 리뷰 코멘트 해결
- `Restrict deletions` — 브랜치 삭제 금지
