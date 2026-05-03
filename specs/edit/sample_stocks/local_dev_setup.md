# S&P 500 Daily 주가 분석 — 로컬 개발 환경 구성

---

## 전제 조건

| 도구 | 버전 | 설치 |
|------|------|------|
| Docker Desktop | 4.x 이상 | https://docs.docker.com/desktop |
| Python | 3.11+ | `pyenv` 권장 |
| Node.js | 20 LTS | `nvm` 권장 |
| `pipenv` | 최신 | `pip install pipenv` |

---

## 디렉터리 구조

```
sp500-platform/
├── airflow/
│   ├── dags/               ← DAG 파일
│   ├── plugins/
│   └── logs/
├── api/                    ← FastAPI 서버
│   ├── app/
│   ├── migrations/         ← Alembic 마이그레이션
│   ├── Pipfile
│   └── Dockerfile
├── frontend/               ← React SPA
│   ├── src/
│   ├── package.json
│   └── Dockerfile
├── data/
│   └── raw/sp500/price/   ← 로컬 Parquet 저장 경로
├── docker-compose.yml
├── docker-compose.override.yml   ← 로컬 전용 오버라이드
└── .env.dev
```

---

## Docker Compose 구성

```yaml
# docker-compose.yml
version: "3.9"

x-airflow-common: &airflow-common
  image: apache/airflow:2.9-python3.11
  environment:
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@airflow-db:5432/airflow
    AIRFLOW__CORE__FERNET_KEY: ${AIRFLOW_FERNET_KEY}
    AIRFLOW__CORE__LOAD_EXAMPLES: "false"
    AIRFLOW__WEBSERVER__SECRET_KEY: ${AIRFLOW_SECRET_KEY}
    STOCK_API: yfinance
    DATABASE_URL: postgresql+asyncpg://sp500:sp500@app-db:5432/sp500
    RAW_DATA_PATH: /opt/airflow/data/raw
    OPENAI_API_KEY: ${OPENAI_API_KEY}
  volumes:
    - ./airflow/dags:/opt/airflow/dags
    - ./airflow/logs:/opt/airflow/logs
    - ./data:/opt/airflow/data
  depends_on:
    airflow-db:
      condition: service_healthy

services:
  # ── 앱 DB (ODS/DW/Mart) ─────────────────────────────────
  app-db:
    image: postgres:15
    environment:
      POSTGRES_USER: sp500
      POSTGRES_PASSWORD: sp500
      POSTGRES_DB: sp500
    ports:
      - "5432:5432"
    volumes:
      - app_db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sp500"]
      interval: 5s
      retries: 5

  # ── Airflow 메타 DB (분리) ────────────────────────────────
  airflow-db:
    image: postgres:15
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - airflow_db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U airflow"]
      interval: 5s
      retries: 5

  # ── Redis ────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      retries: 5

  # ── Keycloak ─────────────────────────────────────────────
  keycloak:
    image: quay.io/keycloak/keycloak:24.0
    command: start-dev
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports:
      - "8180:8080"

  # ── Airflow Webserver ────────────────────────────────────
  airflow-webserver:
    <<: *airflow-common
    command: webserver
    ports:
      - "8080:8080"

  airflow-scheduler:
    <<: *airflow-common
    command: scheduler

  # ── FastAPI ──────────────────────────────────────────────
  api:
    build: ./api
    environment:
      DATABASE_URL: postgresql+asyncpg://sp500:sp500@app-db:5432/sp500
      REDIS_URL: redis://redis:6379/0
      KEYCLOAK_URL: http://keycloak:8080
      KEYCLOAK_REALM: sp500-platform
      KEYCLOAK_CLIENT_ID: backend-api
      ENV: dev
      LOG_LEVEL: DEBUG
    ports:
      - "8000:8000"
    depends_on:
      app-db:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./api:/app

  # ── React Frontend ────────────────────────────────────────
  frontend:
    build:
      context: ./frontend
      target: dev
    environment:
      VITE_API_BASE_URL: http://localhost:8000
      VITE_KEYCLOAK_URL: http://localhost:8180
      VITE_KEYCLOAK_REALM: sp500-platform
      VITE_KEYCLOAK_CLIENT_ID: frontend-app
    ports:
      - "3000:3000"
    volumes:
      - ./frontend/src:/app/src

volumes:
  app_db_data:
  airflow_db_data:
```

---

## 환경변수 파일 (`.env.dev`)

```env
# .env.dev — 로컬 개발 전용, Git 커밋 금지
OPENAI_API_KEY=sk-...
AIRFLOW_FERNET_KEY=...    # python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
AIRFLOW_SECRET_KEY=dev-secret-key
```

---

## 초기 실행 순서

```bash
# 1. 컨테이너 전체 기동
docker compose up -d

# 2. Airflow DB 초기화
docker compose exec airflow-webserver airflow db migrate
docker compose exec airflow-webserver airflow users create \
  --username admin --password admin \
  --firstname Admin --lastname User \
  --role Admin --email admin@example.com

# 3. 앱 DB 마이그레이션 (Alembic)
cd api
pipenv install
pipenv run alembic upgrade head

# 4. 초기 시드 데이터 (dim_sector 11개 Sector)
pipenv run python scripts/seed_sectors.py

# 5. Keycloak Realm·Client 초기 설정 (아래 참조)
# → 브라우저에서 http://localhost:8180 접속

# 6. 개발용 주가 데이터 수집 (10개 종목)
docker compose exec airflow-scheduler \
  airflow dags trigger ingest_sp500_tickers
docker compose exec airflow-scheduler \
  airflow dags trigger ingest_sp500_daily_price \
  --conf '{"trade_date": "2026-05-01", "dev_mode": true}'

# 7. API 확인
curl http://localhost:8000/healthz
curl http://localhost:8000/ready

# 8. 프론트엔드
cd frontend && npm install && npm run dev
```

---

## Alembic 마이그레이션

```bash
# 초기화 (최초 1회)
cd api && pipenv run alembic init migrations

# 새 마이그레이션 생성
pipenv run alembic revision --autogenerate -m "add ods_stock_price_daily"

# 적용
pipenv run alembic upgrade head

# 롤백
pipenv run alembic downgrade -1
```

**마이그레이션 파일 위치**: `api/migrations/versions/`  
**적용 순서**: `dim_sector` → `dim_ticker` → `ods_*` → `fact_*` → `mart_*` → `pipeline_dq_log`

---

## 시드 데이터 (`scripts/seed_sectors.py`)

```python
# 11개 GICS Sector 초기 데이터
SECTORS = [
    {"sector_name": "Information Technology",   "sector_code": "IT"},
    {"sector_name": "Health Care",              "sector_code": "HC"},
    {"sector_name": "Financials",               "sector_code": "FN"},
    {"sector_name": "Consumer Discretionary",   "sector_code": "CD"},
    {"sector_name": "Communication Services",   "sector_code": "CS"},
    {"sector_name": "Industrials",              "sector_code": "IN"},
    {"sector_name": "Consumer Staples",         "sector_code": "ST"},
    {"sector_name": "Energy",                   "sector_code": "EN"},
    {"sector_name": "Utilities",                "sector_code": "UT"},
    {"sector_name": "Real Estate",              "sector_code": "RE"},
    {"sector_name": "Materials",                "sector_code": "MA"},
]
```

---

## Keycloak 초기 구성 (브라우저 `http://localhost:8180`)

### 1. Realm 생성

- `Master` realm 로그인 (admin / admin)
- **Create Realm** → `sp500-platform`

### 2. Client 등록

| Client ID | Type | 설명 |
|-----------|------|------|
| `frontend-app` | Public (PKCE) | React SPA |
| `backend-api` | Bearer-only | FastAPI |

**`frontend-app` 설정값:**
```
Valid Redirect URIs:  http://localhost:3000/*
Web Origins:          http://localhost:3000
```

### 3. Realm Roles 생성

`sp500-platform` realm → Realm roles → `viewer`, `analyst`, `admin`

### 4. 초기 사용자 생성

| 사용자 | 역할 | 비밀번호 |
|--------|------|---------|
| `dev-viewer` | viewer | `dev-viewer` |
| `dev-analyst` | analyst | `dev-analyst` |
| `dev-admin` | admin | `dev-admin` |

---

## Refresh Token 처리 (Frontend)

```typescript
// keycloak-js 라이브러리 사용
import Keycloak from 'keycloak-js';

const kc = new Keycloak({
  url: import.meta.env.VITE_KEYCLOAK_URL,
  realm: import.meta.env.VITE_KEYCLOAK_REALM,
  clientId: import.meta.env.VITE_KEYCLOAK_CLIENT_ID,
});

await kc.init({ onLoad: 'login-required', pkceMethod: 'S256' });

// API 요청 전 토큰 갱신 (만료 30초 전 자동 갱신)
await kc.updateToken(30);
axios.defaults.headers.common['Authorization'] = `Bearer ${kc.token}`;
```

---

## dev 모드 DAG 설정 (10개 종목)

```python
# dags/config.py
DEV_TICKERS = ["AAPL", "MSFT", "GOOGL", "AMZN", "NVDA",
                "META", "TSLA", "BRK.B", "JPM", "JNJ"]

def get_tickers(dev_mode: bool = False) -> list[str]:
    if dev_mode or os.getenv("ENV") == "dev":
        return DEV_TICKERS
    return load_all_sp500_tickers_from_ods()
```
