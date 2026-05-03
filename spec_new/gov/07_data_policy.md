# GOV-07 — 데이터 분류·보존·접근 정책

> **강제 대상**: 데이터를 저장·처리·전송하는 모든 컴포넌트  
> **게이트**: 마이그레이션 PR → DBA·보안 리뷰 필수  
> **원본 참조**: `common/data_governance.md`, `enterprise/05.02_std_data_platform_standards.md`

---

## 데이터 민감도 등급 (MUST)

| 등급 | 정의 | 처리 기준 |
|------|------|---------|
| **L1 공개** | 누구나 접근 가능 | 별도 보호 불필요 |
| **L2 내부** | 내부 직원만 접근 | 접근 제어·암호화 전송 |
| **L3 기밀** | 특정 역할만 접근 | 암호화 저장·접근 로그 필수 |
| **L4 PII** | 최소 권한 | 암호화+마스킹+보존 기간 강제 |

모든 테이블·필드는 등급을 주석 또는 Data Catalog에 명시한다.

---

## 데이터 보존 기간 규칙 (MUST)

| 데이터 유형 | 보존 기간 | 삭제 방식 |
|-----------|---------|---------|
| 프로필 스냅샷 (L3) | 90일 | `expires_at` 배치 삭제 |
| LLM 분석 결과 (L3) | 90일 | `expires_at` 배치 삭제 |
| 검색 이력 (L3) | 1년 | 연간 배치 삭제 |
| 감사 로그 (L2) | 3년 | 콜드 스토리지 아카이빙 |
| OTel 로그 (L2) | 90일 | ILM 자동 삭제 |

---

## PII 처리 필수 규칙 (MUST)

- [ ] 로그에 이름·이메일·LinkedIn URL 평문 출력 금지
- [ ] `raw_data` (외부 API 응답) 전체 로그 출력 금지
- [ ] PII 필드는 응답에서 필요한 것만 반환 (최소화)
- [ ] OTel Collector `redaction` 프로세서로 로그 자동 마스킹

```python
# structlog PII 마스킹 필수
structlog.contextvars.bind_contextvars(
    name="[MASKED]",           # 이름 마스킹
    linkedin_url="[MASKED]",   # URL 마스킹
)
```

---

## 마이그레이션 규칙 (MUST)

| 규칙 | 내용 |
|------|------|
| 파일 네이밍 | `V{N}__{설명}.sql` (Flyway) 또는 Alembic 버전 |
| 컬럼 추가 | NULLABLE로 먼저 추가 → 데이터 채우기 → NOT NULL 제약 |
| 운영 대형 테이블 인덱스 | `CREATE INDEX CONCURRENTLY` 사용 |
| 롤백 | 운영 환경 rollback 금지 — 새 마이그레이션으로 수정 |
| 리뷰 | 대용량 테이블 DDL은 DBA 리뷰 필수 |

---

## 접근 제어 최소 기준 (MUST)

```sql
-- PostgreSQL RLS 활성화 (L4 테이블 필수)
ALTER TABLE profile_snapshot ENABLE ROW LEVEL SECURITY;

-- 역할별 접근 정책 명시
CREATE POLICY analyst_select ON profile_snapshot
    FOR SELECT TO analyst_role USING (true);
```

- 애플리케이션 DB 계정은 읽기·쓰기만 — DDL 권한 금지
- 마이그레이션 전용 계정 분리
- DB 접속 자격증명은 Vault 또는 K8s Secret 관리

---

## Data Contract 규칙 (팀 간 데이터 공유 시)

팀 간 데이터 공유는 반드시 `contracts/` 디렉토리의 YAML 계약으로 정의한다.

```yaml
# contracts/{domain}/{dataset}_v{N}.yaml 필수 항목
id:      "{domain}.{dataset}.v{N}"
status:  active
owner:   { team: "...", slack: "#..." }
schema:  { fields: [...] }
quality: { completeness: ">=98%", freshness: "..." }
sla:     { delivery_time: "...", retention: "..." }
```

계약 없이 타 팀 DB/테이블 직접 접근 금지.
