# S&P 500 Daily 주가 분석 — 데이터 파이프라인 (Airflow)

> 참조 공통 스펙: `common/data_pipeline_airflow_ods_dw_mart.md`, `common/infra_cicd.md`

---

## DAG 목록

| DAG ID | 역할 | 스케줄 | 의존 DAG |
|--------|------|--------|---------|
| `ingest_sp500_tickers` | S&P 500 종목 메타 수집 | `0 6 * * MON` (주 1회) | — |
| `ingest_sp500_daily_price` | 일별 OHLCV 수집 → Raw | `0 7 * * 1-5` (거래일 장 마감 후) | `ingest_sp500_tickers` |
| `load_raw_to_ods` | Raw → ODS 적재 및 품질 검증 | `30 7 * * 1-5` | `ingest_sp500_daily_price` |
| `build_dw_sector` | ODS → DW Sector 집계 | `0 8 * * 1-5` | `load_raw_to_ods` |
| `build_mart_daily_index` | DW → Mart Key Index 생성 | `30 8 * * 1-5` | `build_dw_sector` |
| `run_agent_trend` | Mart → LLM Agent 트렌드 분석 | `0 9 * * 1-5` | `build_mart_daily_index` |

> 스케줄은 미 동부시간 기준 장 마감(16:00 ET) 이후를 감안하여 UTC로 설정한다.  
> **[확정]** 공휴일 캘린더: `exchange_calendars` 라이브러리 (`XNYS` = NYSE 캘린더)  
> **[확정]** 주가 API: `yfinance` (무료, 병렬 수집 ~3분)  
> **[확정]** Raw 스토리지: 로컬 Parquet (`./data/raw/sp500/price/dt={date}/`)

---

## 전체 의존 흐름

```
ingest_sp500_tickers (월 1회)
        │
        ▼
ingest_sp500_daily_price ──► Raw Parquet/S3
        │
        ▼
load_raw_to_ods ──► ODS (PostgreSQL)
        │
        ▼
build_dw_sector ──► DW (PostgreSQL)
        │
        ▼
build_mart_daily_index ──► mart_daily_key_index
        │
        ▼
run_agent_trend ──► mart_sector_trend_analysis
```

---

## DAG 상세

### 1. `ingest_sp500_tickers`

**목적**: Wikipedia/API에서 S&P 500 구성 종목 메타 수집 → `ods_sp500_ticker` upsert

```python
with DAG(
    dag_id="ingest_sp500_tickers",
    schedule="0 6 * * MON",
    catchup=False,
    max_active_runs=1,
    default_args={"retries": 2, "retry_delay": timedelta(minutes=5)},
) as dag:

    fetch = PythonOperator(
        task_id="fetch_sp500_list",
        python_callable=fetch_sp500_from_wikipedia,  # pandas read_html
        execution_timeout=timedelta(minutes=10),
    )
    upsert = PythonOperator(
        task_id="upsert_ods_tickers",
        python_callable=upsert_ods_sp500_ticker,
        execution_timeout=timedelta(minutes=5),
    )
    dq = PythonOperator(
        task_id="dq_ticker_count",
        python_callable=check_ticker_count_gt_490,    # 500 ± 10 허용
    )
    fetch >> upsert >> dq
```

---

### 2. `ingest_sp500_daily_price`

**목적**: `yfinance`로 전 종목 OHLCV 수집 → 로컬 Parquet 저장

```python
with DAG(
    dag_id="ingest_sp500_daily_price",
    schedule="0 7 * * 1-5",
    catchup=False,
    max_active_runs=1,
    default_args={"retries": 3, "retry_delay": timedelta(minutes=10)},
) as dag:

    market_sensor = PythonSensor(
        task_id="check_market_open",
        python_callable=is_us_market_trading_day,
        timeout=600,
    )
    fetch_prices = PythonOperator(
        task_id="fetch_daily_prices",
        python_callable=fetch_all_sp500_ohlcv_yfinance,
        execution_timeout=timedelta(minutes=30),  # yfinance 병렬: ~3min
    )
    save_raw = PythonOperator(
        task_id="save_to_raw_parquet",
        python_callable=save_parquet_local,  # ./data/raw/sp500/price/dt={date}/
    )
    market_sensor >> fetch_prices >> save_raw
```

**yfinance 수집 구현**

```python
# dags/tasks/ingest.py
import yfinance as yf
import pandas as pd
import exchange_calendars as ecals

def is_us_market_trading_day(trade_date: str) -> bool:
    """exchange_calendars로 NYSE 거래일 여부 확인"""
    cal = ecals.get_calendar("XNYS")
    return cal.is_session(trade_date)

def fetch_all_sp500_ohlcv_yfinance(trade_date: str, dev_mode: bool = False) -> list[dict]:
    tickers = get_tickers(dev_mode)
    # yfinance는 rate limit 없음 — 배치 다운로드 가능
    raw = yf.download(
        tickers=" ".join(tickers),
        start=trade_date,
        end=pd.Timestamp(trade_date) + pd.Timedelta(days=1),
        auto_adjust=True,
        threads=True,          # 병렬 다운로드
        group_by="ticker",
    )
    results = []
    for ticker in tickers:
        try:
            row = raw[ticker].loc[trade_date]
            results.append({
                "ticker": ticker,
                "trade_date": trade_date,
                "open_price": round(float(row["Open"]), 4),
                "high_price": round(float(row["High"]), 4),
                "low_price":  round(float(row["Low"]),  4),
                "close_price": round(float(row["Close"]), 4),
                "adj_close_price": round(float(row["Close"]), 4),  # auto_adjust=True 시 동일
                "volume": int(row["Volume"]),
            })
        except (KeyError, TypeError):
            log_quarantine(ticker, trade_date, "yfinance no data")
    return results

def save_parquet_local(records: list[dict], trade_date: str):
    path = f"./data/raw/sp500/price/dt={trade_date}/data.parquet"
    os.makedirs(os.path.dirname(path), exist_ok=True)
    pd.DataFrame(records).to_parquet(path, index=False)
```

**수집 원칙**
- yfinance `threads=True`로 전 종목 병렬 다운로드 (~3분)
- `auto_adjust=True`로 수정 주가 자동 반영
- 재시도 시 동일 파일 덮어쓰기(멱등)
- 데이터 없는 종목 → `ods_price_quarantine` 기록

---

### 3. `load_raw_to_ods`

**목적**: Raw Parquet → `ods_stock_price_daily` 품질 검증 후 upsert

```python
with TaskGroup("validate") as validate:
    check_null   = PythonOperator(task_id="check_null_close")   # close_price not null
    check_volume = PythonOperator(task_id="check_volume_gt_0")  # volume >= 0
    check_date   = PythonOperator(task_id="check_trade_date")   # trade_date == execution_date

with TaskGroup("load") as load:
    upsert_price  = PythonOperator(task_id="upsert_ods_price")
    record_batch  = PythonOperator(task_id="record_batch_meta") # batch_id, ingested_at 기록

validate >> load
```

**DQ 임계치**

| 검증 항목 | 임계치 | 실패 시 |
|-----------|--------|--------|
| close_price null 비율 | < 1% | DAG fail |
| volume < 0 건수 | 0 | DAG fail |
| 수집 종목 수 | > 480 | 경고 알림 |
| trade_date 불일치 | 0건 | DAG fail |

---

### 4. `build_dw_sector`

**목적**: `ods_stock_price_daily` + `dim_ticker` → `fact_sector_daily` Sector 집계

```sql
-- build_dw_sector.sql (완전한 쿼리)
WITH prev_day AS (
    -- 직전 거래일 종가 (비거래일 건너뜀)
    SELECT
        o.ticker,
        o.close_price AS prev_close,
        o.trade_date  AS prev_date
    FROM ods_stock_price_daily o
    WHERE o.trade_date = (
        SELECT MAX(trade_date)
        FROM ods_stock_price_daily
        WHERE trade_date < :trade_date
    )
),
ranked AS (
    SELECT
        o.ticker,
        o.trade_date,
        o.close_price,
        o.high_price,
        o.low_price,
        o.volume,
        p.prev_close,
        CASE WHEN p.prev_close > 0
             THEN (o.close_price - p.prev_close) / p.prev_close * 100
             ELSE NULL END                                              AS return_pct,
        RANK() OVER (
            PARTITION BY t.sector_id
            ORDER BY
              CASE WHEN p.prev_close > 0
                   THEN (o.close_price - p.prev_close) / p.prev_close
                   ELSE NULL END DESC NULLS LAST
        )                                                               AS rank_gain,
        RANK() OVER (
            PARTITION BY t.sector_id
            ORDER BY
              CASE WHEN p.prev_close > 0
                   THEN (o.close_price - p.prev_close) / p.prev_close
                   ELSE NULL END ASC NULLS LAST
        )                                                               AS rank_loss,
        t.sector_id
    FROM ods_stock_price_daily o
    JOIN dim_ticker t ON t.ticker = o.ticker AND t.is_current = TRUE
    LEFT JOIN prev_day p ON p.ticker = o.ticker
    WHERE o.trade_date = :trade_date
)
INSERT INTO fact_sector_daily (
    trade_date, sector_id, ticker_count,
    avg_close, total_volume,
    advance_count, decline_count, unchanged_count,
    sector_return_pct, top_gainer_ticker, top_loser_ticker,
    created_at
)
SELECT
    :trade_date                                                          AS trade_date,
    r.sector_id,
    COUNT(*)                                                             AS ticker_count,
    ROUND(AVG(r.close_price)::numeric, 4)                               AS avg_close,
    SUM(r.volume)                                                        AS total_volume,
    COUNT(*) FILTER (WHERE r.return_pct > 0)                            AS advance_count,
    COUNT(*) FILTER (WHERE r.return_pct < 0)                            AS decline_count,
    COUNT(*) FILTER (WHERE r.return_pct = 0 OR r.return_pct IS NULL)   AS unchanged_count,
    ROUND(AVG(r.return_pct)::numeric, 4)                                AS sector_return_pct,
    MAX(r.ticker) FILTER (WHERE r.rank_gain = 1)                        AS top_gainer_ticker,
    MAX(r.ticker) FILTER (WHERE r.rank_loss = 1)                        AS top_loser_ticker,
    NOW()                                                                AS created_at
FROM ranked r
GROUP BY r.sector_id
ON CONFLICT (trade_date, sector_id) DO UPDATE SET
    ticker_count       = EXCLUDED.ticker_count,
    avg_close          = EXCLUDED.avg_close,
    total_volume       = EXCLUDED.total_volume,
    advance_count      = EXCLUDED.advance_count,
    decline_count      = EXCLUDED.decline_count,
    unchanged_count    = EXCLUDED.unchanged_count,
    sector_return_pct  = EXCLUDED.sector_return_pct,
    top_gainer_ticker  = EXCLUDED.top_gainer_ticker,
    top_loser_ticker   = EXCLUDED.top_loser_ticker;
```

---

### 5. `build_mart_daily_index`

**목적**: `fact_sector_daily` + 이동평균/RSI 계산 → `mart_daily_key_index`

계산 지표:

| 지표 | 계산 방법 |
|------|-----------|
| `ma5/ma20/ma60_close` | Sector avg_close 이동 평균 (window 함수) |
| `rsi_14` | 14일 종목 평균 RSI → Sector 단순 평균 |
| `volume_vs_ma20` | 당일 volume / 20일 평균 volume |
| `advance_ratio` | advance_count / ticker_count × 100 |
| `daily_range_pct` | Sector 종목 평균 (high-low)/prev_close |
| `sector_rank_by_return` | 당일 수익률 기준 Sector 순위 `RANK()` |

---

### 6. `run_agent_trend` (Agent 호출)

**목적**: `mart_daily_key_index` → LLM Agent → `mart_sector_trend_analysis` 적재

→ 상세는 `agent_trend.md` 참조

---

## 공통 운영 기준

| 항목 | 기준 |
|------|------|
| `catchup` | `False` (필요 시 CLI `backfill` 명시 실행) |
| `max_active_runs` | `1` (중복 실행 방지) |
| `retries` | 3회, 지수 백오프 |
| 실패 알림 | Slack `#data-alerts` 채널 |
| 재처리 | 날짜 파티션 단위, 멱등 보장 |
| Late data | 최근 3 거래일 재처리 허용 |

---

## 데이터 품질 결과 테이블

```sql
CREATE TABLE pipeline_dq_log (
    id          BIGSERIAL PRIMARY KEY,
    dag_id      TEXT NOT NULL,
    run_id      TEXT NOT NULL,
    trade_date  DATE NOT NULL,
    check_name  TEXT NOT NULL,
    status      TEXT CHECK (status IN ('pass', 'warn', 'fail')),
    detail      JSONB,
    checked_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 미결 기술 과제

- [x] 주가 API: **yfinance** 확정
- [x] Raw 스토리지: **로컬 Parquet** 확정
- [x] 이동평균·RSI 계산: **SQL Window 함수** 확정 (Python 불필요)
- [x] 비거래일 캘린더: **exchange_calendars XNYS** 확정
