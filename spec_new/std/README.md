# STD — 구현 표준 인덱스

> 각 `0N_*.md` 파일은 BDD/SDD 맥락에 맞게 정제한 **요약·실행 패턴**이다.  
> 같은 번호의 **전체 분량 명세**는 [`detail/`](./detail/) 아래에 두었으며, `specs/edit/` 를 열지 않아도 된다.

---

## 파일 목록

| 파일 | 핵심 내용 | 상세 (`std/detail/`) |
|------|---------|----------------------|
| `01_backend.md` | FastAPI·SpringBoot 계층·에러·비동기 패턴 | `backend_fastapi.md`, `backend_springboot.md` |
| `02_frontend.md` | React 상태·API 클라이언트·에러 처리 | `frontend.md` |
| `03_database.md` | 스키마 설계·인덱스·마이그레이션 | `database.md` |
| `04_auth.md` | Keycloak PKCE·JWT 검증·역할 | `auth_keycloak.md` |
| `05_infra.md` | Docker·K8s·CI/CD 핵심 패턴 | `infra_cicd.md` |
| `06_observability.md` | OTel 계측·구조화 로그·OpenSearch | `observability_otel_opensearch.md` |
| `07_ai.md` | ReAct Agent·RAG·LLM 체인 | `agent.md`, `rag.md` |
| `08_data_pipeline.md` | Airflow·ODS→DW→MART | `data_pipeline_airflow.md` |

---

## BDD와 STD의 연결

```
.feature 파일 (bdd/)
    │  "사용자가 분석을 조회하면 결과가 반환된다"
    ▼
Step 구현 (bdd/02, bdd/03)
    │  TestClient.get("/v1/analyses/trend")
    ▼
실제 구현 코드 (std/ 패턴 참조)
    ├─ API 엔드포인트 → std/01_backend.md
    ├─ DB 조회 → std/03_database.md
    ├─ 인증 검증 → std/04_auth.md
    └─ 로그 기록 → std/06_observability.md
```

---

## STD 파일 사용 기준

| 상황 | 참조 파일 |
|------|---------|
| API 엔드포인트 작성 | `01_backend.md` + `gov/06_api_design.md` |
| DB 스키마 변경 | `03_database.md` + `gov/07_data_policy.md` |
| 인증 연동 | `04_auth.md` |
| Docker 이미지 작성 | `05_infra.md` |
| 로그 추가 | `06_observability.md` |
| LLM 체인 작성 | `07_ai.md` |
| Airflow DAG 작성 | `08_data_pipeline.md` |
