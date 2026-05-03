# Common Spec — 데이터 파이프라인 (Airflow 기반 ODS/DW/Mart)

---

## 목적

운영 데이터와 외부 데이터를 안정적으로 수집·적재하고, 분석/리포팅/의사결정에 사용할 수 있도록  
**ODS → DW → Data Mart** 계층을 Airflow로 오케스트레이션하는 표준을 정의한다.

---

## 범위

- 데이터 수집(배치, CDC) 및 적재
- ODS, DW, Mart 계층 모델링
- Airflow DAG 설계, 재처리(backfill), 실패 복구
- 데이터 품질 검증, 모니터링, 보안/거버넌스 연계

---

## 레이어 정의

| 레이어 | 목적 | 데이터 특성 | 주요 사용자 |
|--------|------|-------------|-------------|
| Landing/Raw | 원천 데이터 원본 보존 | 정제 최소화, append 중심 | 데이터 엔지니어 |
| ODS | 운영 데이터 통합/정규화 | 원천 스키마와 유사, 품질 보정 | 백엔드/분석 엔지니어 |
| DW | 전사 공통 지표 모델 | conformed dimension + fact | 분석가/BI |
| Data Mart | 도메인별 최종 분석 뷰 | 지표 중심, 조회 최적화 | PO/사업/운영 |

---

## 전체 아키텍처

```
[Source Systems]
  ├─ OLTP DB (PostgreSQL/MySQL)
  ├─ SaaS API (CRM, Ads, Billing)
  ├─ Event Stream (Kafka)
  └─ Files (S3/FTP/CSV)
            │
            ▼
      [Ingestion Layer]
      (Airflow DAG + Extractors)
            │
            ▼
      [Landing/Raw Storage]
      (S3/GCS/HDFS, parquet/json)
            │
            ▼
      [ODS]
      (정합성 보정, 표준화, upsert)
            │
            ▼
      [DW]
      (SCD/Fact/Dimension, 공통 지표)
            │
            ▼
      [Data Mart]
      (팀/도메인별 KPI 모델)
            │
            ├─ BI Dashboard
            ├─ Ad-hoc SQL
            └─ ML Feature/Analytics

Airflow: 스케줄링/의존성/재시도/백필/알림 오케스트레이션
```

---

## Airflow DAG 설계 원칙

### DAG 분리

| DAG | 역할 | 실행 주기(예시) |
|-----|------|-----------------|
| `ingest_<source>_to_raw` | 원천 추출 및 raw 저장 | 5분~1시간 |
| `load_raw_to_ods` | 정합성 검사 + ODS upsert | 1시간 |
| `build_dw_core` | DW fact/dim 생성 | 1시간~일 1회 |
| `build_mart_<domain>` | 도메인 마트 구성 | 일 1회 |
| `dq_<layer>_checks` | 품질 검증 및 알림 | 각 DAG 후행 |

### Task Group 구조 (권장)

```
extract -> validate_raw -> load_ods -> transform_dw -> build_mart -> dq_check -> publish
```

### 공통 설정

- `catchup`: 기본 `False`, 백필 시 명시 실행
- `max_active_runs`: DAG 성격에 따라 제한(중복 실행 방지)
- `retries`: 2~5회, `retry_delay`: 지수 백오프
- `execution_timeout`: 태스크별 명시
- 실패 시 Slack/Email/PagerDuty 알림

---

## 데이터 수집 전략

## 1) 배치 수집

- API/파일 소스는 **증분 윈도우**(`updated_at`, `date partition`) 기반 수집
- 재실행 시 동일 결과를 보장하도록 멱등키(`source_id`, `event_id`) 사용

## 2) CDC 수집

- DB 트랜잭션 로그 기반 CDC(Debezium 등) 사용 가능
- 삭제 이벤트(`op=DELETE`)를 ODS에서 soft delete 컬럼으로 반영

## 3) 수집 메타데이터

| 컬럼 | 설명 |
|------|------|
| `ingested_at` | 수집 시각 (UTC) |
| `source_system` | 원천 시스템 식별자 |
| `batch_id` | 배치 실행 ID |
| `record_hash` | 중복/변경 감지용 해시 |

---

## ODS 모델링 기준

- 원천 스키마를 크게 훼손하지 않고 표준 컬럼(시간대, 식별자) 정규화
- **upsert 기본**: PK/unique key 기준 최신값 반영
- 데이터 품질 오류 행은 quarantine 테이블로 분리

예시:

```sql
CREATE TABLE ods_customer (
    customer_id      TEXT PRIMARY KEY,
    name             TEXT,
    email            TEXT,
    status           TEXT,
    source_updated_at TIMESTAMPTZ,
    ingested_at      TIMESTAMPTZ NOT NULL,
    is_deleted       BOOLEAN DEFAULT FALSE
);
```

---

## DW 모델링 기준

- Star schema 권장 (Fact + Conformed Dimension)
- SCD Type 2 적용 대상 명확화(고객 등급, 조직 구조 등)
- 지표 정의는 DW 레이어에서 단일화하여 마트 간 불일치 방지

예시 엔티티:

- `dim_date`, `dim_customer`, `dim_product`
- `fact_order`, `fact_payment`, `fact_session`

---

## Data Mart 구성 기준

- 도메인 목적별 마트 분리: `mart_sales`, `mart_marketing`, `mart_cs`
- KPI 조회 성능 우선(사전 집계/물질화 뷰 사용 가능)
- BI 도구와 계약된 스키마 변경 시 버전 관리 필요

예시:

```sql
CREATE MATERIALIZED VIEW mart_sales_daily AS
SELECT
    order_date,
    channel,
    SUM(revenue) AS revenue,
    COUNT(DISTINCT order_id) AS orders
FROM dw.fact_order
GROUP BY 1, 2;
```

---

## 증분 처리·파티셔닝

| 항목 | 권장 기준 |
|------|-----------|
| 증분 기준 컬럼 | `updated_at` + 보조 키 |
| 파티션 키 | `dt`(일), 필요 시 `source` 추가 |
| 재처리 단위 | 날짜 파티션 단위 재처리 |
| 중복 방지 | merge/upsert + natural key/hash |

권장 규칙:

- Late arriving data 허용 윈도우를 운영 정책으로 정의(예: 최근 3일 재계산)
- 이벤트 시간(`event_time`)과 적재 시간(`ingested_at`) 모두 보존

---

## 데이터 품질(DQ) 규칙

| 검증 유형 | 예시 |
|-----------|------|
| 완전성 | 필수 컬럼 null 비율 임계치 |
| 유일성 | PK/unique key 중복 건수 |
| 참조 무결성 | fact FK가 dim에 존재하는지 |
| 범위 검증 | 금액 음수/비정상 값 |
| 볼륨 이상 탐지 | 전일 대비 급감/급증 |

실패 처리:

1. 임계치 초과 시 해당 DAG fail
2. 영향 범위 태깅(소스/테이블/파티션)
3. 알림 및 재실행 플레이북 연결

---

## Airflow 구현 예시

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.utils.task_group import TaskGroup
from datetime import datetime, timedelta

default_args = {
    "owner": "data-platform",
    "retries": 3,
    "retry_delay": timedelta(minutes=5),
}

with DAG(
    dag_id="load_raw_to_ods",
    start_date=datetime(2026, 1, 1),
    schedule="0 * * * *",
    catchup=False,
    max_active_runs=1,
    default_args=default_args,
) as dag:
    with TaskGroup("ingestion") as ingestion:
        extract = PythonOperator(task_id="extract_raw", python_callable=lambda: None)
        validate = PythonOperator(task_id="validate_raw", python_callable=lambda: None)
        extract >> validate

    with TaskGroup("transform") as transform:
        upsert_ods = PythonOperator(task_id="upsert_ods", python_callable=lambda: None)
        dq = PythonOperator(task_id="dq_ods", python_callable=lambda: None)
        upsert_ods >> dq

    ingestion >> transform
```

---

## 메타데이터·운영 관리

- Airflow Variables/Connections는 코드에 하드코딩하지 않는다
- DAG/태스크 명명 규칙 표준화: `<layer>_<domain>_<action>`
- 데이터셋 계약(스키마, 파티션, SLA) 문서화

SLA 예시:

| 데이터셋 | 도착 SLA | 허용 지연 |
|----------|----------|----------|
| `ods_customer` | 매시 10분 이내 | 20분 |
| `dw.fact_order` | 매시 30분 이내 | 60분 |
| `mart_sales_daily` | 매일 07:00 | 30분 |

---

## 보안·거버넌스 연계

- `data_governance.md` 분류 기준(L1~L4) 적용
- PII 컬럼은 ODS/DW/Mart별 마스킹 정책 분리
- 접근 제어는 레이어별 role 기반 최소 권한 적용
- 감사 로그: 누가 어떤 데이터셋에 접근했는지 추적

---

## 배포 전략 (DAG/SQL)

- DAG 코드: Git 기반 형상관리 + PR 리뷰 필수
- 배포 전 검증: `airflow dags test` 또는 스테이징 실행
- SQL 모델 변경 시 영향 범위(마트/대시보드) 사전 점검
- 실패 시 직전 안정 버전으로 DAG 롤백 가능해야 함

---

## 기술 선택 후보

| 영역 | 선택 후보 |
|------|-----------|
| Orchestrator | Apache Airflow (기본), Prefect(비교 대상) |
| Raw Storage | S3, GCS, HDFS |
| ODS/DW | PostgreSQL, BigQuery, Snowflake, Redshift |
| Transform | SQL + dbt, Spark SQL |
| DQ | Great Expectations, SQL 기반 custom check |
| Metadata | OpenMetadata, DataHub |

---

## 단계별 도입 로드맵 (권장)

1. **Phase 1**: Raw + ODS + 기본 DQ + 일 단위 마트
2. **Phase 2**: DW conformed 모델 + 증분 최적화 + SLA 모니터링
3. **Phase 3**: CDC 고도화 + 자동 데이터 계약 검증 + 비용 최적화

---

## 미결 기술 과제

- [ ] ODS/DW 저장소 확정: PostgreSQL 확장 vs 클라우드 DW
- [ ] dbt 도입 여부 및 Airflow와의 책임 경계
- [ ] CDC 파이프라인 표준(Debezium/Kafka Connect) 확정
- [ ] DQ 임계치 및 fail-open/fail-close 정책 합의
- [ ] 마트 스키마 버전 정책(브레이킹 체인지 공지 절차) 수립

