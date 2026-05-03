# Common Spec — 브랜치 전략 (Branch Strategy)

> `infra_cicd.md`의 CI 트리거·CD 흐름, `testing_strategy.md`의 PR/머지 시점별 테스트와 정합을 맞춘다.

---

## 목적

- **통합·릴리스·핫픽스** 흐름을 저장소 브랜치로 명시해, 누가 어느 브랜치에 머지하는지·어떤 가드가 걸리는지 혼선을 없앤다.
- `main` / `develop`을 기준으로 **자동 배포·테스트 게이트**(`CI 통합 매트릭스`)를 일관되게 연결한다.

---

## 영구 브랜치

| 브랜치 | 역할 | 직접 push | 기본 머지 대상 |
|--------|------|-----------|----------------|
| `main` | 프로덕션 후보·태그 릴리스 소스 | 불가 (PR만) | — |
| `develop` | 다음 릴리스 통합선 | 불가 (PR만) | 기능·버그 PR |

- **`main`**: 스테이징·프로덕션 파이프라인 트리거의 소스. 릴리스 태그(`v*`)는 `main` HEAD 기준으로 생성한다.
- **`develop`**: 일상 통합. CI에서 `lint → test → build`가 돌아가며, 정책에 따라 이미지 푸시는 `main`만 허용할 수 있다 (`infra_cicd.md` CI 예시와 동일하게 조정).

---

## 단기 브랜치

| 패턴 | 출발점 | 목적 | 생명주기 |
|------|--------|------|----------|
| `feature/<이슈>-<요약>` | `develop` | 신규 기능·개선 | 머지 후 삭제 |
| `fix/<이슈>-<요약>` | `develop` | 버그 수정 | 머지 후 삭제 |
| `release/<버전>` | `develop` | 릴리스 준비(버전 고정·CHANGELOG) | `main` + `develop`에 머지 후 삭제 |
| `hotfix/<이슈>-<요약>` | `main` | 프로덕션 긴급 수정 | `main` + `develop`(또는 `release`)에 백포트 |

**네이밍**: 소문자·하이픈. 요약은 영문 slug 권장 (도구·URL 호환).

---

## 흐름 (개요)

```
feature/fix ──PR──► develop ──PR──► main ──tag──► v1.x.x
                         │                │
                         └── release/* ───┘ (선택: 릴리스 브랜치 분리 시)

hotfix/* ──PR──► main ──동일 패치──► develop
```

---

## Pull Request 규칙

| 항목 | 권장 |
|------|------|
| 최소 리뷰어 | 1인 이상 (리포지토리 정책에 따라 2인) |
| CI 필수 | PR 대상 브랜치 기준 `lint`, **Unit** 통과 (`testing_strategy.md`) |
| 머지 방식 | Squash merge 권장 — `main`/`develop` 히스토리 단순화 |
| Draft | WIP는 Draft PR로 표시 |

**통합 테스트**: PR **머지 전** 통과 필수로 두는 경우, 대상 브랜치가 `develop` 또는 `main`일 때 Integration job을 필수 체크로 등록한다 (`testing_strategy.md` 표와 동일).

---

## 브랜치 보호 (Git 호스팅 설정)

| 브랜치 | Force push | 삭제 | 필수 체크 |
|--------|------------|------|-----------|
| `main` | 금지 | 금지 | lint, unit, integration (정책에 따라) |
| `develop` | 금지 | 금지 | lint, unit (동일) |

- 릴리스 직전에는 **Integration** 실패 시 머지 불가로 두는 것을 권장한다.

---

## CI 트리거와의 대응

`infra_cicd.md` 예시: `push`는 `main`, `develop`, `pull_request` 전 구간.

| 이벤트 | 브랜치 | 기대 동작 |
|--------|--------|----------|
| PR 열림/갱신 | 임의 ↔ `develop` | lint, unit (전체 PR) |
| PR 열림/갱신 | `develop` ↔ `main` | 위 + integration |
| `push` `develop` | 통합 완료 후 | 이미지 빌드 정책에 따라 레지스트리 푸시 |
| `push` `main` | 릴리스·핫픽스 반영 | 이미지 푸시·**CD dev** 등 (`deployment_strategy.md`) |

구체적 job 이름·브랜치 필터는 저장소 `.github/workflows`에서 본 문서와 일치시킨다.

---

## 태그·릴리스

- **형식**: `vMAJOR.MINOR.PATCH` (SemVer).
- **생성 지점**: `main`에 `release` 또는 `hotfix` 반영 후.
- **컨테이너 태그**: `infra_cicd.md` — 불변 식별자로 `{git-sha}`, 릴리스 표시로 `{version}` 병행.

---

## 미결 기술 과제

- [ ] 트렁크 기반 개발(브랜치 수 최소화) 전환 여부
- [ ] `develop` 없이 `main`만 쓰는 단순 모델과의 선택 기준 문서화

---

## 관련 문서

- `infra_cicd.md` — Docker·K8s·CI/CD 워크플로
- `testing_strategy.md` — 단위·통합·E2E 시점 및 CI 매트릭스
- `deployment_strategy.md` — 환경별 배포·승인·롤백
