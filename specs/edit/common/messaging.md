# Common Spec — 메시징 및 이벤트 드리븐 아키텍처

---

## 메시징 도입 목적

| 동기식 문제 | 메시징으로 해결 |
|-----------|--------------|
| LLM 호출(~30s) 동안 HTTP 연결 유지 | 비동기 처리 후 결과 폴링/웹소켓 전달 |
| 외부 API 장애 시 요청 유실 | 큐에 적재 → 장애 복구 후 재처리 |
| 서비스 간 강결합 | 이벤트 발행/구독으로 결합 제거 |
| 순간 트래픽 급증 | 큐가 버퍼 역할 → 백엔드 보호 |

---

## 메시징 시스템 비교

| 항목 | Kafka | RabbitMQ | Redis Streams |
|------|-------|----------|--------------|
| 모델 | 로그 기반 Pub/Sub | 메시지 큐 + Pub/Sub | 로그 기반 스트림 |
| 보존 | 설정 기간 보존 (재처리 가능) | 소비 후 삭제 (기본) | 설정 기간 보존 |
| 처리량 | 매우 높음 (수백만/초) | 높음 (수만/초) | 중간 |
| 순서 보장 | 파티션 내 보장 | 큐 내 보장 | 스트림 내 보장 |
| Consumer Group | 지원 | 지원 | 지원 |
| 적합한 상황 | 대용량 이벤트 스트리밍, 감사 로그 | 태스크 큐, RPC 패턴 | Redis 이미 사용 중, 소규모 |

---

## 이벤트 설계 원칙

### 이벤트 구조

```json
{
  "event_id":   "uuid-v4",
  "event_type": "analysis.requested",
  "version":    "1.0",
  "timestamp":  "2026-05-03T10:00:00Z",
  "source":     "ice-breaker-api",
  "correlation_id": "request-trace-id",
  "payload": {
    "name": "Harrison Chase",
    "requested_by": "user-uuid"
  }
}
```

| 필드 | 설명 |
|------|------|
| `event_id` | 중복 처리 방지용 멱등성 키 |
| `event_type` | `{도메인}.{동사}.{상태}` 형식 |
| `version` | 스키마 버전 — 하위 호환성 관리 |
| `correlation_id` | 분산 트레이싱 연결 |

### 이벤트 타입 네이밍

```
{도메인}.{동사과거형}      ← 발생한 사실 (이벤트)
{도메인}.{동사원형}        ← 처리할 명령 (커맨드)

예:
  analysis.requested      ← 분석 요청됨 (이벤트)
  analysis.completed      ← 분석 완료됨 (이벤트)
  analysis.failed         ← 분석 실패함 (이벤트)
  profile.fetch           ← 프로필 조회 명령
```

---

## Kafka 토픽 설계

```
토픽 네이밍: {서비스}.{도메인}.{이벤트타입}

ice-breaker.analysis.requests     ← 분석 요청 수신
ice-breaker.analysis.results      ← 분석 결과 발행
ice-breaker.profile.snapshots     ← 프로필 스냅샷 저장
ice-breaker.audit.events          ← 감사 이벤트 (보존 기간 1년)
```

### 파티션 전략

| 기준 | 파티션 키 | 효과 |
|------|---------|------|
| 사용자별 순서 보장 | `user_id` | 동일 사용자 요청 순서 보장 |
| 부하 분산 | `random` / `round-robin` | 처리량 극대화 |
| 도메인 집약 | `person_name` 해시 | 동일 인물 이벤트 집약 |

```python
# 파티션 키 지정 (Python kafka-python)
producer.send(
    topic="ice-breaker.analysis.requests",
    key=b"harrison-chase",         # 파티션 키 — 동일 인물 순서 보장
    value=event_json.encode(),
)
```

---

## Consumer Group 패턴

```
토픽: ice-breaker.analysis.requests
    │
    ├─ Consumer Group: analysis-workers    ← 분석 처리 (경쟁적 소비)
    │       ├─ worker-1
    │       ├─ worker-2
    │       └─ worker-3
    │
    └─ Consumer Group: audit-logger        ← 감사 로그 (독립적 소비)
            └─ logger-1
```

- 같은 그룹 내 Consumer는 파티션을 나눠 처리 (경쟁적 소비)
- 다른 그룹은 동일 메시지를 각자 처리 (독립적 소비)

```python
# FastAPI + aiokafka Consumer
from aiokafka import AIOKafkaConsumer
import asyncio, json

async def consume_analysis_requests():
    consumer = AIOKafkaConsumer(
        "ice-breaker.analysis.requests",
        bootstrap_servers="kafka:9092",
        group_id="analysis-workers",
        auto_offset_reset="earliest",
        enable_auto_commit=False,        # 수동 커밋 — 처리 후 커밋
    )
    await consumer.start()
    try:
        async for msg in consumer:
            event = json.loads(msg.value)
            await process_analysis(event)
            await consumer.commit()       # 처리 성공 후 오프셋 커밋
    finally:
        await consumer.stop()
```

---

## Dead Letter Queue (DLQ)

```
정상 흐름:
  requests 토픽 → Consumer → 처리 성공 → 커밋

실패 흐름:
  requests 토픽 → Consumer → 처리 실패
      └─ 재시도 1회 (즉시)
      └─ 재시도 2회 (1분 후)
      └─ 재시도 3회 (5분 후)
      └─ 실패 → DLQ 토픽으로 이동
                  └─ 알림 발송 + 수동 조사
```

```python
# DLQ 처리 패턴
MAX_RETRIES = 3

async def process_with_retry(event: dict, attempt: int = 0):
    try:
        await process_analysis(event)
    except Exception as e:
        if attempt < MAX_RETRIES:
            delay = 60 * (2 ** attempt)       # 1분, 2분, 4분
            await asyncio.sleep(delay)
            await process_with_retry(event, attempt + 1)
        else:
            # DLQ로 이동
            await producer.send(
                "ice-breaker.analysis.requests.dlq",
                value=json.dumps({
                    **event,
                    "failure_reason": str(e),
                    "failed_at": datetime.utcnow().isoformat(),
                    "attempts": attempt + 1,
                }).encode()
            )
            logger.error("dlq_moved", event_id=event["event_id"], error=str(e))
```

---

## Outbox 패턴 (이벤트 유실 방지)

트랜잭션 DB 저장과 이벤트 발행을 원자적으로 처리.

```
문제: DB 저장 성공 → 카프카 발행 실패 → 이벤트 유실

해결 (Outbox):
  1. DB 저장 + outbox 테이블 INSERT (같은 트랜잭션)
  2. Outbox Poller가 미발행 이벤트를 주기적으로 카프카에 발행
  3. 발행 성공 후 outbox 레코드 published_at 업데이트
```

```sql
CREATE TABLE outbox_events (
    id           BIGSERIAL PRIMARY KEY,
    event_id     UUID NOT NULL UNIQUE,
    event_type   VARCHAR(100) NOT NULL,
    topic        VARCHAR(200) NOT NULL,
    payload      JSONB NOT NULL,
    created_at   TIMESTAMP NOT NULL DEFAULT NOW(),
    published_at TIMESTAMP           -- NULL = 미발행
);

CREATE INDEX idx_outbox_unpublished ON outbox_events (created_at)
    WHERE published_at IS NULL;
```

---

## 멱등성 처리

네트워크 재시도로 인한 중복 메시지를 처리.

```python
# Redis로 처리된 event_id 기록
async def idempotent_process(event: dict):
    event_id = event["event_id"]
    lock_key = f"processed:{event_id}"

    # 이미 처리된 이벤트인지 확인
    if await redis.exists(lock_key):
        logger.info("duplicate_event_skipped", event_id=event_id)
        return

    # 처리
    await process_analysis(event)

    # 처리 완료 기록 (TTL = 메시지 보존 기간보다 길게)
    await redis.setex(lock_key, timedelta(days=8), "1")
```

---

## 비동기 처리 응답 패턴

LLM 처리처럼 응답이 긴 경우 클라이언트에 결과 전달 방식.

| 패턴 | 흐름 | 적합한 상황 |
|------|------|------------|
| 폴링 | 클라이언트가 주기적으로 상태 조회 | 단순, 추가 인프라 불필요 |
| WebSocket | 완료 시 서버가 클라이언트에 push | 실시간 피드백 필요 |
| SSE (Server-Sent Events) | 단방향 서버 push 스트림 | 단순 단방향 알림 |
| Webhook | 완료 시 클라이언트 URL 호출 | 서버 간 통신 |

```python
# FastAPI SSE 예시
from fastapi.responses import StreamingResponse

@router.post("/v1/process/stream")
async def process_stream(body: ProcessRequest):
    async def event_generator():
        yield "data: {\"status\": \"started\"}\n\n"
        result = await service.run(body.name)
        yield f"data: {result.model_dump_json()}\n\n"
        yield "data: {\"status\": \"done\"}\n\n"

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream"
    )
```

---

## 미결 기술 과제

- [ ] Kafka vs RabbitMQ 최종 선택 — 처리량·운영 복잡도 PoC 후 결정
- [ ] Kafka 클러스터 구성 — 브로커 수, 파티션 수, 복제 인수 결정
- [ ] Schema Registry 도입 — Confluent Schema Registry로 이벤트 스키마 버전 관리
- [ ] Outbox Poller 구현체 선택 — Debezium CDC vs 직접 구현
- [ ] DLQ 모니터링 알림 — 미처리 DLQ 메시지 수 임계값 정의
