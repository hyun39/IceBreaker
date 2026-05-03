# RAG Agent 스펙 — Ice Breaker

## 개요

> **현재 상태**: 미구현. 이 스펙은 RAG 기능 도입을 위한 설계 문서다.

현재 Ice Breaker는 매번 LinkedIn·Twitter를 실시간 스크래핑하고 LLM으로 분석한다.  
RAG Agent를 도입하면 **과거 분석 결과 및 외부 지식베이스**를 검색하여  
더 정확하고 빠른 응답을 제공할 수 있다.

---

## 도입 목적

| 문제 | RAG로 해결 |
|------|-----------|
| 같은 인물 재조회 시 매번 LLM 비용 발생 | 벡터 스토어에서 기존 결과 검색 후 재활용 |
| 아이스브레이커 품질이 일반적 수준 | 도메인별 예시 문서로 퀄리티 향상 |
| LinkedIn mock 데이터가 단일 인물로 고정 | 여러 샘플 프로필 문서를 지식베이스화 |

---

## 제안 아키텍처

```
name 입력
  → VectorStore 검색 (기존 분석 결과 있는지 확인)
      └─ 히트: 저장된 결과 반환 (LLM 호출 없음)
      └─ 미스: 기존 파이프라인 실행
               → 결과 생성 후 VectorStore에 저장
```

### 컴포넌트

| 컴포넌트 | 후보 기술 | 역할 |
|---------|----------|------|
| 임베딩 모델 | `text-embedding-3-small` | 텍스트 → 벡터 변환 |
| 벡터 스토어 | Chroma (로컬) / Pinecone (클라우드) | 벡터 저장·검색 |
| 문서 로더 | LangChain `JSONLoader` | LinkedIn JSON 적재 |
| 리트리버 | `VectorStoreRetriever` | 유사도 검색 |
| RAG 체인 | `RetrievalQA` 또는 LCEL | 검색 결과 + LLM 조합 |

---

## 지식베이스 구성안

### 문서 유형 1 — 과거 분석 결과
- 키: 인물 이름 (정규화)
- 내용: Summary, TopicOfInterest, IceBreaker 전체
- 갱신 주기: 7일 (LinkedIn 데이터 변동 고려)

### 문서 유형 2 — 아이스브레이커 예시 라이브러리
- 직군별 / 관심사별 고품질 대화 시작 예시
- 정적 문서로 관리 (`data/icebreaker_examples.json`)

---

## RAG 체인 설계 (LCEL)

```python
# 예시 구조 (미구현)
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

rag_chain = (
    {"context": retriever, "name": RunnablePassthrough()}
    | ice_breaker_rag_prompt
    | llm
    | ice_breaker_parser
)
```

---

## 데이터 흐름

```
인물 이름
  → normalize(name)  →  벡터 검색 (코사인 유사도 > 0.9)
       ↓ 히트                        ↓ 미스
  캐시 결과 반환          기존 파이프라인 실행
                              → 결과 저장 (embed + upsert)
                              → 결과 반환
```

---

## 구현 시 고려사항

| 항목 | 내용 |
|------|------|
| 캐시 무효화 | 저장 시 타임스탬프 기록, 7일 초과 시 재수집 |
| 동명이인 | 이름 외 회사/직함 메타데이터 함께 저장 |
| 벡터 스토어 선택 | 로컬 개발은 Chroma, 프로덕션은 Pinecone 검토 |
| 개인정보 | 벡터 스토어에 저장되는 데이터 범위 정책 수립 필요 |

---

## 구현 우선순위

- [ ] Phase 1: Chroma 로컬 벡터 스토어 세팅
- [ ] Phase 2: 분석 결과 자동 저장 파이프라인
- [ ] Phase 3: 캐시 히트 시 LLM 우회 로직
- [ ] Phase 4: 아이스브레이커 예시 라이브러리 적재

---

## 변경 이력

| 날짜 | 버전 | 내용 |
|------|------|------|
| 2026-05-03 | v1.0 | 초안 작성 (미구현 설계 문서) |
