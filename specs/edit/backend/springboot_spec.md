# 백엔드 스펙 (Spring Boot) — Ice Breaker

> **미구현 설계안**. Python 스택을 Java/Spring Boot로 재구현할 경우의 스펙.  
> LangChain4j를 사용하여 현행 LangChain Python과 동일한 AI 파이프라인을 구현.

---

## 기술 스택

| 항목 | 선택 | 비고 |
|------|------|------|
| 언어 | Java 21 | Virtual Threads (Loom) 활용 가능 |
| 웹 프레임워크 | Spring Boot 3.x | Spring MVC (또는 WebFlux) |
| LLM 오케스트레이션 | LangChain4j | Python LangChain의 Java 대응 |
| LLM 모델 | gpt-3.5-turbo (체인), gpt-4o-mini (에이전트) | 현행 유지 |
| 데이터 검증 | Bean Validation (Jakarta) | `@Valid`, `@NotBlank` |
| HTTP 클라이언트 | Spring WebClient (비동기) 또는 RestClient | 외부 API 호출 |
| 빌드 도구 | Gradle (Kotlin DSL) | 또는 Maven |
| 패키지 관리 | Maven Central / Gradle |  |

---

## 제안 패키지 구조

```
com.icebreaker/
├── IceBreakerApplication.java
├── controller/
│   └── IceBreakerController.java     ← POST /process
├── service/
│   └── IceBreakerService.java        ← 핵심 파이프라인 오케스트레이션
├── agent/
│   ├── LinkedInLookupAgent.java      ← LinkedIn URL 탐색 ReAct 에이전트
│   └── TwitterLookupAgent.java       ← Twitter username 탐색 ReAct 에이전트
├── chain/
│   └── IceBreakerChains.java         ← 3개 LangChain4j 체인 정의
├── thirdparty/
│   ├── LinkedInClient.java           ← Scrapin.io 연동
│   └── TwitterClient.java            ← Tweepy 대신 Twitter4J 또는 직접 호출
├── dto/
│   ├── ProcessRequest.java           ← 요청 DTO
│   └── ProcessResponse.java          ← 응답 DTO
└── config/
    └── LangChain4jConfig.java        ← LLM 빈 설정
```

---

## API 엔드포인트

### `GET /`
- 응답: React 정적 파일 서빙 또는 `{"status": "ok"}`

### `POST /process`
- Content-Type: `application/json`
- 요청:
```json
{ "name": "Harrison Chase" }
```
- 응답:
```json
{
  "summaryAndFacts": {
    "summary": "...",
    "facts": ["...", "..."]
  },
  "interests": {
    "topicsOfInterest": ["...", "...", "..."]
  },
  "iceBreakers": {
    "iceBreakers": ["...", "..."]
  },
  "pictureUrl": "https://..."
}
```

> **네이밍 주의**: Java 컨벤션(camelCase) 적용. 기존 프론트엔드(`snake_case`)와 연동 시 `@JsonProperty` 또는 Jackson 설정으로 변환 필요.

- 에러 응답: `@ControllerAdvice` 글로벌 예외 핸들러로 통일

---

## 처리 흐름 (`IceBreakerService.java`)

```
name 입력
  → linkedInLookupAgent.lookup(name)        → linkedInUrl
  → linkedInClient.scrapeProfile(url)       → LinkedInData
  → twitterLookupAgent.lookup(name)         → twitterUsername
  → twitterClient.fetchTweetsMock(user)     → List<Tweet>
  → chains.summaryChain(data, tweets)       → Summary
  → chains.interestsChain(data, tweets)     → TopicOfInterest
  → chains.iceBreakerChain(data, tweets)    → IceBreaker
  → return ProcessResponse(...)
```

---

## LangChain4j 체인 구성

LangChain4j의 `AiServices` 인터페이스 기반으로 현행 체인을 재현.

```java
// 예시 구조
@AiService
interface SummaryAiService {
    @SystemMessage("...")
    @UserMessage("Given info: {{information}} and tweets: {{tweets}}, create a summary and 2 facts.")
    Summary summarize(String information, String tweets);
}
```

| 서비스 | 모델 | temperature | 비고 |
|--------|------|-------------|------|
| SummaryAiService | gpt-3.5-turbo | 0 | |
| InterestsAiService | gpt-3.5-turbo | 0 | |
| IceBreakerAiService | gpt-3.5-turbo | 1 | |

---

## ReAct 에이전트 구성

LangChain4j `Agent` + `Tool` 어노테이션 사용.

```java
// TavilySearchTool.java
public class TavilySearchTool {
    @Tool("Searches the web for a LinkedIn or Twitter profile URL")
    public String search(String query) {
        // TavilySearch HTTP 호출
    }
}

// LinkedInLookupAgent.java
AgentExecutor agent = AiServices.builder(LinkedInLookupAgent.class)
    .chatLanguageModel(model)
    .tools(new TavilySearchTool())
    .build();
```

---

## 데이터 모델 (DTO)

```java
// Python Pydantic → Java record
public record Summary(String summary, List<String> facts) {}
public record TopicOfInterest(List<String> topicsOfInterest) {}
public record IceBreaker(List<String> iceBreakers) {}

public record ProcessResponse(
    Summary summaryAndFacts,
    TopicOfInterest interests,
    IceBreaker iceBreakers,
    String pictureUrl
) {}
```

---

## 외부 서비스 연동

### LinkedIn (`LinkedInClient.java`)
- Mock: 정적 JSON 파일 클래스패스 로드 (`resources/mock/linkedin.json`)
- Real: `RestClient`로 Scrapin.io API 호출

### Twitter (`TwitterClient.java`)
- Mock: 정적 JSON 파일 클래스패스 로드
- Real: Twitter4J 또는 Spring WebClient로 Twitter API v2 직접 호출
- Python의 `tweepy` 의존성 제거 → 환경변수 5개는 동일하게 필요

---

## 환경변수 / 설정 (`application.yml`)

```yaml
openai:
  api-key: ${OPENAI_API_KEY}

scrapin:
  api-key: ${SCRAPIN_API_KEY}

tavily:
  api-key: ${TAVILY_API_KEY}

twitter:
  bearer-token: ${TWITTER_BEARER_TOKEN}
  api-key: ${TWITTER_API_KEY}
  api-key-secret: ${TWITTER_API_KEY_SECRET}
  access-token: ${TWITTER_ACCESS_TOKEN}
  access-token-secret: ${TWITTER_ACCESS_TOKEN_SECRET}

langchain4j:
  tracing:
    enabled: ${LANGCHAIN_TRACING_V2:false}
```

---

## Python Flask 대비 차이점

| 항목 | Python Flask (현행) | Spring Boot |
|------|-------------------|-------------|
| 언어 | Python 3.x | Java 21 |
| LLM 라이브러리 | LangChain | LangChain4j |
| 데이터 검증 | Pydantic | Bean Validation + record |
| 비동기 | 제한적 | Virtual Threads / WebFlux |
| 의존성 주입 | 없음 (수동) | Spring IoC 컨테이너 |
| 설정 관리 | `.env` + python-dotenv | `application.yml` + Spring profiles |
| API 문서 | 없음 | SpringDoc OpenAPI (`/swagger-ui`) |
| 배포 단위 | Python 런타임 | JAR (fat jar) |
| 테스트 | pytest | JUnit 5 + Mockito |

---

## 미결 과제

- [ ] LangChain4j 버전 확정 및 ReAct 에이전트 호환성 검증
- [ ] JSON 네이밍 전략 결정 (`camelCase` vs `snake_case` — 프론트엔드 호환)
- [ ] Twitter 연동 라이브러리 선택 (Twitter4J vs 직접 WebClient)
- [ ] Mock/Real 프로파일 분리 (`@Profile("mock")`, `@Profile("prod")`)
- [ ] 응답 캐싱 전략 — Spring Cache + Redis 검토
- [ ] CORS 설정 (`@CrossOrigin` 또는 `WebMvcConfigurer`)

---

## 변경 이력

| 날짜 | 버전 | 내용 |
|------|------|------|
| 2026-05-03 | v1.0 | 초안 작성 (api_spec.md에서 분리, 미구현 설계안) |
