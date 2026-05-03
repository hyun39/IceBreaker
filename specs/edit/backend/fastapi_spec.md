# 백엔드 스펙 (FastAPI) — Ice Breaker

> **현재 구현체 기준 전환 설계안**. Flask → FastAPI 전환 시 적용할 스펙.  
> LangChain·LLM·외부 서비스 연동 로직은 그대로 유지하고 웹 레이어만 교체.

---

## 기술 스택

| 항목 | 선택 | 비고 |
|------|------|------|
| 웹 프레임워크 | FastAPI | Uvicorn ASGI 서버 |
| LLM 오케스트레이션 | LangChain | 현행 유지 |
| LLM 모델 | gpt-3.5-turbo (체인), gpt-4o-mini (에이전트) | 현행 유지 |
| 데이터 검증 | Pydantic v2 | FastAPI 기본 내장 |
| 데이터 수집 | Scrapin.io (LinkedIn), Tweepy (Twitter) | 현행 유지 |
| 웹 검색 | Tavily | 현행 유지 |
| 패키지 관리 | pip / poetry | pipenv에서 전환 권장 |

---

## 제안 파일 구조

```
app/
├── main.py                  ← FastAPI 앱 진입점, 라우터 등록
├── routers/
│   └── icebreaker.py        ← POST /process 엔드포인트
├── services/
│   └── ice_breaker.py       ← 현행 ice_breaker.py 이식
├── schemas/
│   └── icebreaker.py        ← Pydantic 요청/응답 스키마
├── chains/
│   └── custom_chains.py     ← 현행 유지
├── agents/
│   ├── linkedin_lookup_agent.py
│   └── twitter_lookup_agent.py
└── third_parties/
    ├── linkedin.py
    └── twitter.py
```

---

## API 엔드포인트

### `GET /`
- 응답: `{"message": "Ice Breaker API"}`  
- React SPA 분리 배포 시 정적 서빙 불필요. 통합 시 `StaticFiles` 마운트

### `POST /process`
- 요청 본문: `application/x-www-form-urlencoded` 또는 `application/json`

**Form 방식 (현행 호환)**
```python
@router.post("/process")
async def process(name: str = Form(...)):
    ...
```

**JSON 방식 (React SPA 연동 권장)**
```python
class ProcessRequest(BaseModel):
    name: str

@router.post("/process")
async def process(body: ProcessRequest):
    ...
```

**응답 스키마**
```python
class SummaryAndFacts(BaseModel):
    summary: str
    facts: list[str]

class IceBreakers(BaseModel):
    ice_breakers: list[str]

class Interests(BaseModel):
    topics_of_interest: list[str]

class ProcessResponse(BaseModel):
    summary_and_facts: SummaryAndFacts
    interests: Interests
    ice_breakers: IceBreakers
    picture_url: str
```

**에러 응답**
```python
raise HTTPException(status_code=500, detail="처리 중 오류가 발생했습니다.")
```

---

## 처리 흐름 (`services/ice_breaker.py`)

```
name 입력
  → linkedin_lookup_agent(name)      → linkedin_url
  → scrape_linkedin_profile(url)     → linkedin_data (dict)
  → twitter_lookup_agent(name)       → twitter_username
  → scrape_user_tweets_mock(user)    → tweets (list)
  → summary_chain.invoke(...)        → Summary
  → interests_chain.invoke(...)      → TopicOfInterest
  → ice_breaker_chain.invoke(...)    → IceBreaker
  → return ProcessResponse(...)
```

비동기 처리: LangChain의 `ainvoke()` 사용 시 엔드포인트를 `async def`로 선언

---

## LangChain 체인 구성

현행(`chains/custom_chains.py`)과 동일하게 유지.  
비동기 전환 시 `ainvoke()` 호출로 교체.

| 체인 | 모델 | temperature | 비고 |
|------|------|-------------|------|
| summary_chain | gpt-3.5-turbo | 0 | `ainvoke` 전환 가능 |
| interests_chain | gpt-3.5-turbo | 0 | `ainvoke` 전환 가능 |
| ice_breaker_chain | gpt-3.5-turbo | 1 | `ainvoke` 전환 가능 |

---

## CORS 설정

React SPA와 다른 포트에서 개발할 경우 필수.

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],  # Vite 개발 서버
    allow_methods=["*"],
    allow_headers=["*"],
)
```

---

## 환경변수

Flask 현행과 동일. `.env` → `python-dotenv` 또는 `pydantic-settings`로 로드.

| 변수 | 필수 여부 | 용도 |
|------|----------|------|
| `OPENAI_API_KEY` | 필수 | LLM 호출 |
| `SCRAPIN_API_KEY` | mock 미사용 시 필수 | LinkedIn 스크래핑 |
| `TAVILY_API_KEY` | 필수 | 에이전트 웹 검색 |
| `TWITTER_BEARER_TOKEN` | 필수(임포트 시) | Tweepy 초기화 |
| `TWITTER_API_KEY` | 필수(임포트 시) | Tweepy 초기화 |
| `TWITTER_API_KEY_SECRET` | 필수(임포트 시) | Tweepy 초기화 |
| `TWITTER_ACCESS_TOKEN` | 필수(임포트 시) | Tweepy 초기화 |
| `TWITTER_ACCESS_TOKEN_SECRET` | 필수(임포트 시) | Tweepy 초기화 |
| `LANGCHAIN_TRACING_V2` | 선택 | LangSmith 트레이싱 |
| `LANGCHAIN_API_KEY` | 선택 | LangSmith |
| `LANGCHAIN_PROJECT` | 선택 | LangSmith 프로젝트명 |

---

## Flask 대비 차이점

| 항목 | Flask (현행) | FastAPI |
|------|------------|---------|
| 요청 검증 | 없음 (수동) | Pydantic 자동 검증 |
| 응답 직렬화 | `jsonify()` + `to_dict()` | `response_model` 자동 직렬화 |
| 비동기 | 미지원 (기본) | `async def` 네이티브 지원 |
| API 문서 | 없음 | `/docs` (Swagger), `/redoc` 자동 생성 |
| 타입 힌트 활용 | 선택적 | 필수 (프레임워크 핵심) |
| 에러 응답 | HTTP 500 (비통일) | `HTTPException` 통일 |

---

## 미결 과제

- [ ] 비동기 전환 — `chain.ainvoke()` 적용 범위 결정
- [ ] `pydantic-settings`로 환경변수 타입 안전 로드
- [ ] Twitter 실제 API 연동 (`scrape_user_tweets` 전환)
- [ ] 응답 캐싱 — `fastapi-cache2` 또는 Redis 검토
- [ ] 글로벌 예외 핸들러 (`@app.exception_handler`) 등록

---

## 변경 이력

| 날짜 | 버전 | 내용 |
|------|------|------|
| 2026-05-03 | v1.0 | 초안 작성 (api_spec.md에서 분리) |
