# GOV-02 — Architecture Decision Record 정책

> **강제 대상**: 아키텍처에 영향을 미치는 모든 기술 결정  
> **게이트**: ADR 없는 주요 기술 변경 PR → 리뷰어 머지 거부  
> **원본 참조**: `enterprise/02.01_gov_adr_format.md`, `common/adr_format.md`

---

## ADR을 작성해야 하는 상황 (MUST)

| 상황 | 예시 |
|------|------|
| 새 기술·라이브러리 도입 | LangChain4j 채택, Kafka 도입 |
| 기존 기술 교체 | Flask → FastAPI, MySQL → PostgreSQL |
| 아키텍처 패턴 변경 | 모노리스 → 마이크로서비스 |
| 보안·데이터 정책 결정 | PII 암호화 수준, 토큰 보존 정책 |
| 인프라 구조 변경 | K8s Executor 교체, 멀티 클러스터 전환 |

## ADR이 불필요한 상황

- 버그 픽스
- 기존 패턴 내 기능 추가
- 의존성 버전 마이너 업데이트

---

## 필수 섹션 (MUST)

```markdown
# ADR-{NNNN}: {결정 제목}

## 상태          → 제안됨 | 수락됨 | 폐기됨 | 대체됨(ADR-NNNN)
## 날짜          → YYYY-MM-DD
## 결정자        → 이름 또는 팀명
## 배경          → 결정이 필요한 기술·조직적 상황
## 검토한 옵션   → 최소 2개 + 각 장단점
## 결정          → "우리는 {X}를 선택한다. 왜냐하면 {Y}이기 때문이다."
## 결과          → 긍정적·부정적·트레이드오프
```

---

## 파일 네이밍·위치 규칙

```
docs/adr/ADR-{NNNN}-{kebab-case-title}.md

예:
  docs/adr/ADR-0001-langchain-for-llm-orchestration.md
  docs/adr/ADR-0002-postgresql-as-primary-database.md
  docs/adr/ADR-0003-keycloak-for-authentication.md
```

- 번호: 4자리 0-패딩 순번
- 수락된 ADR은 **수정하지 않는다** — 변경 필요 시 새 ADR로 대체
- `docs/adr/README.md` 인덱스를 항상 최신 상태로 유지

---

## 상태 전이

```
제안됨 ──► 수락됨 ──► 폐기됨
                └──► 대체됨 (→ 새 ADR 번호 명시)
```

---

## 금지 사항

| 금지 | 이유 |
|------|------|
| 수락된 ADR 내용 소급 수정 | 결정 당시 컨텍스트가 사라짐 |
| 부정적 결과 누락 | 미래 독자가 트레이드오프를 알 수 없음 |
| 검토한 옵션 1개만 기재 | 선택지 비교가 ADR의 핵심 가치 |
