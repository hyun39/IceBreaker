# STD-05 — 인프라·CI/CD 구현 표준

> 전체 상세: [`detail/infra_cicd.md`](./detail/infra_cicd.md)

---

## Docker 멀티스테이지 빌드 (필수)

```dockerfile
# FastAPI
FROM python:3.11-slim AS builder
WORKDIR /app
RUN pip install pipenv && pipenv install --deploy --system

FROM python:3.11-slim AS runtime
RUN useradd -m -u 1000 appuser     # non-root 필수
COPY --from=builder /usr/local/lib/python3.11/site-packages .
COPY . .
USER appuser
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## 이미지 태그 규칙

```
ghcr.io/{org}/{service}:{git-sha}   ← 배포 추적용 (불변)
ghcr.io/{org}/{service}:latest      ← main 브랜치 최신
ghcr.io/{org}/{service}:v1.2.3      ← 릴리스 태그
```

---

## K8s 필수 설정

```yaml
spec:
  containers:
    - resources:
        requests: { cpu: "250m", memory: "256Mi" }
        limits:   { cpu: "1000m", memory: "1Gi" }
      livenessProbe:
        httpGet: { path: /healthz, port: 8000 }
        initialDelaySeconds: 15
      readinessProbe:
        httpGet: { path: /ready, port: 8000 }
        initialDelaySeconds: 5
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
```

---

## CI 파이프라인 순서

```yaml
# .github/workflows/ci.yml
jobs:
  lint:    runs: ruff / black / isort
  test:    runs: pytest --cov --cov-fail-under=80 + BDD
  security: runs: trivy image (HIGH,CRITICAL → exit-code 1)
  build:   runs: docker buildx + push (main 브랜치만)
  # needs: [lint, test, security] → build
```

---

## 환경별 배포 전략

| 환경 | 트리거 | 승인 |
|------|--------|------|
| dev | main 머지 즉시 | 자동 |
| staging | dev 배포 성공 후 | 1인 수동 |
| prod | staging QA 완료 | 2인 수동 |

---

## 헬스체크 엔드포인트 (필수)

```python
@app.get("/healthz")   # Liveness — 항상 200
async def liveness(): return {"status": "ok"}

@app.get("/ready")     # Readiness — DB 연결 확인 후 200
async def readiness(db = Depends(get_db)):
    await db.execute("SELECT 1")
    return {"status": "ready"}
```

---

## BDD 환경 (로컬)

```yaml
# docker-compose.yml — BDD 테스트 실행 환경
services:
  postgres: { image: postgres:15, ports: ["5432:5432"] }
  redis:    { image: redis:7-alpine }
  api:      { build: ., depends_on: [postgres, redis] }
```

```bash
docker compose up -d postgres
pytest tests/bdd/ -v
```
