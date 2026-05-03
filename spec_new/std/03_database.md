# STD-03 — 데이터베이스 구현 표준

> 전체 상세: [`detail/database.md`](./detail/database.md)

---

## 컬럼 타입 선택 기준

| 데이터 | 타입 | 이유 |
|--------|------|------|
| URL, 긴 텍스트 | `TEXT` | 길이 예측 불가 |
| 고정 코드 값 | `VARCHAR(N)` + `CHECK` | ENUM보다 값 추가 용이 |
| 외부 API 응답 | `JSONB` | 스키마 유동성 |
| 배열 (facts, ice_breakers) | `JSONB` | 검색 불필요 시 |
| 금액 | `NUMERIC(p,s)` | FLOAT 부동소수점 오류 방지 |
| 타임스탬프 | `TIMESTAMP WITH TIME ZONE` | 타임존 보존 |

---

## PK 전략

```sql
-- 내부 조인용: BIGSERIAL (성능)
id BIGSERIAL PRIMARY KEY

-- 외부 노출용: UUID v4 별도 컬럼
external_id UUID NOT NULL DEFAULT gen_random_uuid() UNIQUE
```

---

## 인덱스 패턴

```sql
-- 단순 조회
CREATE INDEX idx_{table}_{col} ON {table} ({col});

-- 복합: 선택도 높은 컬럼을 앞에
CREATE INDEX idx_snapshot_person_source ON profile_snapshot (person_id, source);

-- Partial: 조건 고정일 때
CREATE INDEX idx_active ON profile_snapshot (expires_at)
    WHERE expires_at IS NOT NULL;

-- 운영 중 인덱스 추가
CREATE INDEX CONCURRENTLY idx_new ON table (col);  -- 락 없이
```

---

## 마이그레이션 필수 패턴

```sql
-- NOT NULL 컬럼 추가 (3단계 무중단)
-- 1단계
ALTER TABLE t ADD COLUMN new_col TEXT;
-- 2단계 (데이터 채우기)
UPDATE t SET new_col = 'default' WHERE new_col IS NULL;
-- 3단계
ALTER TABLE t ALTER COLUMN new_col SET NOT NULL;
```

```python
# Alembic revision
def upgrade():
    op.add_column("t", sa.Column("new_col", sa.TEXT, nullable=True))
    op.execute("UPDATE t SET new_col = 'default' WHERE new_col IS NULL")
    op.alter_column("t", "new_col", nullable=False)

def downgrade():
    op.drop_column("t", "new_col")
```

---

## BDD 테스트 연결 포인트

```python
# conftest.py — Testcontainers로 실제 DB
@pytest.fixture(scope="session")
def postgres():
    with PostgresContainer("postgres:15") as pg:
        # 마이그레이션 적용
        alembic_upgrade(pg.get_connection_url(), "head")
        yield pg

# Given step에서 DB 직접 적재
@given("분석 결과가 저장되어 있다")
def seed_analysis(db_session):
    db_session.add(AnalysisResult(person_id=1, summary="test"))
    db_session.flush()
```

---

## 참조 무결성 요약

| 관계 | ON DELETE | 언제 |
|------|-----------|------|
| 부모 삭제 시 자식 무의미 | `CASCADE` | profile_snapshot → person |
| 자식 존재 시 부모 삭제 차단 | `RESTRICT` | analysis_result → snapshot |
| 자식은 독립적으로 유효 | `SET NULL` | search_history → analysis_result |
