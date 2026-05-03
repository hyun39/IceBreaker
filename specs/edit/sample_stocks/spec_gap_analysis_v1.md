# S&P 500 분석 서비스 — 스펙 갭 분석 v1 (결정사항 반영)

> `spec_gap_analysis.md`의 미결 과제와 스펙 공백을 모두 결정·보완한 버전.  
> 이 문서 이후로는 `spec_gap_analysis.md`를 폐기하고 본 문서를 기준으로 삼는다.

---

## 확정된 기술 결정 (v1 기준)

| 항목 | 결정값 | 근거 |
|------|--------|------|
| 주가 API | `yfinance` (무료, 비공식) | 비용 0, 500 종목 병렬 수집 ~3분 |
| Raw 스토리지 | 로컬 Parquet | 초기 단계 S3 불필요, 경로: `./data/raw/sp500/price/dt={date}/` |
| LLM 모델 (prod) | `gpt-4o-mini` | 비용 최소화, 분석 품질 충분 |
| LLM 모델 (dev) | `gpt-4o-mini` | prod 동일 (비용 낮아 환경 구분 불필요) |
| 분석 출력 언어 | 한국어 + 영어 이중 출력 | 국내외 사용자 대응 |
| 차트 라이브러리 | `Recharts` | MIT 오픈소스, React 네이티브 |
| 면책문구 | UI 고정 노출 + API 응답 포함 | 투자 자문 아님 명시 |
| 공휴일 캘린더 | `exchange_calendars` (PyPI) | NYSE/NASDAQ 정확한 거래일 지원 |
| DB 마이그레이션 | `Alembic` | FastAPI/SQLAlchemy 표준 연동 |
| Redis 구성 | Standalone (초기) | Cluster는 트래픽 증가 후 전환 |
| Secrets 관리 | Kubernetes Sealed Secrets | Vault 대비 운영 단순 |
| LLM 비용 차단 | 월 $20 한도, 초과 시 DAG skip + Slack | AI팀 합의 |
| 캐시 무효화 | DAG 완료 후 FastAPI HTTP 훅 호출 | 메시지 큐 불필요한 규모 |
| Airflow 메타 DB | 앱 DB와 분리 (별도 postgres 컨테이너) | 운영 안정성 |
| Keycloak Realm | `sp500-platform` | — |

---

## 갭 현황 (v1 기준)

| 갭 항목 | v0 상태 | v1 상태 | 해결 위치 |
|--------|---------|---------|---------|
| build_dw_sector SQL 완성 | ❌ | ✅ | `data_pipeline.md` 수정 |
| quarantine 테이블 DDL | ❌ | ✅ | `schema.md` 수정 |
| Pydantic 응답 스키마 | ❌ | ✅ | `api_contract.md` 신규 |
| 로컬 개발 환경 | ❌ | ✅ | `local_dev_setup.md` 신규 |
| DB 마이그레이션 전략 | ❌ | ✅ | `local_dev_setup.md` 포함 |
| Keycloak 초기 설정 | ❌ | ✅ | `local_dev_setup.md` 포함 |
| 테스트 전략 | ❌ | ✅ | `testing_strategy.md` 신규 |
| Frontend 컴포넌트 구조 | ❌ | ✅ | `frontend.md` 수정 |
| yfinance 수집 방법 | ⚠️ | ✅ | `data_pipeline.md` 수정 |
| 비거래일 캘린더 라이브러리 | ⚠️ | ✅ | `data_pipeline.md` 수정 |
| LLM 이중 출력 | ⚠️ | ✅ | `agent_trend.md` 수정 |
| 면책문구 위치·내용 | ⚠️ | ✅ | `agent_trend.md` + `frontend.md` 수정 |

---

## 스펙 파일 완성도 (v1)

| 파일 | 상태 | 비고 |
|------|------|------|
| `overview.md` | ✅ 완료 | — |
| `business_spec.md` | ✅ 완료 | — |
| `schema.md` | ✅ 완료 | quarantine 테이블 추가 |
| `data_pipeline.md` | ✅ 완료 | yfinance, SQL 완성, 캘린더 |
| `agent_trend.md` | ✅ 완료 | gpt-4o-mini, 이중 출력, 면책 |
| `frontend.md` | ✅ 완료 | Recharts, 컴포넌트 구조, 면책 |
| `software_architecture.md` | ✅ 완료 | 결정값 반영 |
| `api_contract.md` | ✅ 완료 | 신규 |
| `local_dev_setup.md` | ✅ 완료 | 신규 |
| `testing_strategy.md` | ✅ 완료 | 신규 |

**결론**: v1 기준 모든 갭 해소 완료. 개발 착수 가능.
