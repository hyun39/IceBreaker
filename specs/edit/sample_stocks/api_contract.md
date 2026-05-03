# S&P 500 Daily 주가 분석 — API 계약 (API Contract)

> FastAPI 서버의 Pydantic 요청·응답 모델 전체를 정의한다.  
> `frontend.md`의 JSON 예시와 1:1 대응한다.

---

## 공통

### 공통 에러 응답

```python
# schemas/common.py
from pydantic import BaseModel
from typing import Any

class ErrorDetail(BaseModel):
    code: str
    message: str
    detail: dict[str, Any] | None = None

class ErrorResponse(BaseModel):
    error: ErrorDetail
```

### 페이지네이션 (Offset 기반)

```python
class PaginationMeta(BaseModel):
    page: int
    size: int
    total: int
    total_pages: int
```

### 공통 쿼리 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|---------|------|--------|------|
| `trade_date` | `date` | 필수 | 조회 거래일 (`YYYY-MM-DD`) |
| `sector_code` | `str \| None` | `None` | Sector 코드 필터 (예: `IT`) |
| `page` | `int` | `1` | 페이지 번호 (1-indexed) |
| `size` | `int` | `50` | 페이지당 행 수 (최대 200) |
| `sort` | `str` | `"ticker"` | 정렬 컬럼 |
| `order` | `"asc" \| "desc"` | `"asc"` | 정렬 방향 |

---

## `/api/v1/ods/prices` — ODS 종목별 일별 가격

### 요청

```
GET /api/v1/ods/prices
  ?trade_date=2026-05-01
  &sector_code=IT        (optional)
  &ticker=AAPL           (optional, 부분일치 검색)
  &page=1
  &size=50
  &sort=ticker
  &order=asc
```

### 응답 Pydantic 모델

```python
# schemas/ods.py
from pydantic import BaseModel, Field
from datetime import date

class OdsPriceItem(BaseModel):
    ticker: str
    company_name: str
    sector: str
    open_price: float | None
    high_price: float | None
    low_price: float | None
    close_price: float
    adj_close_price: float | None
    volume: int | None
    ingested_at: str   # ISO 8601

class OdsPriceListResponse(BaseModel):
    trade_date: date
    pagination: PaginationMeta
    items: list[OdsPriceItem]
```

### 응답 예시

```json
{
  "trade_date": "2026-05-01",
  "pagination": { "page": 1, "size": 50, "total": 503, "total_pages": 11 },
  "items": [
    {
      "ticker": "AAPL",
      "company_name": "Apple Inc.",
      "sector": "Information Technology",
      "open_price": 185.20,
      "high_price": 187.50,
      "low_price": 184.80,
      "close_price": 186.90,
      "adj_close_price": 186.90,
      "volume": 58234000,
      "ingested_at": "2026-05-01T07:12:34Z"
    }
  ]
}
```

---

## `/api/v1/dw/sector` — DW Sector 집계

### 요청

```
GET /api/v1/dw/sector
  ?trade_date=2026-05-01
  &sector_code=IT        (optional)
```

### 응답 Pydantic 모델

```python
# schemas/dw.py
class SectorAggregation(BaseModel):
    sector_name: str
    sector_code: str
    ticker_count: int
    avg_close: float
    total_volume: int
    advance_count: int
    decline_count: int
    unchanged_count: int
    sector_return_pct: float | None
    sector_rank_by_return: int | None
    top_gainer_ticker: str | None
    top_loser_ticker: str | None

class DwSectorResponse(BaseModel):
    trade_date: date
    sectors: list[SectorAggregation]
```

### 응답 예시

```json
{
  "trade_date": "2026-05-01",
  "sectors": [
    {
      "sector_name": "Information Technology",
      "sector_code": "IT",
      "ticker_count": 65,
      "avg_close": 486.32,
      "total_volume": 2341870000,
      "advance_count": 51,
      "decline_count": 14,
      "unchanged_count": 0,
      "sector_return_pct": 1.23,
      "sector_rank_by_return": 1,
      "top_gainer_ticker": "NVDA",
      "top_loser_ticker": "INTC"
    }
  ]
}
```

---

## `/api/v1/mart/key-index` — Mart Key Index (단일 날짜)

### 요청

```
GET /api/v1/mart/key-index
  ?trade_date=2026-05-01
  &sector_code=IT        (optional)
```

### 응답 Pydantic 모델

```python
# schemas/mart.py
class KeyIndexItem(BaseModel):
    trade_date: date
    sector_name: str
    sector_code: str
    avg_close: float
    sector_return_pct: float | None
    total_volume: int | None
    volume_vs_ma20: float | None
    advance_ratio: float | None
    ma5_close: float | None
    ma20_close: float | None
    ma60_close: float | None
    rsi_14: float | None
    daily_range_pct: float | None
    sector_rank_by_return: int | None

class KeyIndexResponse(BaseModel):
    trade_date: date
    items: list[KeyIndexItem]
```

---

## `/api/v1/mart/key-index/trend` — Mart Key Index 시계열

### 요청

```
GET /api/v1/mart/key-index/trend
  ?sector_code=IT    (필수)
  &days=60           (기본값: 60, 최대: 252)
```

### 응답 Pydantic 모델

```python
class KeyIndexTrendResponse(BaseModel):
    sector_code: str
    sector_name: str
    days: int
    series: list[KeyIndexItem]   # 날짜 오름차순
```

---

## `/api/v1/mart/trend-analysis` — LLM 트렌드 분석

### 요청

```
GET /api/v1/mart/trend-analysis
  ?trade_date=2026-05-01
  &sector_code=IT        (optional)
```

### 응답 Pydantic 모델

```python
class DriverItem(BaseModel):
    driver: str
    detail_ko: str    # 한국어
    detail_en: str    # 영어

class TrendAnalysisItem(BaseModel):
    sector_name: str
    sector_code: str
    sentiment: str    # "bullish" | "bearish" | "neutral"
    trend_summary_ko: str
    trend_summary_en: str
    key_drivers: list[DriverItem]
    risk_factors: list[DriverItem]
    recommendation_ko: str
    recommendation_en: str
    llm_model: str
    prompt_version: str
    input_date_range: str | None
    created_at: str
    disclaimer_ko: str = "본 분석은 투자 자문이 아니며 참고 목적으로만 활용하시기 바랍니다."
    disclaimer_en: str = "This analysis is not investment advice and is for reference purposes only."

class TrendAnalysisResponse(BaseModel):
    trade_date: date
    is_complete: bool       # Agent 분석 완료 여부
    total_sectors: int      # 전체 Sector 수 (11)
    completed_sectors: int  # 분석 완료 Sector 수
    analyses: list[TrendAnalysisItem]
```

### 응답 예시 (is_complete=false 시)

```json
{
  "trade_date": "2026-05-01",
  "is_complete": false,
  "total_sectors": 11,
  "completed_sectors": 7,
  "analyses": []
}
```

---

## `/api/v1/meta/trading-dates` — 거래일 목록

### 요청

```
GET /api/v1/meta/trading-dates
  ?from=2026-01-01   (optional, 기본: 최근 1년)
  &to=2026-05-01     (optional)
```

### 응답 Pydantic 모델

```python
class TradingDatesResponse(BaseModel):
    from_date: date
    to_date: date
    trading_dates: list[date]   # 오름차순
    latest_available: date      # 데이터가 적재된 가장 최근 날짜
```

---

## `/healthz` / `/ready` — 헬스체크

```python
class HealthResponse(BaseModel):
    status: str   # "ok"

class ReadyResponse(BaseModel):
    status: str   # "ok" | "degraded"
    db: str       # "ok" | "error"
    redis: str    # "ok" | "error" | "degraded"
```

---

## FastAPI 라우터 등록

```python
# main.py
from routers import ods, dw, mart, meta

app.include_router(ods.router)
app.include_router(dw.router)
app.include_router(mart.router)
app.include_router(meta.router)

# CORS (내부 서비스 전용)
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,   # ["http://localhost:3000"]
    allow_methods=["GET"],
    allow_headers=["Authorization"],
)
```
