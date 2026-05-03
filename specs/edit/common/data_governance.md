# Common Spec — 데이터 거버넌스

---

## 개요

데이터의 수집·저장·사용·삭제 전 과정에서 품질, 보안, 규정 준수를 보장하기 위한 기술 기준.

---

## 데이터 분류

### 민감도 등급

| 등급 | 정의 | 예시 | 처리 기준 |
|------|------|------|----------|
| **L1 — 공개** | 누구나 접근 가능 | 공개 LinkedIn 정보, 서비스 이름 | 별도 보호 불필요 |
| **L2 — 내부** | 내부 직원만 접근 | 서비스 설정, 로그 | 접근 제어, 암호화 전송 |
| **L3 — 기밀** | 특정 역할만 접근 | 사용자 행동 이력, API 사용량 | 암호화 저장 + 접근 로그 |
| **L4 — 개인정보(PII)** | 최소 권한 접근 | 이름, 이메일, 사진 URL | 암호화 + 마스킹 + 보존 기간 강제 |

### PII 항목 목록 (Ice Breaker 기준)

| 항목 | 출처 | 등급 | 저장 여부 | 마스킹 대상 |
|------|------|------|----------|-----------|
| 이름 (`full_name`) | 사용자 입력 | L4 | 저장 | 로그 마스킹 |
| LinkedIn URL | 에이전트 탐색 | L3 | 저장 | — |
| Twitter username | 에이전트 탐색 | L3 | 저장 | — |
| 프로필 사진 URL | LinkedIn API | L3 | 저장 | — |
| LinkedIn raw 응답 | Scrapin.io | L4 | 저장(JSONB) | 로그 금지 |
| 이메일 (LinkedIn) | LinkedIn API | L4 | 저장(raw_data 내) | 로그 마스킹 |

---

## 데이터 보존 기간

| 데이터 | 보존 기간 | 근거 | 삭제 방식 |
|--------|---------|------|----------|
| `profile_snapshot` | 90일 | 프로필 변동 주기 | 자동 만료 후 배치 삭제 |
| `analysis_result` | 90일 | 스냅샷 기반 동기화 | 자동 만료 후 배치 삭제 |
| `search_history` | 1년 | 서비스 개선 분석 | 연간 배치 삭제 |
| 감사 로그 (Audit Log) | 3년 | 법적 요건 | 콜드 스토리지 아카이빙 |
| OpenSearch 로그 | 90일 | ILM Delete 단계 | ILM 자동 삭제 |

```sql
-- 만료 데이터 배치 삭제 (일별 실행)
DELETE FROM profile_snapshot
WHERE expires_at < NOW()
  AND id IN (
    SELECT id FROM profile_snapshot
    WHERE expires_at < NOW()
    ORDER BY expires_at
    LIMIT 1000
  );
```

---

## 데이터 마스킹

### 로그 마스킹 — FastAPI

```python
# core/logging.py
import re

PII_PATTERNS = {
    "email":    (r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}", "***@***.***"),
    "linkedin": (r"linkedin\.com/in/[a-zA-Z0-9_-]+", "linkedin.com/in/[MASKED]"),
    "name":     None,   # 이름은 패턴 특정 불가 — 필드 레벨 마스킹
}

def mask_pii(text: str) -> str:
    for _, (pattern, replacement) in PII_PATTERNS.items():
        if pattern:
            text = re.sub(pattern, replacement, text)
    return text

# structlog 프로세서로 자동 마스킹
def mask_pii_processor(_, __, event_dict: dict) -> dict:
    if "name" in event_dict:
        event_dict["name"] = "[MASKED]"
    if "linkedin_url" in event_dict:
        event_dict["linkedin_url"] = "linkedin.com/in/[MASKED]"
    return event_dict
```

### OTel Collector 마스킹 (redaction 프로세서)

```yaml
processors:
  redaction:
    allow_all_keys: true
    blocked_values:
      - "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"   # 이메일
      - "linkedin\\.com/in/[a-zA-Z0-9_-]+"                    # LinkedIn URL
```

---

## 데이터 계보 (Data Lineage)

```
[외부 소스]
  Scrapin.io (LinkedIn) ──┐
  Twitter API (Mock)    ──┤
                          ▼
              [profile_snapshot 테이블]  ← raw_data JSONB
                          │
                          ▼
              [analysis_result 테이블]   ← LLM 가공 결과
                    ├─ summary
                    ├─ facts
                    ├─ topics_of_interest
                    └─ ice_breakers
                          │
                          ▼
              [API 응답]  → 클라이언트 화면 표시
                          │
                          ▼
              [search_history 테이블]    ← 접근 이력
```

| 변환 단계 | 입력 | 출력 | 변환 주체 |
|----------|------|------|---------|
| 수집 | 외부 API | `profile_snapshot.raw_data` | `third_parties/*.py` |
| 분석 | `raw_data` | `analysis_result.*` | LangChain 체인 + LLM |
| 캐싱 | `analysis_result` | Redis | 서비스 레이어 |
| 표시 | API 응답 | 브라우저 화면 | 프론트엔드 |

---

## 스키마 버전 관리 (마이그레이션)

### Flyway (Spring Boot)

```
db/migration/
├── V1__create_initial_tables.sql
├── V2__add_expires_at_to_analysis.sql
├── V3__add_search_history_table.sql
└── V4__add_index_normalized_name.sql
```

```sql
-- V2__add_expires_at_to_analysis.sql
-- 하위 호환: NULLABLE로 추가 후 데이터 채우기
ALTER TABLE analysis_result ADD COLUMN expires_at TIMESTAMP;
UPDATE analysis_result
  SET expires_at = created_at + INTERVAL '90 days'
  WHERE expires_at IS NULL;
```

### Alembic (FastAPI / Python)

```python
# alembic/versions/0002_add_expires_at.py
def upgrade():
    op.add_column("analysis_result",
        sa.Column("expires_at", sa.TIMESTAMP, nullable=True))
    op.execute("""
        UPDATE analysis_result
        SET expires_at = created_at + INTERVAL '90 days'
        WHERE expires_at IS NULL
    """)

def downgrade():
    op.drop_column("analysis_result", "expires_at")
```

### 마이그레이션 원칙

| 원칙 | 내용 |
|------|------|
| 순방향 전용 | 운영 DB에 rollback 사용 금지 — 항상 새 migration으로 수정 |
| 무중단 | 컬럼 추가는 NULLABLE → 데이터 채우기 → NOT NULL 순서 |
| 검토 필수 | 대용량 테이블 DDL은 반드시 DBA 검토 후 `CONCURRENTLY` 사용 |
| 테스트 | 마이그레이션을 CI에서 빈 DB에 적용 후 통합 테스트 실행 |

---

## 개인정보 처리 방침 기술 요건

### 수집 최소화

```python
# LinkedIn raw_data에서 불필요 필드 제거 후 저장
EXCLUDED_FIELDS = {"certifications", "recommendations", "connections"}

def sanitize_linkedin_data(raw: dict) -> dict:
    return {k: v for k, v in raw.items()
            if k not in EXCLUDED_FIELDS
            and v not in ([], "", None)}
```

### 접근 제어

| 데이터 | 접근 가능 역할 | DB 레벨 제어 |
|--------|-------------|------------|
| `profile_snapshot.raw_data` | `admin`, `data-analyst` | Row Level Security (RLS) |
| `search_history` | `admin` | RLS |
| `analysis_result` | `user` (본인 데이터) | application 레벨 필터 |

```sql
-- PostgreSQL RLS 예시
ALTER TABLE profile_snapshot ENABLE ROW LEVEL SECURITY;

CREATE POLICY admin_only ON profile_snapshot
    FOR ALL
    TO admin_role
    USING (true);

CREATE POLICY user_own ON profile_snapshot
    FOR SELECT
    TO app_role
    USING (person_id IN (
        SELECT id FROM person WHERE normalized_name = current_setting('app.current_user')
    ));
```

### 삭제 요청 처리 (Right to Erasure)

```python
# 사용자 데이터 삭제 API
async def delete_user_data(name: str):
    normalized = normalize(name)
    async with db.transaction():
        person = await db.get_person_by_name(normalized)
        if person:
            # CASCADE로 관련 데이터 자동 삭제
            await db.delete_person(person.id)
            # 캐시 무효화
            await invalidate_cache(f"icebreaker:v1:analysis:{normalized}")
            # 감사 로그
            logger.info("user_data_deleted", normalized_name=normalized)
```

---

## 미결 기술 과제

- [ ] 개인정보 영향평가(PIA) 수행 시점 결정
- [ ] PostgreSQL RLS 적용 범위 확정 — 전체 PII 테이블 vs 핵심 테이블만
- [ ] 데이터 계보 자동화 도구 검토 — OpenLineage / Marquez 도입 여부
- [ ] 콜드 스토리지 아카이빙 구현 — S3 Glacier / GCS Archive 연동
- [ ] GDPR 적용 여부 판단 — EU 사용자 서비스 여부 확인 후 결정
