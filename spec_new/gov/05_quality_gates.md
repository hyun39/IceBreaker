# GOV-05 — 테스트·품질 게이트

> **강제 대상**: 모든 PR 및 배포  
> **게이트**: 각 단계 실패 시 다음 단계 진행 불가  
> **원본 참조**: `enterprise/06.01_gov_testing_strategy.md`, `common/testing_strategy.md`

---

## 테스트 피라미드 비율 기준

```
Unit (70%) ──► Integration (20%) ──► Contract (7%) ──► E2E (3%)
   PR 즉시          PR 머지 전           PR 머지 전      main 머지 후
```

| 레이어 | 커버리지 게이트 | 실패 시 |
|--------|-------------|--------|
| Unit | 전체 80% 이상, Service 90% 이상 | PR 차단 |
| Integration | 핵심 흐름 전체 통과 | PR 차단 |
| BDD (.feature) | 모든 Scenario 통과 | PR 차단 |
| Contract (Pact) | Provider 검증 통과 | PR 차단 |
| E2E | Happy Path + 주요 예외 통과 | 배포 차단 |

---

## BDD 게이트 규칙 (MUST)

- [ ] 모든 비즈니스 스펙의 Gherkin 인수기준은 `.feature` 파일로 존재한다
- [ ] 모든 `.feature` Scenario는 Step 구현이 완료되어 있다
- [ ] BDD 테스트는 PR CI에서 자동 실행된다
- [ ] 신규 기능 PR에는 대응하는 `.feature` 변경이 포함된다

```yaml
# CI BDD 게이트
- name: BDD Tests
  run: pytest tests/bdd/ --tb=short -q
  # 실패 시 PR 머지 차단
```

---

## 커버리지 게이트 설정

### Python (pytest-cov)
```bash
pytest --cov=app --cov-fail-under=80 --cov-report=xml
```

### Java (JaCoCo)
```xml
<limit>
  <counter>LINE</counter>
  <value>COVEREDRATIO</value>
  <minimum>0.80</minimum>
</limit>
```

### CI 통합
```yaml
- uses: codecov/codecov-action@v4
  with:
    fail_ci_if_error: true
    threshold: 80        # 전체 커버리지 80% 미만 시 실패
```

---

## 전체 CI 게이트 순서

```
PR 생성
  ├─ lint / type-check           (실패 → 즉시 차단)
  ├─ unit test + coverage        (80% 미만 → 차단)
  ├─ BDD test                    (실패 → 차단)
  └─ security scan (Trivy)       (HIGH+ → 차단)

PR 승인 후 머지 전
  └─ integration test            (실패 → 차단)

main 머지
  └─ E2E test (dev 환경)         (실패 → staging 차단)

릴리스 전 (수동)
  └─ performance test            (p95 기준 미달 → 배포 검토)
```

---

## 테스트 환경 격리 원칙

| 규칙 | 내용 |
|------|------|
| DB Mock 금지 | 통합 테스트는 실제 DB (Testcontainers) 사용 |
| LLM Mock 필수 | 비용·속도 이유로 외부 LLM은 Mock |
| 외부 API Mock | Scrapin·Twitter·Tavily 등 외부 API는 Mock |
| 테스트 격리 | 각 테스트는 독립적 — 순서 의존성 금지 |

---

## DoD (Definition of Done)

PR이 머지되기 전 반드시 충족해야 하는 기준:

```
[ ] 모든 CI 게이트 통과
[ ] 코드 리뷰 최소 1인 승인
[ ] 비즈니스 스펙의 모든 인수기준 BDD로 검증됨
[ ] 새 기능에 대한 unit test 포함
[ ] CHANGELOG.md 또는 PR 설명에 변경 내용 기록
[ ] 보안 취약점 없음 (HIGH/CRITICAL)
```
