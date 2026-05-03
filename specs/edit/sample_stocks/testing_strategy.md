# S&P 500 Daily 주가 분석 — 테스트 전략 (Testing Strategy)

> 참조 공통 스펙: `common/testing_strategy.md`

---

## 테스트 피라미드

```
        ╱▔▔▔▔▔▔▔╲
       ╱  E2E(5%) ╲         Playwright — 화면 탭별 핵심 시나리오
      ╱─────────────╲
     ╱ Integration   ╲
    ╱    (25%)        ╲      Testcontainers — 실 DB, DAG 통합
   ╱───────────────────╲
  ╱    Unit (70%)       ╲   pytest — Service/Repo/Agent 로직
 ╱───────────────────────╲
```

| 레이어 | 비율 | 실행 시점 | 실패 시 |
|--------|------|----------|--------|
| Unit | 70% | PR 생성 즉시 | PR 머지 차단 |
| Integration | 25% | PR 머지 전 | PR 머지 차단 |
| E2E | 5% | main 머지 후 (dev 배포 후) | staging 배포 차단 |

---

## 단위 테스트 (pytest)

### 원칙
- FIRST: Fast, Isolated, Repeatable, Self-validating, Timely
- AAA: Arrange → Act → Assert
- DB, Redis, 외부 API, LLM 모두 Mock
- 커버리지 목표: Service 90% 이상, 전체 80% 이상

### FastAPI Service 계층

```python
# tests/unit/test_mart_service.py
import pytest
from unittest.mock import AsyncMock
from app.services.mart_service import MartService

@pytest.fixture
def mart_repo():
    return AsyncMock()

@pytest.fixture
def mart_service(mart_repo):
    return MartService(repo=mart_repo)

class TestMartService:
    async def test_get_key_index_returns_items(self, mart_service, mart_repo):
        mart_repo.get_key_index.return_value = [
            {"sector_code": "IT", "sector_name": "Information Technology",
             "avg_close": 486.32, "rsi_14": 62.4}
        ]
        result = await mart_service.get_key_index(
            trade_date=date(2026, 5, 1), sector_code=None
        )
        assert len(result.items) == 1
        assert result.items[0].sector_code == "IT"

    async def test_get_key_index_empty_date_returns_not_found(self, mart_service, mart_repo):
        mart_repo.get_key_index.return_value = []
        with pytest.raises(AppException) as exc:
            await mart_service.get_key_index(date(2026, 1, 1), None)
        assert exc.value.code == "DATA_NOT_FOUND"
```

### Airflow Task 함수

```python
# tests/unit/test_pipeline_tasks.py
import pytest
from unittest.mock import patch, MagicMock
import pandas as pd
from dags.tasks.ingest import fetch_single_ticker_yfinance

class TestFetchSingleTicker:
    @patch("yfinance.Ticker")
    def test_fetch_returns_ohlcv_dict(self, mock_ticker):
        mock_ticker.return_value.history.return_value = pd.DataFrame({
            "Open": [185.0], "High": [187.0], "Low": [184.0],
            "Close": [186.0], "Volume": [5000000]
        }, index=pd.DatetimeIndex(["2026-05-01"]))

        result = fetch_single_ticker_yfinance("AAPL", "2026-05-01")
        assert result["ticker"] == "AAPL"
        assert result["close_price"] == 186.0

    @patch("yfinance.Ticker")
    def test_empty_history_raises(self, mock_ticker):
        mock_ticker.return_value.history.return_value = pd.DataFrame()
        with pytest.raises(ValueError, match="no data"):
            fetch_single_ticker_yfinance("INVALID", "2026-05-01")
```

### LLM Agent (Pydantic 파싱)

```python
# tests/unit/test_agent_output_parser.py
from app.agent.trend_agent import parse_trend_output

def test_parse_valid_bilingual_output():
    raw = """{
      "trend_summary_ko": "IT 섹터 강세",
      "trend_summary_en": "IT sector bullish",
      "key_drivers": [{"driver": "AI", "detail_ko": "AI 수요", "detail_en": "AI demand"}],
      "sentiment": "bullish",
      "risk_factors": [],
      "recommendation_ko": "비중 유지",
      "recommendation_en": "Maintain position"
    }"""
    result = parse_trend_output(raw)
    assert result.sentiment == "bullish"
    assert result.trend_summary_ko == "IT 섹터 강세"

def test_parse_invalid_sentiment_raises():
    raw = '{"sentiment": "unknown", ...}'
    with pytest.raises(ValueError):
        parse_trend_output(raw)
```

### 커버리지 설정

```ini
# pytest.ini
[pytest]
asyncio_mode = auto
testpaths = tests
addopts = --cov=app --cov-report=xml --cov-fail-under=80
```

---

## 통합 테스트 (Testcontainers)

### 원칙
- 실제 PostgreSQL 15 + Redis 7 컨테이너 사용
- LLM, yfinance 외부 호출만 Mock
- 테스트 간 DB 상태 격리 (트랜잭션 롤백 또는 픽스처 초기화)

### FastAPI 통합 테스트

```python
# tests/integration/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer
from testcontainers.redis import RedisContainer
from fastapi.testclient import TestClient
from app.main import app
from app.core.database import get_db
from alembic.config import Config
from alembic import command

@pytest.fixture(scope="session")
def postgres():
    with PostgresContainer("postgres:15") as pg:
        # Alembic 마이그레이션 적용
        alembic_cfg = Config("alembic.ini")
        alembic_cfg.set_main_option("sqlalchemy.url", pg.get_connection_url())
        command.upgrade(alembic_cfg, "head")
        yield pg

@pytest.fixture(scope="session")
def redis_container():
    with RedisContainer("redis:7-alpine") as r:
        yield r

@pytest.fixture(scope="function")
def client(postgres, redis_container):
    # 의존성 오버라이드
    app.dependency_overrides[get_db] = lambda: get_test_db(postgres)
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

```python
# tests/integration/test_ods_api.py
def test_ods_prices_returns_200(client, seed_ods_data):
    response = client.get("/api/v1/ods/prices", params={"trade_date": "2026-05-01"})
    assert response.status_code == 200
    assert response.json()["pagination"]["total"] > 0

def test_ods_prices_nontrading_day_returns_404(client):
    response = client.get("/api/v1/ods/prices", params={"trade_date": "2026-05-03"})  # 주말
    assert response.status_code == 404
    assert response.json()["error"]["code"] == "NON_TRADING_DAY"

def test_trend_analysis_pending_returns_202(client):
    response = client.get("/api/v1/mart/trend-analysis", params={"trade_date": "2026-05-01"})
    assert response.status_code == 202
    assert response.json()["is_complete"] is False
```

### Airflow DAG 통합 테스트

```python
# tests/integration/test_dags.py
from airflow.models import DagBag

def test_dag_loads_without_error():
    dagbag = DagBag(dag_folder="airflow/dags", include_examples=False)
    assert len(dagbag.import_errors) == 0

def test_ingest_dag_task_count():
    dagbag = DagBag(dag_folder="airflow/dags", include_examples=False)
    dag = dagbag.get_dag("ingest_sp500_daily_price")
    assert dag is not None
    assert len(dag.tasks) >= 3  # sensor, fetch, save

def test_build_dw_dag_dependencies():
    dagbag = DagBag(dag_folder="airflow/dags", include_examples=False)
    dag = dagbag.get_dag("build_dw_sector")
    task_ids = [t.task_id for t in dag.tasks]
    assert "upsert_fact_sector" in task_ids
    assert "dq_fact_sector" in task_ids
```

---

## E2E 테스트 (Playwright)

### 대상 시나리오

| 시나리오 | 우선순위 |
|---------|---------|
| 로그인 → ODS 탭 데이터 표시 | 필수 |
| DW 탭 Sector 차트 표시 | 필수 |
| Mart Key Index 카드 표시 | 필수 |
| Trend Analysis 완료 카드 표시 | 필수 |
| Trend Analysis 로딩 중 spinner 표시 | 권장 |
| 비거래일 선택 → 최근 거래일 자동 이동 | 권장 |

```typescript
// tests/e2e/sp500.spec.ts
import { test, expect } from "@playwright/test";

test.beforeEach(async ({ page }) => {
  // dev-analyst 계정으로 Keycloak 로그인
  await page.goto("/");
  await page.fill('#username', 'dev-analyst');
  await page.fill('#password', 'dev-analyst');
  await page.click('#kc-login');
  await expect(page).toHaveURL(/\//);
});

test("ODS 탭: 종목 데이터가 테이블로 표시된다", async ({ page }) => {
  await page.click('[data-tab="ods"]');
  await expect(page.locator("table tbody tr")).toHaveCount(50);  // 기본 50행
  await expect(page.locator("td:has-text('AAPL')")).toBeVisible();
});

test("Mart Trend: 분석 완료 시 카드가 표시된다", async ({ page }) => {
  await page.click('[data-tab="trend"]');
  await expect(page.locator(".trend-card")).toHaveCount(11, { timeout: 60_000 });
  await expect(page.locator(".disclaimer")).toContainText("투자 자문이 아니며");
});

test("비거래일 선택 시 최근 거래일로 이동한다", async ({ page }) => {
  await page.fill('[data-testid="date-picker"]', "2026-05-03");  // 일요일
  await expect(page.locator("[data-testid='notice']")).toContainText("비거래일");
  await expect(page.locator("[data-testid='date-picker']")).not.toHaveValue("2026-05-03");
});
```

```typescript
// playwright.config.ts
export default {
  testDir: "./tests/e2e",
  timeout: 90_000,
  retries: process.env.CI ? 2 : 0,
  use: {
    baseURL: process.env.E2E_BASE_URL ?? "http://localhost:3000",
    trace: "retain-on-failure",
    screenshot: "only-on-failure",
  },
};
```

---

## CI 통합 매트릭스

```
PR 생성
  └─ Unit Test + Lint      → 실패 시 리뷰 차단
  └─ Type Check (mypy)     → 실패 시 리뷰 차단

PR 승인 후 머지 전
  └─ Integration Test      → 실패 시 머지 차단

main 머지 후
  └─ E2E Test (dev 환경)   → 실패 시 staging 배포 차단
```

---

## 테스트 픽스처 데이터

```python
# tests/fixtures/seed.py
FIXTURE_SECTORS = [
    {"sector_id": 1, "sector_name": "Information Technology", "sector_code": "IT"},
    # ... 11개
]

FIXTURE_TICKERS = [
    {"ticker": "AAPL", "company_name": "Apple Inc.", "sector_id": 1},
    {"ticker": "MSFT", "company_name": "Microsoft Corp.", "sector_id": 1},
]

FIXTURE_PRICES = [
    {"ticker": "AAPL", "trade_date": "2026-05-01",
     "open_price": 185.0, "high_price": 187.0,
     "low_price": 184.0, "close_price": 186.0, "volume": 5000000},
]
```
