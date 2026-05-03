# 테이블 설계 스펙 — Ice Breaker

> RDBMS: PostgreSQL 15 기준. 타 DB 적용 시 하단 "DB 엔진별 타입 호환" 참조.

---

## 테이블 목록 및 역할

| 테이블 | 설명 | 정규화 수준 |
|--------|------|------------|
| `person` | 인물 마스터 — URL 탐색 결과 포함 | 3NF |
| `profile_snapshot` | 외부 API 원본 응답 버전 관리 | 2NF (raw_data 비정규화 의도적) |
| `analysis_result` | LLM 생성 결과 — 스냅샷 버전과 연결 | 3NF |
| `search_history` | 조회 이력 — append-only, 수정 없음 | 3NF |

---

## ER 다이어그램

```
person
  │ 1
  ├────< profile_snapshot   (person_id FK, CASCADE DELETE)
  │            │ 1
  │            └────< analysis_result (snapshot_id FK, RESTRICT DELETE)
  │ 1
  ├────< analysis_result    (person_id FK, CASCADE DELETE)
  │
  └────< search_history     (person_id FK, RESTRICT DELETE)
                │ N
                └──── analysis_result (analysis_result_id FK, SET NULL)
```

---

## 테이블 상세

### `person`

| 컬럼 | 타입 | 제약 | 설계 근거 |
|------|------|------|----------|
| `id` | BIGSERIAL | PK | 외부 노출 없음 — 내부 조인 키 전용 |
| `full_name` | VARCHAR(255) | NOT NULL | 입력 원문 보존 |
| `normalized_name` | VARCHAR(255) | NOT NULL, UNIQUE | 검색 키 — 소문자·공백 trim 적용 파생 컬럼 |
| `linkedin_url` | TEXT | NULLABLE | URL 길이 가변 → TEXT |
| `twitter_username` | VARCHAR(100) | NULLABLE | Twitter username 최대 15자, 여유 확보 |
| `profile_pic_url` | TEXT | NULLABLE | URL 길이 가변 → TEXT |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | linkedin_url·twitter_username 갱신 시 트리거 업데이트 |

```sql
CREATE TABLE person (
    id               BIGSERIAL PRIMARY KEY,
    full_name        VARCHAR(255) NOT NULL,
    normalized_name  VARCHAR(255) NOT NULL UNIQUE,
    linkedin_url     TEXT,
    twitter_username VARCHAR(100),
    profile_pic_url  TEXT,
    created_at       TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMP NOT NULL DEFAULT NOW()
);
```

---

### `profile_snapshot`

| 컬럼 | 타입 | 제약 | 설계 근거 |
|------|------|------|----------|
| `id` | BIGSERIAL | PK | |
| `person_id` | BIGINT | NOT NULL, FK → person.id CASCADE | 인물 삭제 시 스냅샷 함께 삭제 |
| `source` | VARCHAR(20) | NOT NULL, CHECK | `'linkedin'` \| `'twitter'` — ENUM 대신 CHECK 사용 (값 추가 용이) |
| `raw_data` | JSONB | NOT NULL | 외부 API 응답 스키마 유동적 → JSONB 비정규화 저장. GIN 인덱스 추후 적용 가능 |
| `is_mock` | BOOLEAN | NOT NULL, DEFAULT FALSE | 테스트 데이터 식별 — 운영 쿼리에서 필터링 |
| `fetched_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | |
| `expires_at` | TIMESTAMP | NULLABLE | NULL = 만료 없음. 파티셔닝 키 후보 |

```sql
CREATE TABLE profile_snapshot (
    id         BIGSERIAL PRIMARY KEY,
    person_id  BIGINT NOT NULL REFERENCES person(id) ON DELETE CASCADE,
    source     VARCHAR(20) NOT NULL CHECK (source IN ('linkedin', 'twitter')),
    raw_data   JSONB NOT NULL,
    is_mock    BOOLEAN NOT NULL DEFAULT FALSE,
    fetched_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP
);
```

---

### `analysis_result`

| 컬럼 | 타입 | 제약 | 설계 근거 |
|------|------|------|----------|
| `id` | BIGSERIAL | PK | |
| `person_id` | BIGINT | NOT NULL, FK → person.id CASCADE | 빠른 인물별 최신 결과 조회용 |
| `snapshot_id` | BIGINT | NOT NULL, FK → profile_snapshot.id RESTRICT | 스냅샷 삭제 전 결과 존재 여부 강제 확인 |
| `summary` | TEXT | NOT NULL | 길이 가변 → TEXT |
| `facts` | JSONB | NOT NULL | 배열 — `["fact1", "fact2"]` |
| `topics_of_interest` | JSONB | NOT NULL | 배열 — `["topic1", ...]` |
| `ice_breakers` | JSONB | NOT NULL | 배열 — `["ice1", "ice2"]` |
| `llm_model` | VARCHAR(50) | NOT NULL | 재현성 추적 — 모델 변경 시 결과 비교 가능 |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | |
| `expires_at` | TIMESTAMP | NULLABLE | profile_snapshot.expires_at 와 동기화 권장 |

```sql
CREATE TABLE analysis_result (
    id                  BIGSERIAL PRIMARY KEY,
    person_id           BIGINT NOT NULL REFERENCES person(id) ON DELETE CASCADE,
    snapshot_id         BIGINT NOT NULL REFERENCES profile_snapshot(id) ON DELETE RESTRICT,
    summary             TEXT NOT NULL,
    facts               JSONB NOT NULL,
    topics_of_interest  JSONB NOT NULL,
    ice_breakers        JSONB NOT NULL,
    llm_model           VARCHAR(50) NOT NULL,
    created_at          TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at          TIMESTAMP
);
```

---

### `search_history`

| 컬럼 | 타입 | 제약 | 설계 근거 |
|------|------|------|----------|
| `id` | BIGSERIAL | PK | |
| `person_id` | BIGINT | NOT NULL, FK → person.id RESTRICT | 인물 삭제 전 이력 처리 강제 |
| `analysis_result_id` | BIGINT | NULLABLE, FK → analysis_result.id SET NULL | 결과 삭제 시 이력 자체는 보존 |
| `queried_name` | VARCHAR(255) | NOT NULL | 입력 원문 — normalized_name과 분리 보존 |
| `cache_hit` | BOOLEAN | NOT NULL, DEFAULT FALSE | |
| `response_ms` | INTEGER | NULLABLE | 부호 없는 값이나 INT로 충분 (최대 ~2억ms) |
| `searched_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | 파티셔닝 키 — RANGE 파티셔닝 적용 시 기준 컬럼 |

```sql
CREATE TABLE search_history (
    id                  BIGSERIAL PRIMARY KEY,
    person_id           BIGINT NOT NULL REFERENCES person(id) ON DELETE RESTRICT,
    analysis_result_id  BIGINT REFERENCES analysis_result(id) ON DELETE SET NULL,
    queried_name        VARCHAR(255) NOT NULL,
    cache_hit           BOOLEAN NOT NULL DEFAULT FALSE,
    response_ms         INTEGER,
    searched_at         TIMESTAMP NOT NULL DEFAULT NOW()
);
```

---

## 참조 무결성 정책 요약

| 관계 | ON DELETE | 근거 |
|------|-----------|------|
| `profile_snapshot` → `person` | CASCADE | 인물 삭제 시 스냅샷 의미 없음 |
| `analysis_result` → `person` | CASCADE | 인물 삭제 시 결과 의미 없음 |
| `analysis_result` → `profile_snapshot` | RESTRICT | 스냅샷 단독 삭제 방지 — 결과 먼저 처리 강제 |
| `search_history` → `person` | RESTRICT | 이력은 인물보다 먼저 아카이빙 필요 |
| `search_history` → `analysis_result` | SET NULL | 결과 삭제 시 이력 행 자체는 보존 |

---

## 인덱스 전략

| 인덱스명 | 테이블 | 컬럼 | 종류 | 용도 |
|---------|--------|------|------|------|
| `idx_person_normalized_name` | `person` | `normalized_name` | B-Tree, UNIQUE | 이름 조회 단일 진입점 |
| `idx_snapshot_person_source` | `profile_snapshot` | `(person_id, source)` | B-Tree, 복합 | 인물·소스별 최신 스냅샷 조회 |
| `idx_snapshot_expires` | `profile_snapshot` | `expires_at` | B-Tree, Partial (`WHERE expires_at IS NOT NULL`) | 만료 행 배치 삭제 |
| `idx_analysis_person_created` | `analysis_result` | `(person_id, created_at DESC)` | B-Tree, 복합 | 인물별 최신 결과 1건 조회 |
| `idx_analysis_expires` | `analysis_result` | `expires_at` | B-Tree, Partial (`WHERE expires_at IS NOT NULL`) | 만료 행 배치 삭제 |
| `idx_history_searched_at` | `search_history` | `searched_at DESC` | B-Tree | 최신 이력 페이지네이션 |
| `idx_history_person` | `search_history` | `person_id` | B-Tree | 인물별 이력 조회 |

> `raw_data` JSONB GIN 인덱스는 특정 키 검색이 필요한 시점에 추가 (`CREATE INDEX ... USING GIN`)

---

## TTL 설계

`expires_at` 컬럼은 `profile_snapshot`과 `analysis_result` 두 테이블에 존재.

| 항목 | 설계 결정 |
|------|----------|
| 컬럼 위치 | 각 테이블에 직접 — 별도 TTL 관리 테이블 불필요 |
| NULL 의미 | 만료 없음 (영구 보존) |
| 설정 주체 | 서비스 레이어 — DB 트리거 미사용 (테스트 용이성) |
| 만료 확인 쿼리 | `WHERE expires_at IS NULL OR expires_at > NOW()` |
| 삭제 방식 | Partial 인덱스 기반 배치 DELETE — soft delete 없이 물리 삭제 |

---

## DB 엔진별 타입 호환

| 항목 | PostgreSQL 15 | MySQL 8.0 | SQLite |
|------|--------------|-----------|--------|
| `BIGSERIAL` | 네이티브 | `BIGINT AUTO_INCREMENT` | `INTEGER` (64bit) |
| `JSONB` | 네이티브 (이진 저장, GIN 지원) | `JSON` (텍스트 저장, 부분 인덱스 제한) | `TEXT` |
| Partial Index | 지원 | 미지원 → 일반 인덱스로 대체 | 지원 |
| `ON DELETE SET NULL` | 지원 | 지원 | 지원 |
| `NOW()` | 지원 | `NOW()` 또는 `CURRENT_TIMESTAMP` | `CURRENT_TIMESTAMP` |

---

## 미결 과제

- [ ] `normalized_name` 생성 규칙 확정 — 소문자·trim 외 유니코드 정규화(NFC) 적용 여부
- [ ] `search_history` RANGE 파티셔닝 도입 시점 결정 (`searched_at` 월별 파티션)
- [ ] `expires_at` 설정 로직 위치 — 서비스 레이어 vs DB 생성 트리거
- [ ] `raw_data` GIN 인덱스 대상 키 확정 (특정 JSON 경로 검색 필요 시)
- [ ] `analysis_result` → `profile_snapshot` RESTRICT 정책으로 인한 삭제 순서 보장 로직

---

## 변경 이력

| 날짜 | 버전 | 내용 |
|------|------|------|
| 2026-05-03 | v1.0 | 초안 작성 |
| 2026-05-03 | v1.1 | 비즈니스 내용 제거, DA 관점으로 재작성 |
