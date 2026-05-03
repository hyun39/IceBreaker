# Common Spec — 캐싱 전략

---

## 캐싱 레이어 구조

```
[클라이언트]
    │  브라우저 캐시 (Cache-Control)
    ▼
[CDN]                       ← 정적 파일, 공개 API 응답
    │  CDN Miss 시
    ▼
[API Gateway]               ← 응답 캐시 (공개 엔드포인트)
    │  Cache Miss 시
    ▼
[애플리케이션 (Redis)]       ← LLM 결과, 세션, 자주 조회 데이터
    │  Cache Miss 시
    ▼
[데이터베이스]               ← DB 쿼리 결과 캐시 (선택적)
```

---

## Redis 데이터 구조별 사용 패턴

| 자료구조 | 사용 용도 | 명령 예시 |
|---------|----------|----------|
| `String` | 단순 값 캐시 (LLM 결과, 토큰) | `SET`, `GET`, `SETEX` |
| `Hash` | 객체 부분 갱신 (사용자 프로필) | `HSET`, `HGET`, `HGETALL` |
| `List` | 최근 검색 이력 큐 | `LPUSH`, `LRANGE`, `LTRIM` |
| `Set` | 중복 제거 집합 (태그, 권한) | `SADD`, `SMEMBERS`, `SISMEMBER` |
| `Sorted Set` | 순위·점수 기반 (인기 검색어) | `ZADD`, `ZRANGE`, `ZRANGEBYSCORE` |
| `JSON` (RedisJSON) | 복잡한 구조 부분 조회 | `JSON.SET`, `JSON.GET` |

---

## 캐시 키 설계

### 네이밍 컨벤션

```
{service}:{version}:{entity}:{identifier}[:{sub-key}]

예시:
  icebreaker:v1:analysis:user-harrison-chase
  icebreaker:v1:profile:linkedin:user-harrison-chase
  icebreaker:v1:search-history:user-uuid-1234
  icebreaker:v1:rate-limit:ip:192.168.1.1
```

| 원칙 | 내용 |
|------|------|
| 네임스페이스 | 서비스명으로 시작 — 멀티 테넌트 충돌 방지 |
| 버전 포함 | 스키마 변경 시 `v1` → `v2`로 키 무효화 |
| 정규화 | `user-harrison-chase` — 소문자·공백→하이픈 |
| 길이 제한 | 256바이트 이하 권장 |

---

## TTL 전략

| 캐시 대상 | TTL | 갱신 트리거 |
|----------|-----|-----------|
| LLM 분석 결과 | 7일 | 명시적 갱신 요청 또는 만료 후 재생성 |
| LinkedIn 프로필 스냅샷 | 7일 | 만료 후 재수집 |
| Twitter 데이터 | 1일 | 만료 후 재수집 |
| 사용자 세션 토큰 | 5분 (Access) | 갱신 시 TTL 연장 |
| Rate Limit 카운터 | 1분 / 1시간 | 윈도우 기반 자동 만료 |
| API Gateway 응답 | 60초 | 공개 엔드포인트만 |

---

## 캐시 패턴

### Cache-Aside (Lazy Loading) — 권장

```python
# app/services/ice_breaker.py
async def get_analysis(self, name: str) -> ProcessResponse:
    cache_key = f"icebreaker:v1:analysis:{normalize(name)}"

    # 1. 캐시 조회
    cached = await redis.get(cache_key)
    if cached:
        return ProcessResponse.model_validate_json(cached)

    # 2. Cache Miss → 실제 처리
    result = await self._run_pipeline(name)

    # 3. 캐시 저장
    await redis.setex(
        cache_key,
        timedelta(days=7),
        result.model_dump_json(),
    )
    return result
```

### Write-Through — DB 쓰기와 캐시 동시 갱신

```python
async def save_analysis(self, name: str, result: ProcessResponse):
    # DB 저장
    await db.save(result)
    # 캐시 동기화
    cache_key = f"icebreaker:v1:analysis:{normalize(name)}"
    await redis.setex(cache_key, timedelta(days=7), result.model_dump_json())
```

### Read-Through — 캐시 레이어가 DB 조회 담당

```python
# Spring Boot — @Cacheable
@Service
class IceBreakerService {

    @Cacheable(value = "analysis", key = "#name.toLowerCase().replaceAll(' ', '-')")
    fun getAnalysis(name: String): ProcessResponse {
        return runPipeline(name)    // 캐시 미스 시만 실행
    }

    @CacheEvict(value = "analysis", key = "#name.toLowerCase().replaceAll(' ', '-')")
    fun invalidateAnalysis(name: String) {}
}
```

---

## Cache Stampede (Dog-pile) 방지

캐시 만료 시 다수 요청이 동시에 DB/LLM을 호출하는 문제.

```python
# 분산 락(Redlock)으로 1개 요청만 생성 허용
import redis.asyncio as redis
from redis.asyncio.lock import Lock

async def get_analysis_safe(self, name: str) -> ProcessResponse:
    cache_key = f"icebreaker:v1:analysis:{normalize(name)}"
    lock_key  = f"{cache_key}:lock"

    cached = await redis.get(cache_key)
    if cached:
        return ProcessResponse.model_validate_json(cached)

    async with Lock(redis, lock_key, timeout=60, blocking_timeout=30):
        # 락 획득 후 재확인 (다른 요청이 이미 생성했을 수 있음)
        cached = await redis.get(cache_key)
        if cached:
            return ProcessResponse.model_validate_json(cached)

        result = await self._run_pipeline(name)
        await redis.setex(cache_key, timedelta(days=7), result.model_dump_json())
        return result
```

---

## 캐시 무효화 전략

| 방식 | 구현 | 적합한 상황 |
|------|------|------------|
| TTL 만료 | `SETEX` 자동 만료 | 데이터 변경이 드물고 일정 수준 staleness 허용 |
| 명시적 삭제 | `DEL key` | 사용자가 직접 갱신 요청 |
| 패턴 삭제 | `SCAN + DEL icebreaker:v1:analysis:*` | 스키마 변경 시 전체 무효화 |
| 버전 증가 | 키의 `v1` → `v2` 변경 | 대규모 무효화, 롤백 가능 |

```python
# 패턴 삭제 — KEYS 금지 (운영 성능 영향), SCAN 사용
async def invalidate_all_analysis(redis_client):
    cursor = 0
    pattern = "icebreaker:v1:analysis:*"
    while True:
        cursor, keys = await redis_client.scan(cursor, match=pattern, count=100)
        if keys:
            await redis_client.delete(*keys)
        if cursor == 0:
            break
```

---

## HTTP 캐시 헤더

```python
# FastAPI — 응답 캐시 헤더
from fastapi import Response

@router.get("/v1/public/trending")
async def get_trending(response: Response):
    response.headers["Cache-Control"] = "public, max-age=60, stale-while-revalidate=30"
    return data

# 개인화 응답 — 캐시 금지
@router.post("/v1/process")
async def process(response: Response, ...):
    response.headers["Cache-Control"] = "no-store"
    return result
```

| 경로 유형 | Cache-Control |
|----------|--------------|
| 정적 파일 | `public, max-age=31536000, immutable` |
| 공개 API 응답 | `public, max-age=60, stale-while-revalidate=30` |
| 개인화 응답 | `no-store` |
| 인증 필요 응답 | `private, no-cache` |

---

## Redis 운영 설정

```yaml
# redis.conf 주요 설정
maxmemory 2gb
maxmemory-policy allkeys-lru    # 메모리 초과 시 LRU 방식으로 키 제거
save 900 1                       # 영속성 — 900초 내 1회 변경 시 RDB 저장
appendonly yes                   # AOF 활성화 (내구성 향상)
```

| 메모리 정책 | 동작 | 권장 상황 |
|-----------|------|----------|
| `allkeys-lru` | 전체 키 중 LRU 제거 | 일반 캐시 |
| `volatile-lru` | TTL 있는 키 중 LRU 제거 | TTL 없는 키 보존 필요 시 |
| `allkeys-lfu` | 전체 키 중 LFU 제거 | 접근 빈도 편차 큰 경우 |
| `noeviction` | 제거 안 함 — 쓰기 오류 | 데이터 유실 절대 불가 |

---

## 미결 기술 과제

- [ ] Redis HA 구성 결정 — Sentinel vs Cluster vs Managed(ElastiCache/Upstash)
- [ ] LLM 응답 캐시 키 설계 — 동일 입력 판단 기준 (이름 정규화 방식)
- [ ] Cache Warming 전략 — 배포 후 콜드 스타트 대응
- [ ] 캐시 적중률 모니터링 — Redis `INFO stats`의 `keyspace_hits/misses` Prometheus 수집
- [ ] 멀티 리전 캐시 동기화 전략 (글로벌 확장 시)
