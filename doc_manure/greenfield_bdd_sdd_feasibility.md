# Greenfield BDD/SDD 개발 착수 가능 여부 분석

> 질문: `specs/edit/enterprise` 스펙으로 그린필드 방식의 BDD/SDD 개발이 가능한가?  
> 작성일: 2026-05-03  
> 대상 스펙: 27개 파일 enterprise spec

---

## 결론

> **"개념적으로 완전하지만, 실행 환경이 없습니다."**  
> 스펙이 훌륭한 청사진이지만, 코드를 한 줄 짜려면 **3주의 선행 작업**이 필요합니다.

---

## 영역별 현실 점검

### ✅ 지금 당장 가능 (스펙만으로)

| 작업 | 근거 스펙 |
|------|---------|
| `business_spec.md` 작성 | `01.01` 템플릿 완비 |
| `api_contract.md` 설계 | `06.02` OpenAPI 설계 기준 |
| Gherkin feature 파일 작성 | `01.03` 형식 완비 |
| ADR-0001 등록 | `02.01` 템플릿 완비 |
| L1 React Skeleton 수동 작성 | `01.02` 패턴 완비 |
| 브랜치 보호 규칙 설정 | `02.02` 기준 완비 |

### ⚠️ 스펙은 있지만 실제 환경 없음 (2~3주 설정 필요)

| 필요한 것 | 우회책 |
|----------|--------|
| K8s + ArgoCD 클러스터 | Docker Compose로 우선 대체 |
| Keycloak 서버 | Docker로 로컬 기동 |
| GitHub Actions 워크플로 | 스펙 예시 보고 직접 작성 (~1일) |
| Pact Broker | PactFlow 무료 또는 Docker |

### ❌ 스펙에만 존재, 코드 없음 (직접 만들어야)

| 필요한 것 | 현실 |
|----------|------|
| `@corp/platform-sdk` | 코드 없음 — 설명만 있음 |
| `@corp/design-system` | 코드 없음 — 설명만 있음 |
| `scripts/proto/*.py` | 예시 코드만 있음, 실행 불가 |
| Backstage Golden Path 템플릿 | 인스턴스 없음 |

---

## 현실적인 착수 로드맵

### Week 1 — 스펙만으로 시작

```bash
# 레포 생성 (Backstage 없이 직접)
gh repo create corp/sp500-platform

# 디렉토리 구조 수동 생성 (02.04 참조)
mkdir -p specs apps/api apps/frontend tests/bdd/features docs

# 스펙 작성 → 바로 시작 가능
vim specs/business_spec.md    # 01.01 형식
vim specs/api_contract.md     # 06.02 형식
vim docs/adr/ADR-0001-xxx.md  # 02.01 형식
vim tests/bdd/features/mart_trend.feature  # 01.03 형식
```

**결과물**: 스펙 3개 + feature 1개 + ADR 1개 → Living Spec의 씨앗

### Week 2 — Docker Compose로 환경 구성

```yaml
# docker-compose.yml (간소화)
services:
  postgres: { image: postgres:15, ports: ["5432:5432"] }
  redis:    { image: redis:7-alpine, ports: ["6379:6379"] }
  keycloak: { image: quay.io/keycloak/keycloak:24.0, command: start-dev, ports: ["8180:8080"] }
  api:      { build: ./apps/api, ports: ["8000:8000"] }
```

```bash
# @corp/platform-sdk 없이 직접 FastAPI 구성 (나중에 교체)
uv init && uv add fastapi sqlalchemy asyncpg
uv add --dev pytest-bdd

# 첫 BDD 실행
uv run pytest tests/bdd/ -v
```

**결과물**: 로컬 동작 환경 + BDD 첫 실행

### Week 3 — GitHub Actions CI 연결

```yaml
# .github/workflows/ci.yml
jobs:
  lint:      { run: "uv run ruff check ." }
  bdd-test:  { run: "uv run pytest tests/bdd/ --tb=short" }
  spec-drift:
    run: |
      uv run python -c "from app.main import app; import json; print(json.dumps(app.openapi()))" > /tmp/current.json
      diff specs/api_contract_baseline.json /tmp/current.json || echo "⚠️ Drift 감지"
```

**결과물**: CI 파이프라인 + Drift 경고 모드

### Month 2+ — 실제 플랫폼으로 전환

```
Docker Compose → K8s + ArgoCD           (04.03)
직접 FastAPI   → @corp/platform-sdk     (05.01 별도 개발)
shadcn/ui      → @corp/design-system    (03.01 별도 개발)
warn-only CI   → block 모드             (04.04)
```

---

## Gap 분석 요약

| 항목 | 상태 | 우회책 |
|------|------|--------|
| 스펙 작성 방법 | ✅ 완전 | — |
| BDD 방법론 | ✅ 완전 | — |
| API 설계 기준 | ✅ 완전 | — |
| Docker 로컬 환경 | ⚠️ 직접 구성 | docker-compose.yml 1개 |
| `@corp/platform-sdk` | ❌ 없음 | 초기엔 직접 FastAPI 구성 |
| `@corp/design-system` | ❌ 없음 | 초기엔 shadcn/ui |
| CI 워크플로 파일 | ❌ 없음 | 스펙 예시 보고 직접 작성 |
| K8s 클러스터 | ❌ 없음 | Docker Compose로 시작 |

---

## 착수 준비도 시각화

```
스펙 문서 작성         ████████████████ 100% ← 바로 시작
BDD feature 작성       ████████████████ 100% ← 바로 시작
코드 구조 설계         ████████████████ 100% ← 바로 시작
로컬 개발 환경         ████████░░░░░░░░  50% ← Docker 1~2일
CI 파이프라인         ████████░░░░░░░░  50% ← 직접 작성 1~3일
공유 라이브러리        ████░░░░░░░░░░░░  25% ← 대체재 또는 신규 개발
운영 플랫폼 (K8s)      ████░░░░░░░░░░░░  25% ← Docker로 시작 후 전환
```

---

## 핵심 인사이트

> 스펙은 **"어떻게 만들어야 하는가"를 완전히 설명**하지만,  
> **"만드는 데 필요한 도구(SDK, 플랫폼)"는 별도로 구축**해야 한다.  
> 이것이 Platform Engineering 팀의 역할이다 (`04.01`이 정의함).

- **Week 1~3**: 스펙 + BDD + 로컬 환경으로 실제 기능 개발 시작 가능
- **Month 2+**: 완전한 enterprise 플랫폼으로 점진적 전환
- **핵심 우회책**: `@corp/sdk` 대신 직접 구성 → SDK 완성 후 교체

---

## 관련 enterprise 스펙

| 파일 | 역할 |
|------|------|
| `04.04_std_spec_code_bootstrap.md` | Greenfield/Brownfield 초기 동기화 상세 |
| `01.03_std_executable_specification.md` | BDD 프레임워크 연동 상세 |
| `07.02_std_living_specification.md` | Living Spec 구성 상세 |
| `02.04_gov_repository_structure.md` | 레포 구조 표준 |
