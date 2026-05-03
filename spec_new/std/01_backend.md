# STD-01 — 백엔드 구현 표준

> 전체 상세: [`detail/backend_fastapi.md`](./detail/backend_fastapi.md), [`detail/backend_springboot.md`](./detail/backend_springboot.md)

---

## 계층 구조 (공통)

```
Router/Controller  ← HTTP 요청·응답, 입력 검증만
      ↓
Service            ← 비즈니스 로직, 트랜잭션 경계
      ↓
Repository/Client  ← DB 접근 또는 외부 API 호출
```

**규칙**: 계층을 건너뛰지 않는다. Controller에서 DB 직접 접근 금지.

---

## FastAPI 핵심 패턴

### 라우터
```python
@router.post("/v1/analyses", response_model=AnalysisResponse, status_code=201)
async def create_analysis(
    body: AnalysisRequest,          # Pydantic 자동 검증
    service: AnalysisService = Depends(),
):
    return await service.run(body.name)
```

### 에러 처리
```python
# 전역 핸들러 등록
@app.exception_handler(AppException)
async def handler(_, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"code": exc.code, "message": exc.message}},
    )
```

### 비동기 병렬 호출
```python
linkedin_data, tweets = await asyncio.gather(
    linkedin_client.fetch(url),
    twitter_client.fetch_mock(username),
)
```

---

## Spring Boot 핵심 패턴

### 컨트롤러
```java
@PostMapping("/v1/analyses")
public ResponseEntity<AnalysisResponse> create(
    @Valid @RequestBody AnalysisRequest request) {
    return ResponseEntity.status(201).body(service.run(request.name()));
}
```

### 에러 처리
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(AppException.class)
    public ResponseEntity<ErrorResponse> handle(AppException ex) {
        return ResponseEntity.status(ex.getStatusCode())
            .body(new ErrorResponse(ex.getCode(), ex.getMessage()));
    }
}
```

### Virtual Threads (Java 21)
```yaml
spring.threads.virtual.enabled: true  # application.yml
```

---

## BDD 테스트 연결 포인트

| Step | 구현 위치 |
|------|---------|
| `When API를 호출하면` | TestClient / TestRestTemplate → Router |
| `Then 응답이 반환된다` | response_model / ResponseEntity 검증 |
| `Given 데이터가 있다` | Service→Repository → TestDB Fixture |

---

## 외부 API 연동 필수 패턴

```python
# timeout + retry 반드시 설정
client = httpx.AsyncClient(
    timeout=httpx.Timeout(connect=5.0, read=30.0)
)

@retry(stop=stop_after_attempt(3),
       wait=wait_exponential(min=1, max=8))
async def fetch(url: str): ...
```
