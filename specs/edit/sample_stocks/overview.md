# S&P 500 Daily 주가 분석 서비스 — 개요 (Overview)

> 참조 공통 스펙: `common/data_pipeline_airflow_ods_dw_mart.md`, `common/agent.md`, `common/frontend.md`

---

## 서비스 목적

미국 S&P 500 구성 종목의 일별 주가 데이터를 자동 수집·정제·집계하고,  
**Sector 단위 트렌드를 LLM Agent가 분석**해 투자 참고 인사이트를 제공하는 내부 분석 플랫폼이다.

---

## 핵심 가치

| 가치 항목 | 설명 |
|----------|------|
| 자동화된 데이터 파이프라인 | 매 거래일 장 마감 후 주가 자동 수집·적재 |
| 계층화된 데이터 모델 | ODS → DW → Mart 단계별 품질·집계 보장 |
| LLM 기반 Sector 트렌드 | Agent가 Mart 데이터를 분석해 인사이트 자동 생성 |
| 통합 조회 UI | ODS 원본부터 Mart 인사이트까지 단일 화면에서 확인 |

---

## 이해관계자

| 역할 | 관심사 |
|------|--------|
| 데이터 엔지니어 | 파이프라인 안정성, DAG 재처리, 데이터 품질 |
| 분석가/퀀트 | Sector 집계 정확성, Mart 지표 정의 |
| 프론트엔드 개발자 | API 응답 스키마, 레이어별 조회 속도 |
| AI 엔지니어 | Agent 정확도, 프롬프트 품질, LLM 비용 |

---

## 전체 아키텍처

```
[Data Source]
  Alpha Vantage / Yahoo Finance API
  ── S&P 500 종목 메타 (ticker, sector, company)
  ── Daily OHLCV (open, high, low, close, volume)
            │
            ▼  (Airflow DAG: ingest_sp500_daily)
  ┌─────────────────────────────────┐
  │  Landing/Raw  (Parquet, S3)     │  원본 보존, append-only
  └─────────────────────────────────┘
            │
            ▼  (Airflow DAG: load_raw_to_ods)
  ┌─────────────────────────────────┐
  │  ODS  (PostgreSQL)              │  정합성 검증, upsert
  │  ods_stock_price_daily          │
  │  ods_sp500_ticker               │
  └─────────────────────────────────┘
            │
            ▼  (Airflow DAG: build_dw_sector)
  ┌─────────────────────────────────┐
  │  DW  (PostgreSQL)               │  Sector 집계, Dimension 관리
  │  dim_sector / dim_ticker        │
  │  fact_sector_daily              │
  └─────────────────────────────────┘
            │
            ▼  (Airflow DAG: build_mart_daily_index)
  ┌─────────────────────────────────┐
  │  Data Mart  (PostgreSQL)        │  Key Index, LLM 트렌드
  │  mart_daily_key_index           │
  │  mart_sector_trend_analysis     │  ← LLM Agent 결과
  └─────────────────────────────────┘
            │
     ┌──────┴──────┐
     ▼             ▼
 [FastAPI]    [LLM Agent]
 REST API     (LangChain + GPT-4o)
     │             │ Sector 트렌드 생성
     ▼             ▼
 [Frontend]   [mart_sector_trend_analysis 적재]
 React SPA
```

---

## 레이어 정의

| 레이어 | 적재 단위 | 갱신 주기 | 보존 기간 |
|--------|----------|----------|----------|
| Raw | Parquet 파일 / 거래일 | 장 마감 후 1회 | 5년 |
| ODS | 종목별 일별 행 | 거래일 1회 | 3년 |
| DW | Sector 집계 행 | 거래일 1회 | 3년 |
| Mart — Key Index | Sector × 일별 지표 | 거래일 1회 | 2년 |
| Mart — Trend Analysis | Sector × 일별 LLM 분석 | 거래일 1회 (Agent 실행 후) | 2년 |

---

## 스펙 문서 목록

| 파일 | 설명 |
|------|------|
| `overview.md` | 서비스 개요 (현재 파일) |
| `business_spec.md` | 비즈니스 스펙 (배경·시나리오·기능요건·성공기준) |
| `software_architecture.md` | 소프트웨어 아키텍처 (백엔드·인증·캐싱·보안·배포·관측성) |
| `spec_gap_analysis.md` | 스펙 완성도 분석 — 개발 착수 전 필요한 추가 항목 |
| `schema.md` | 전 레이어 테이블 스키마 정의 |
| `data_pipeline.md` | Airflow DAG 설계·수집·ODS·DW·Mart 상세 |
| `agent_trend.md` | LLM Agent Sector 트렌드 분석 스펙 |
| `frontend.md` | 화면 구성·API 계약·UI/UX 스펙 |

---

## 미결 기술 과제

- [ ] 주가 API 공급사 확정 (Alpha Vantage vs yfinance vs Polygon.io)
- [ ] Raw 스토리지 확정 (로컬 Parquet vs S3)
- [ ] LLM 모델 및 비용 한도 확정
- [ ] 프론트엔드 차트 라이브러리 선택
