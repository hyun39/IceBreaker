# S&P 500 Daily 주가 분석 — Sector 트렌드 Agent 스펙

> 참조 공통 스펙: `common/agent.md`, `common/rag.md`, `common/observability_otel_opensearch.md`

---

## 목적

매 거래일 `mart_daily_key_index`에 적재된 Sector별 지표를 LLM에 입력하여  
**Sector 단위 트렌드 요약·주요 동인·리스크·투자 참고 의견**을 생성하고  
`mart_sector_trend_analysis` 테이블에 적재한다.

---

## Agent 패턴

| 항목 | 선택 | 근거 |
|------|------|------|
| 패턴 | Chain-of-Thought (CoT) | 툴 호출 불필요, Mart 데이터를 직접 주입하는 단순 분석 |
| LLM 모델 | `gpt-4o-mini` | 비용 최소화, 분석 품질 충분 |
| 병렬화 | Sector 11개 병렬 호출 | 각 Sector 독립 분석 |
| 출력 언어 | 한국어 + 영어 이중 출력 | 국내외 사용자 대응 |
| 출력 형식 | 구조화된 JSON (Pydantic 파싱) | DB 적재 자동화 |

---

## 입력 데이터 구성

**단일 Sector 호출 시 입력 컨텍스트:**

```python
context = {
    "sector": "Information Technology",
    "trade_date": "2026-05-01",
    "recent_30d": [
        {
            "trade_date": "2026-05-01",
            "sector_return_pct": 1.23,
            "advance_ratio": 78.5,
            "total_volume": 12345678900,
            "volume_vs_ma20": 1.15,
            "rsi_14": 62.4,
            "ma5_close": 4820.3,
            "ma20_close": 4790.1,
            "ma60_close": 4720.5,
            "sector_rank_by_return": 2,
            "top_gainer_ticker": "NVDA",
            "top_loser_ticker": "INTC",
        },
        # ... 최근 30 거래일 데이터
    ],
}
```

- 최근 **30 거래일** 지표를 컨텍스트로 제공해 단기 트렌드 추론 가능
- 전날 대비 급변 종목(`top_gainer`, `top_loser`)을 포함해 이벤트 단서 제공

---

## 프롬프트 설계

### 시스템 프롬프트 (`v1.0`)

```
You are a professional equity market analyst specializing in S&P 500 sector analysis.
You are given daily quantitative metrics for a specific GICS sector over the last 30 trading days.
Your job is to produce a structured analysis in JSON format.

Rules:
- Base your analysis ONLY on the provided data. Do not invent external news or events.
- Be concise and factual.
- Output MUST be valid JSON matching the schema below.
- Do not include explanation outside the JSON block.
```

### 사용자 프롬프트 템플릿 (한국어·영어 이중 출력)

```
Sector: {sector}
Analysis Date: {trade_date}
Data (last 30 trading days, newest first):
{json_data}

Produce a bilingual JSON analysis (Korean and English):
{{
  "trend_summary_ko": "string (2-3 sentences in Korean)",
  "trend_summary_en": "string (2-3 sentences in English)",
  "key_drivers": [
    {{"driver": "string", "detail_ko": "string", "detail_en": "string"}}
  ],
  "sentiment": "bullish | bearish | neutral",
  "risk_factors": [
    {{"factor": "string", "detail_ko": "string", "detail_en": "string"}}
  ],
  "recommendation_ko": "string (1-2 sentences in Korean)",
  "recommendation_en": "string (1-2 sentences in English)"
}}
```

### 출력 검증 (Pydantic — 이중 출력)

```python
from pydantic import BaseModel
from typing import Literal

DISCLAIMER_KO = "본 분석은 투자 자문이 아니며 참고 목적으로만 활용하시기 바랍니다."
DISCLAIMER_EN = "This analysis is not investment advice and is for reference purposes only."

class DriverBilingual(BaseModel):
    driver: str
    detail_ko: str
    detail_en: str

class TrendAnalysis(BaseModel):
    trend_summary_ko: str
    trend_summary_en: str
    key_drivers: list[DriverBilingual]
    sentiment: Literal["bullish", "bearish", "neutral"]
    risk_factors: list[DriverBilingual]
    recommendation_ko: str
    recommendation_en: str
    disclaimer_ko: str = DISCLAIMER_KO
    disclaimer_en: str = DISCLAIMER_EN
```

---

## Airflow DAG 연동 (`run_agent_trend`)

```python
with DAG(
    dag_id="run_agent_trend",
    schedule="0 9 * * 1-5",
    catchup=False,
    max_active_runs=1,
    default_args={"retries": 2, "retry_delay": timedelta(minutes=10)},
) as dag:

    sectors = get_active_sectors()  # 11개 Sector

    with TaskGroup("agent_calls") as agent_calls:
        for sector in sectors:
            PythonOperator(
                task_id=f"analyze_{sector['sector_code']}",
                python_callable=run_sector_trend_agent,
                op_kwargs={
                    "sector_id": sector["sector_id"],
                    "sector_name": sector["sector_name"],
                    "trade_date": "{{ ds }}",
                    "lookback_days": 30,
                },
                execution_timeout=timedelta(minutes=3),
            )

    validate = PythonOperator(
        task_id="validate_all_sectors_written",
        python_callable=check_all_sectors_have_trend,  # 11개 모두 적재됐는지 확인
    )

    agent_calls >> validate
```

---

## 핵심 실행 함수

```python
from langchain.chat_models import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.output_parsers import PydanticOutputParser

def run_sector_trend_agent(sector_id, sector_name, trade_date, lookback_days=30):
    # 1. Mart 데이터 조회
    rows = query_mart_key_index(sector_id, trade_date, lookback_days)

    # 2. 프롬프트 구성
    parser = PydanticOutputParser(pydantic_object=TrendAnalysis)
    prompt = ChatPromptTemplate.from_messages([
        ("system", SYSTEM_PROMPT),
        ("human", USER_PROMPT_TEMPLATE),
    ])

    # 3. LLM 호출
    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    chain = prompt | llm | parser

    result: TrendAnalysis = chain.invoke({
        "sector": sector_name,
        "trade_date": trade_date,
        "json_data": json.dumps([r.dict() for r in rows], ensure_ascii=False),
    })

    # 4. Mart 적재
    upsert_trend_analysis(
        trade_date=trade_date,
        sector_id=sector_id,
        sector_name=sector_name,
        analysis=result,
        llm_model="gpt-4o",
        prompt_version="v1.0",
    )
```

---

## 안전 한계 및 에러 처리

| 항목 | 기준 |
|------|------|
| LLM timeout | 60초 / Sector |
| Pydantic 파싱 실패 | 2회 재시도 후 `trend_summary="분석 실패"`, sentinel 값 저장 |
| API rate limit | OpenAI Tier에 맞게 병렬 11개 호출 속도 조절 |
| 비거래일 | Airflow 센서로 실행 skip |
| 데이터 부족 | `lookback_days` 내 데이터 < 5일이면 skip + 경고 알림 |

---

## 비용 추정 (참고)

| 항목 | 값 |
|------|----|
| Sector 수 | 11개 |
| 입력 토큰 / Sector | 약 2,000 tokens (30일 데이터) |
| 출력 토큰 / Sector | 약 400 tokens |
| 일별 총 토큰 | 입력 22,000 + 출력 4,400 |
| gpt-4o-mini 단가 (참고) | 입력 $0.15/1M, 출력 $0.60/1M |
| **일별 예상 비용** | **약 $0.006/일** (월 약 $0.13) |
| 월 비용 한도 | **$20** 초과 시 DAG skip + Slack 알림 |

---

## 관측성

- LangSmith 또는 OTLP → 각 Sector 호출의 latency, token 사용량 추적
- `agent_run_id`를 `mart_sector_trend_analysis.agent_run_id`에 기록해 재현 가능
- 프롬프트 버전 관리: `prompt_version` 컬럼으로 A/B 추적

---

## 프롬프트 버전 이력

| 버전 | 변경 내용 | 적용일 |
|------|-----------|--------|
| v1.0 | 초기 구조화 JSON 출력 | 2026-05-01 |

---

## 미결 기술 과제

- [ ] `sentiment` 판별 정확도 평가 기준 정의
- [ ] 외부 경제 뉴스 컨텍스트 추가 여부 (RAG 연동, v2 검토)
- [x] 한국어 + 영어 이중 출력 확정
- [x] LLM 모델: gpt-4o-mini 확정
- [x] 면책문구 Pydantic 기본값으로 자동 포함
- [ ] 프롬프트 A/B 실험 프레임워크 구축
