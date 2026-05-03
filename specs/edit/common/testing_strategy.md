# Common Spec — 테스트 전략

---

## 테스트 피라미드

```
           ╱▔▔▔▔▔▔▔╲
          ╱  E2E(5%) ╲          느림, 비용 높음, 신뢰도 최고
         ╱─────────────╲
        ╱ Integration   ╲
       ╱    (25%)        ╲       중간 속도·비용
      ╱───────────────────╲
     ╱    Unit (70%)       ╲    빠름, 비용 낮음, 범위 좁음
    ╱───────────────────────╲
```

| 레이어 | 비율 | 실행 시점 | 실패 시 |
|--------|------|----------|--------|
| Unit | 70% | PR 생성 즉시 | PR 머지 차단 |
| Integration | 25% | PR 머지 전 | PR 머지 차단 |
| E2E | 5% | main 머지 후 / 배포 전 | 배포 차단 |
| Performance | 별도 | 릴리스 전 수동 | 배포 검토 |

---

## 단위 테스트

### 원칙

| 원칙 | 내용 |
|------|------|
| FIRST | Fast, Isolated, Repeatable, Self-validating, Timely |
| AAA | Arrange(준비) → Act(실행) → Assert(검증) 구조 |
| 단일 책임 | 테스트 1개 = 동작 1개 검증 |
| 외부 격리 | DB, 외부 API, LLM 모두 Mock |

### FastAPI (pytest)

```python
# tests/unit/test_ice_breaker_service.py
import pytest
from unittest.mock import AsyncMock, patch, MagicMock
from app.services.ice_breaker import IceBreakerService

@pytest.fixture
def service():
    return IceBreakerService(
        linkedin_client=AsyncMock(),
        twitter_client=AsyncMock(),
        summary_chain=MagicMock(),
    )

class TestIceBreakerService:

    async def test_run_returns_response(self, service):
        # Arrange
        service.linkedin_client.lookup_url.return_value = "https://linkedin.com/in/test"
        service.summary_chain.ainvoke.return_value = MagicMock(
            summary="Test summary", facts=["fact1", "fact2"]
        )
        # Act
        result = await service.run("Test User")
        # Assert
        assert result.summary_and_facts.summary == "Test summary"
        service.linkedin_client.lookup_url.assert_called_once_with("Test User")

    async def test_run_raises_on_lookup_failure(self, service):
        service.linkedin_client.lookup_url.side_effect = Exception("API 오류")
        with pytest.raises(Exception, match="API 오류"):
            await service.run("Test User")
```

### Spring Boot (JUnit 5 + Mockito)

```java
// src/test/java/com/icebreaker/service/IceBreakerServiceTest.java
@ExtendWith(MockitoExtension.class)
class IceBreakerServiceTest {

    @Mock LinkedInClient linkedInClient;
    @Mock TwitterClient twitterClient;
    @Mock SummaryChain summaryChain;
    @InjectMocks IceBreakerService service;

    @Test
    @DisplayName("정상 입력 시 ProcessResponse 반환")
    void run_withValidName_returnsResponse() {
        // Arrange
        given(linkedInClient.lookupUrl("Test User"))
            .willReturn("https://linkedin.com/in/test");
        given(summaryChain.invoke(any()))
            .willReturn(new Summary("Test summary", List.of("fact1")));
        // Act
        var result = service.run("Test User");
        // Assert
        assertThat(result.summaryAndFacts().summary()).isEqualTo("Test summary");
        then(linkedInClient).should().lookupUrl("Test User");
    }

    @Test
    @DisplayName("LinkedIn API 실패 시 ExternalApiException 발생")
    void run_whenLinkedInFails_throwsException() {
        given(linkedInClient.lookupUrl(any()))
            .willThrow(new ExternalApiException("API 오류"));
        assertThatThrownBy(() -> service.run("Test User"))
            .isInstanceOf(ExternalApiException.class);
    }
}
```

### 커버리지 목표

| 대상 | 목표 | 측정 도구 |
|------|------|----------|
| Service 레이어 | 90% 이상 | pytest-cov / JaCoCo |
| 도메인 로직 | 95% 이상 | |
| Controller | 80% 이상 | |
| 전체 | 80% 이상 | Codecov |

---

## 통합 테스트

### 원칙

- 실제 DB(PostgreSQL) 사용 — Mock DB 금지
- 외부 LLM·API는 Mock 처리 (비용·속도 이유)
- 테스트 간 DB 상태 격리 — 트랜잭션 롤백 또는 픽스처 초기화

### FastAPI + Testcontainers

```python
# tests/integration/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer
from fastapi.testclient import TestClient
from app.main import app
from app.core.database import get_db

@pytest.fixture(scope="session")
def postgres():
    with PostgresContainer("postgres:15") as pg:
        yield pg

@pytest.fixture(scope="function")
def client(postgres):
    # 의존성 오버라이드 — 테스트 DB 연결
    app.dependency_overrides[get_db] = lambda: test_db_session(postgres.get_connection_url())
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()

# tests/integration/test_process_api.py
def test_process_returns_200(client):
    response = client.post("/v1/process", json={"name": "Test User"})
    assert response.status_code == 200
    assert "summary_and_facts" in response.json()

def test_process_validates_empty_name(client):
    response = client.post("/v1/process", json={"name": ""})
    assert response.status_code == 422
```

### Spring Boot + @SpringBootTest

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@Testcontainers
@Transactional                          // 테스트 후 롤백
class ProcessControllerIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:15")
            .withDatabaseName("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
    }

    @Autowired TestRestTemplate restTemplate;

    @MockBean LinkedInClient linkedInClient;   // 외부 API만 Mock
    @MockBean TwitterClient twitterClient;

    @Test
    void process_withValidName_returns200() {
        given(linkedInClient.lookupUrl(any())).willReturn("https://linkedin.com/in/test");
        given(twitterClient.fetchMock(any())).willReturn(List.of());

        var response = restTemplate.postForEntity(
            "/v1/process",
            new ProcessRequest("Test User"),
            ProcessResponse.class
        );

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().summaryAndFacts()).isNotNull();
    }
}
```

---

## E2E 테스트

### 대상 시나리오

| 시나리오 | 우선순위 |
|---------|---------|
| 이름 입력 → 결과 화면 정상 표시 | 필수 |
| 잘못된 입력 → 에러 메시지 표시 | 필수 |
| 로그인 → 보호된 API 접근 | 필수 |
| 로딩 스피너 표시 → 결과 렌더링 | 권장 |

### Playwright

```typescript
// tests/e2e/process.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Ice Breaker 메인 플로우", () => {

  test("이름 입력 후 결과가 표시된다", async ({ page }) => {
    await page.goto("/");
    await page.fill('input[name="name"]', "Harrison Chase");
    await page.click('button[type="submit"]');

    // 로딩 스피너 표시 확인
    await expect(page.locator("#spinner")).toBeVisible();

    // 결과 영역 표시 확인 (최대 30초 대기)
    await expect(page.locator("#result")).toBeVisible({ timeout: 30_000 });
    await expect(page.locator("#summary")).not.toBeEmpty();
  });

  test("빈 이름 제출 시 에러가 표시된다", async ({ page }) => {
    await page.goto("/");
    await page.click('button[type="submit"]');
    await expect(page.locator(".error-banner")).toBeVisible();
  });
});
```

```typescript
// playwright.config.ts
export default {
  testDir: "./tests/e2e",
  timeout: 60_000,
  retries: process.env.CI ? 2 : 0,
  use: {
    baseURL: process.env.BASE_URL ?? "http://localhost:8000",
    trace: "retain-on-failure",
    screenshot: "only-on-failure",
  },
};
```

---

## 성능 테스트 (k6)

```javascript
// tests/performance/process_load.js
import http from "k6/http";
import { check, sleep } from "k6";
import { Rate } from "k6/metrics";

const errorRate = new Rate("errors");

export const options = {
  stages: [
    { duration: "1m",  target: 10  },   // 워밍업
    { duration: "3m",  target: 50  },   // 부하
    { duration: "1m",  target: 100 },   // 피크
    { duration: "1m",  target: 0   },   // 쿨다운
  ],
  thresholds: {
    http_req_duration: ["p(95)<30000"],  // p95 30초 이내 (LLM 포함)
    errors: ["rate<0.01"],               // 에러율 1% 미만
  },
};

export default function () {
  const res = http.post(
    `${__ENV.BASE_URL}/v1/process`,
    JSON.stringify({ name: "Harrison Chase" }),
    { headers: { "Content-Type": "application/json" } }
  );
  errorRate.add(res.status !== 200);
  check(res, { "status 200": (r) => r.status === 200 });
  sleep(1);
}
```

### 성능 기준

| 지표 | 목표값 |
|------|--------|
| p95 응답시간 | 30초 이내 (LLM 체인 포함) |
| p99 응답시간 | 60초 이내 |
| 에러율 | 1% 미만 |
| 동시 사용자 50명 | 서비스 정상 |

---

## CI 통합 매트릭스

```
PR 생성
  └─ Unit Test         → 실패 시 리뷰 차단
  └─ Lint / Type Check → 실패 시 리뷰 차단

PR 승인 후 머지
  └─ Integration Test  → 실패 시 머지 차단
  └─ Security Scan     → HIGH/CRITICAL 발견 시 차단

main 머지
  └─ E2E Test (dev 환경)  → 실패 시 staging 배포 차단

릴리스 전
  └─ Performance Test  → 수동 실행, 기준 미달 시 배포 검토
```

---

## 테스트 데이터 관리

| 방식 | 적합한 상황 |
|------|------------|
| Fixture 파일 (`fixtures/`) | 정적인 입력값, 예상 응답 |
| Factory 함수 | 동적 생성 필요한 모델 인스턴스 |
| Seed 스크립트 | 통합 테스트 DB 초기 데이터 |
| Testcontainers | 매 실행마다 깨끗한 DB 보장 |

```python
# tests/factories.py — Factory Boy 사용 예시
import factory
from app.schemas.process import ProcessResponse

class ProcessResponseFactory(factory.Factory):
    class Meta:
        model = ProcessResponse

    summary_and_facts = factory.SubFactory(SummaryAndFactsFactory)
    picture_url = factory.Faker("image_url")
```

---

## 미결 기술 과제

- [ ] 테스트 커버리지 게이트 수치 CI에 적용 (`--cov-fail-under=80`)
- [ ] 성능 테스트 실행 주기 확정 — 릴리스마다 vs 주간 정기 실행
- [ ] Snapshot 테스트 도입 여부 — API 응답 구조 회귀 감지
- [ ] Chaos Engineering 도입 시점 — 서비스 안정화 이후 검토
- [ ] 계약 테스트(Consumer-Driven Contract) — 마이크로서비스 분리 시 Pact 도입
