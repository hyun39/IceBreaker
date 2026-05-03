# S&P 500 Daily 주가 분석 — 스키마 정의 (Schema)

> 참조 공통 스펙: `common/database.md`, `common/data_governance.md`

---

## 설계 원칙

- PK 내부 조인용 `BIGSERIAL` + 외부 노출용 `TEXT` 자연키 조합
- 모든 타임스탬프는 `TIMESTAMPTZ` (UTC 기준)
- 레이어 접두사: `raw_` / `ods_` / `dim_` / `fact_` / `mart_`
- 비정규화 시 반드시 주석으로 사유 명시

---

## ODS 레이어

### `ods_sp500_ticker` — S&P 500 구성 종목 메타

```sql
CREATE TABLE ods_sp500_ticker (
    id              BIGSERIAL PRIMARY KEY,
    ticker          TEXT NOT NULL UNIQUE,          -- 종목 코드 (AAPL, MSFT …)
    company_name    TEXT NOT NULL,
    sector          TEXT NOT NULL,                 -- GICS Sector
    sub_industry    TEXT,
    exchange        TEXT,                          -- NYSE / NASDAQ
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    source_updated_at TIMESTAMPTZ,
    ingested_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_ods_ticker UNIQUE (ticker)
);

CREATE INDEX idx_ods_ticker_sector ON ods_sp500_ticker (sector);
```

### `ods_stock_price_daily` — 종목별 일별 OHLCV

```sql
CREATE TABLE ods_stock_price_daily (
    id              BIGSERIAL PRIMARY KEY,
    ticker          TEXT NOT NULL,
    trade_date      DATE NOT NULL,
    open_price      NUMERIC(12, 4),
    high_price      NUMERIC(12, 4),
    low_price       NUMERIC(12, 4),
    close_price     NUMERIC(12, 4) NOT NULL,
    adj_close_price NUMERIC(12, 4),               -- 수정 종가
    volume          BIGINT,
    source_system   TEXT NOT NULL DEFAULT 'alpha_vantage',
    batch_id        TEXT,
    record_hash     TEXT,                          -- 변경 감지용 SHA-256
    ingested_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_ods_price_daily UNIQUE (ticker, trade_date)
);

CREATE INDEX idx_ods_price_ticker_date ON ods_stock_price_daily (ticker, trade_date DESC);
CREATE INDEX idx_ods_price_date ON ods_stock_price_daily (trade_date DESC);
```

---

## DW 레이어

### `dim_sector` — Sector 차원

```sql
CREATE TABLE dim_sector (
    sector_id       BIGSERIAL PRIMARY KEY,
    sector_name     TEXT NOT NULL UNIQUE,          -- 예: Information Technology
    sector_code     TEXT NOT NULL UNIQUE,          -- 예: IT
    valid_from      DATE NOT NULL,
    valid_to        DATE,                          -- SCD Type 2: NULL = 현재 유효
    is_current      BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `dim_ticker` — 종목 차원

```sql
CREATE TABLE dim_ticker (
    ticker_id       BIGSERIAL PRIMARY KEY,
    ticker          TEXT NOT NULL,
    company_name    TEXT NOT NULL,
    sector_id       BIGINT NOT NULL REFERENCES dim_sector(sector_id),
    exchange        TEXT,
    valid_from      DATE NOT NULL,
    valid_to        DATE,
    is_current      BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_dim_ticker_history UNIQUE (ticker, valid_from)
);

CREATE INDEX idx_dim_ticker_sector ON dim_ticker (sector_id, is_current);
```

### `fact_sector_daily` — Sector별 일별 집계 Fact

```sql
CREATE TABLE fact_sector_daily (
    id              BIGSERIAL PRIMARY KEY,
    trade_date      DATE NOT NULL,
    sector_id       BIGINT NOT NULL REFERENCES dim_sector(sector_id),
    ticker_count    INT NOT NULL,                  -- 해당 날 집계된 종목 수
    avg_close       NUMERIC(12, 4) NOT NULL,       -- 단순 평균 종가
    total_volume    BIGINT NOT NULL,               -- Sector 합산 거래량
    advance_count   INT NOT NULL,                  -- 전일 대비 상승 종목 수
    decline_count   INT NOT NULL,                  -- 전일 대비 하락 종목 수
    unchanged_count INT NOT NULL,
    sector_return_pct NUMERIC(8, 4),               -- Sector 평균 수익률(%)
    top_gainer_ticker TEXT,                        -- 당일 최고 상승 종목
    top_loser_ticker  TEXT,                        -- 당일 최고 하락 종목
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_fact_sector_daily UNIQUE (trade_date, sector_id)
);

CREATE INDEX idx_fact_sector_date ON fact_sector_daily (trade_date DESC, sector_id);
```

---

## Mart 레이어

### `mart_daily_key_index` — 일별 주요 Key Index

```sql
CREATE TABLE mart_daily_key_index (
    id                  BIGSERIAL PRIMARY KEY,
    trade_date          DATE NOT NULL,
    sector_id           BIGINT NOT NULL REFERENCES dim_sector(sector_id),
    sector_name         TEXT NOT NULL,             -- 비정규화: 조회 편의
    -- 가격 지표
    avg_close           NUMERIC(12, 4) NOT NULL,
    sector_return_pct   NUMERIC(8, 4),
    -- 거래량 지표
    total_volume        BIGINT,
    volume_vs_ma20      NUMERIC(8, 4),             -- 20일 평균 거래량 대비 비율
    -- 모멘텀 지표
    advance_ratio       NUMERIC(5, 2),             -- 상승 종목 비율(%)
    ma5_close           NUMERIC(12, 4),
    ma20_close          NUMERIC(12, 4),
    ma60_close          NUMERIC(12, 4),
    rsi_14              NUMERIC(6, 2),             -- 14일 RSI (Sector 평균)
    -- 변동성 지표
    daily_range_pct     NUMERIC(8, 4),             -- (High - Low) / prev_close
    -- 순위
    sector_rank_by_return INT,                     -- 당일 수익률 기준 Sector 순위
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_mart_key_index UNIQUE (trade_date, sector_id)
);

CREATE INDEX idx_mart_key_date ON mart_daily_key_index (trade_date DESC);
CREATE INDEX idx_mart_key_sector ON mart_daily_key_index (sector_id, trade_date DESC);
```

### `mart_sector_trend_analysis` — LLM Agent 생성 트렌드 분석

```sql
CREATE TABLE mart_sector_trend_analysis (
    id                  BIGSERIAL PRIMARY KEY,
    trade_date          DATE NOT NULL,
    sector_id           BIGINT NOT NULL REFERENCES dim_sector(sector_id),
    sector_name         TEXT NOT NULL,
    -- LLM 분석 결과
    trend_summary       TEXT NOT NULL,             -- 핵심 트렌드 요약 (한국어/영어)
    key_drivers         JSONB,                     -- 주요 동인 목록 [{driver, detail}]
    sentiment           TEXT CHECK (sentiment IN ('bullish','bearish','neutral')),
    risk_factors        JSONB,                     -- 위험 요인 목록
    recommendation      TEXT,                      -- 투자 참고 의견
    -- 메타
    llm_model           TEXT NOT NULL,             -- 사용한 LLM 모델명
    prompt_version      TEXT NOT NULL,             -- 프롬프트 버전 (재현용)
    input_date_range    TEXT,                      -- 분석에 사용된 데이터 기간
    agent_run_id        TEXT,                      -- Agent 실행 추적 ID
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_mart_trend UNIQUE (trade_date, sector_id)
);

CREATE INDEX idx_mart_trend_date ON mart_sector_trend_analysis (trade_date DESC);
CREATE INDEX idx_mart_trend_sector ON mart_sector_trend_analysis (sector_id, trade_date DESC);
```

---

## 데이터 흐름 요약

```
[API] daily OHLCV
  → ods_stock_price_daily (upsert)
  → ods_sp500_ticker (upsert)
  → fact_sector_daily (집계 INSERT)
  → mart_daily_key_index (지표 계산 INSERT)
  → [Agent 실행] mart_sector_trend_analysis (LLM 결과 INSERT)
```

---

## 운영 테이블

### `ods_price_quarantine` — 수집 실패 종목 격리

```sql
CREATE TABLE ods_price_quarantine (
    id          BIGSERIAL PRIMARY KEY,
    ticker      TEXT NOT NULL,
    trade_date  DATE NOT NULL,
    reason      TEXT NOT NULL,        -- "yfinance no data", "parse error" 등
    raw_payload JSONB,                -- yfinance 원본 응답 (디버그용)
    batch_id    TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_quarantine_date ON ods_price_quarantine (trade_date DESC);
```

### `pipeline_dq_log` — DQ 검증 결과

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
CREATE INDEX idx_dq_log_date ON pipeline_dq_log (trade_date DESC);
```

---

## 거버넌스 분류

| 테이블 | 민감도 등급 | PII 포함 | 보존 기간 |
|--------|------------|---------|----------|
| `ods_stock_price_daily` | L1 (공개) | 없음 | 3년 |
| `ods_sp500_ticker` | L1 (공개) | 없음 | 3년 |
| `fact_sector_daily` | L1 (공개) | 없음 | 3년 |
| `mart_daily_key_index` | L1 (공개) | 없음 | 2년 |
| `mart_sector_trend_analysis` | L2 (내부) | 없음 | 2년 |
| `ods_price_quarantine` | L1 (공개) | 없음 | 1년 |
| `pipeline_dq_log` | L2 (내부) | 없음 | 1년 |
