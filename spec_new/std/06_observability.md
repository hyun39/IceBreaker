# STD-06 — 관찰성 (OTel + OpenSearch)

> 전체 상세: [`detail/observability_otel_opensearch.md`](./detail/observability_otel_opensearch.md)

---

## 계측 설정 (FastAPI)

```python
# main.py
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor

def setup_otel(app):
    provider = TracerProvider(resource=Resource.create({
        SERVICE_NAME: "ice-breaker-api",
        DEPLOYMENT_ENVIRONMENT: settings.env,
    }))
    provider.add_span_processor(BatchSpanProcessor(
        OTLPSpanExporter(endpoint=settings.otel_endpoint)
    ))
    trace.set_tracer_provider(provider)
    FastAPIInstrumentor.instrument_app(app)   # HTTP 자동 계측
    HTTPXClientInstrumentor().instrument()    # 외부 API 자동 계측
```

---

## 수동 Span (LLM 체인 추적)

```python
tracer = trace.get_tracer(__name__)

async def run_summary_chain(data):
    with tracer.start_as_current_span("summary_chain") as span:
        span.set_attribute("llm.model", "gpt-3.5-turbo")
        result = await chain.ainvoke(data)
        span.set_attribute("llm.output_tokens",
                           result.usage.completion_tokens)
        return result
```

---

## 구조화 로그 필수 형식

```json
{
  "timestamp": "ISO8601",
  "level":     "INFO",
  "service":   "ice-breaker-api",
  "trace_id":  "4bf92f35...",
  "span_id":   "00f067aa...",
  "message":   "external_api_call_completed",
  "attributes": { "api": "scrapin", "duration_ms": 342 }
}
```

**`trace_id`와 `span_id`는 모든 로그에 포함** — OTel 미들웨어로 자동 주입.

---

## structlog 설정 (FastAPI)

```python
structlog.configure(processors=[
    structlog.contextvars.merge_contextvars,   # trace_id 자동 포함
    structlog.processors.add_log_level,
    structlog.processors.TimeStamper(fmt="iso"),
    structlog.processors.JSONRenderer(),
])

# PII 마스킹 (필수)
def mask_pii(_, __, event_dict):
    event_dict.pop("name", None)
    event_dict.pop("linkedin_url", None)
    return event_dict
```

---

## OpenSearch 인덱스 규칙

```
인덱스명: logs-{service}-YYYY.MM.DD  (ILM 일별 롤링)
보존: Hot 7일 → Warm 30일 → Cold 60일 → Delete 90일
```

---

## BDD 테스트에서 관찰성 검증

```python
# BDD에서 로그 출력 검증 (선택적)
@then("요청이 감사 로그에 기록된다")
def check_audit_log(caplog):
    assert any(
        "AUTH_SUCCESS" in record.message
        for record in caplog.records
    )
```
