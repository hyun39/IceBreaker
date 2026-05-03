# Agentic Agent 스펙 — Ice Breaker

## 개요

사람 이름만으로 **LinkedIn URL**과 **Twitter username**을 자율 탐색하는 ReAct 에이전트 2종.  
LLM이 Tavily 검색 툴을 반복 호출하여 원하는 URL을 찾을 때까지 추론-행동 루프를 실행한다.

---

## 에이전트 목록

| 에이전트 | 파일 | 출력 |
|---------|------|------|
| LinkedIn Lookup | `agents/linkedin_lookup_agent.py` | LinkedIn 프로필 URL (string) |
| Twitter Lookup | `agents/twitter_lookup_agent.py` | Twitter username (string) |

---

## 공통 아키텍처 (ReAct 패턴)

```
name 입력
  → PromptTemplate.format_prompt(name_of_person=name)
  → AgentExecutor.invoke(input)
      └─ ReAct 루프:
           Thought → Action(Tavily 검색) → Observation → ...
           → Final Answer
  → result["output"] 반환
```

- **LLM**: `gpt-4o-mini`, `temperature=0`
- **ReAct 프롬프트**: LangChain Hub `hwchase17/react` (원격 pull)
- **툴**: `get_profile_url_tavily` (Tavily 검색, `tools/tools.py`)

---

## 에이전트별 프롬프트

### LinkedIn Lookup Agent
```
given the full name {name_of_person} I want you to get it me a link to their
Linkedin profile page. Your answer should contain only a URL
```
- 출력 제약: URL만 반환

### Twitter Lookup Agent
```
given the name {name_of_person} I want you to find a link to their Twitter/X
profile page, and extract from it their username.
In Your Final answer only the person's username
which is extracted from: https://x.com/USERNAME
```
- 출력 제약: username만 반환 (`@` 없는 문자열)

---

## 툴 구성 (`tools/tools.py`)

| 툴 이름 | 함수 | 설명 |
|---------|------|------|
| Crawl Google 4 linkedin profile page | `get_profile_url_tavily` | LinkedIn URL 검색 |
| Crawl Google 4 Twitter profile page | `get_profile_url_tavily` | Twitter URL 검색 |

- 두 에이전트 모두 동일한 `TavilySearch` 함수를 재사용
- `TAVILY_API_KEY` 환경변수 필수

---

## 실행 설정

| 항목 | 값 | 비고 |
|------|-----|------|
| verbose | True | 추론 단계 콘솔 출력 |
| max_iterations | 기본값 | 무한 루프 방지용 제한 설정 검토 필요 |
| handle_parsing_errors | 미설정 | 파싱 오류 시 비정상 종료 가능 |

---

## 현재 구현의 한계

| 항목 | 현황 | 개선 방향 |
|------|------|-----------|
| 동명이인 처리 | 없음 | 추가 컨텍스트(직책, 회사 등) 프롬프트에 포함 |
| 에러 핸들링 | 없음 | `handle_parsing_errors=True` 설정 |
| 탐색 실패 | 빈 문자열 또는 예외 | Fallback 전략 정의 필요 |
| 비용 | 툴 호출 횟수 무제한 | `max_iterations` 명시적 설정 |

---

## 미결 과제

- [ ] `AgentExecutor(handle_parsing_errors=True)` 적용
- [ ] `max_iterations=5` 등 상한 설정
- [ ] 탐색 실패 시 호출자에게 명확한 예외 전달
- [ ] 프롬프트에 추가 컨텍스트 필드 지원 (직책, 회사명 등)

---

## 변경 이력

| 날짜 | 버전 | 내용 |
|------|------|------|
| 2026-05-03 | v1.0 | 초안 작성 |
