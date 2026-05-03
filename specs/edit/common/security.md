# Common Spec — 보안 (Security)

---

## 보안 레이어 구조

```
[인터넷]
    │  TLS 1.3
    ▼
[API Gateway / Ingress]     ← Rate Limiting, IP 필터링, WAF
    │  JWT 검증
    ▼
[애플리케이션]              ← 입력 검증, 출력 인코딩, CORS
    │  최소 권한
    ▼
[데이터베이스]              ← 암호화, 접근 제어, 감사 로그
```

---

## OWASP Top 10 대응

| # | 취약점 | 대응 방법 |
|---|--------|----------|
| A01 | Broken Access Control | Keycloak RBAC + 메서드 레벨 `@PreAuthorize` |
| A02 | Cryptographic Failures | TLS 1.3 강제, AES-256 저장 암호화, bcrypt 패스워드 |
| A03 | Injection | Pydantic/Bean Validation 입력 검증, ORM 파라미터 바인딩 |
| A04 | Insecure Design | Threat Modeling, 최소 권한 원칙 |
| A05 | Security Misconfiguration | 기본 자격증명 제거, 불필요 포트 차단, 보안 헤더 |
| A06 | Vulnerable Components | Dependabot, Trivy, `pip audit` 정기 스캔 |
| A07 | Auth Failures | Keycloak Brute Force Protection, MFA, 토큰 단일 사용 |
| A08 | Software Integrity | 이미지 서명(Cosign), SBOM 생성 |
| A09 | Logging Failures | 모든 인증 이벤트 로그, PII 마스킹, 변조 불가 로그 |
| A10 | SSRF | 외부 URL 호출 화이트리스트, 내부망 URL 차단 |

---

## TLS 설정

### cert-manager (Kubernetes)

```yaml
# ClusterIssuer — Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx

# Ingress에 TLS 적용
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts: [api.example.com]
      secretName: api-tls-cert
```

### TLS 버전·암호화 스위트 제한

```nginx
# nginx.conf
ssl_protocols TLSv1.3;                       # TLS 1.0/1.1/1.2 비활성
ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers on;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_stapling on;
ssl_stapling_verify on;
```

---

## 보안 HTTP 헤더

### FastAPI 미들웨어

```python
from fastapi import FastAPI, Request, Response

SECURITY_HEADERS = {
    "Strict-Transport-Security": "max-age=63072000; includeSubDomains; preload",
    "X-Content-Type-Options":    "nosniff",
    "X-Frame-Options":           "DENY",
    "X-XSS-Protection":          "0",           # 최신 브라우저는 CSP로 대체
    "Referrer-Policy":           "strict-origin-when-cross-origin",
    "Permissions-Policy":        "geolocation=(), camera=(), microphone=()",
    "Content-Security-Policy":   "default-src 'self'; script-src 'self'",
}

@app.middleware("http")
async def add_security_headers(request: Request, call_next):
    response: Response = await call_next(request)
    for key, value in SECURITY_HEADERS.items():
        response.headers[key] = value
    return response
```

### Spring Boot

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .headers(headers -> headers
            .frameOptions(f -> f.deny())
            .contentTypeOptions(Customizer.withDefaults())
            .httpStrictTransportSecurity(hsts -> hsts
                .maxAgeInSeconds(63072000)
                .includeSubDomains(true)
            )
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'")
            )
        )
        .build();
}
```

---

## 입력 검증 원칙

| 규칙 | 내용 |
|------|------|
| Allowlist | 허용 값 목록 기반 검증 — Blocklist 방식 사용 금지 |
| 길이 제한 | 모든 문자열 필드에 `max_length` 명시 |
| 타입 강제 | Pydantic / Bean Validation 통해 자동 타입 변환 차단 |
| SQL 파라미터 바인딩 | ORM 사용 — raw SQL 작성 시 `?` 플레이스홀더 필수 |
| 파일 업로드 | 확장자 + MIME 타입 검증, 크기 제한, 바이러스 스캔 |

```python
# Pydantic 입력 검증 예시
from pydantic import BaseModel, Field, field_validator
import re

class ProcessRequest(BaseModel):
    name: str = Field(min_length=1, max_length=100)

    @field_validator("name")
    @classmethod
    def name_must_be_safe(cls, v: str) -> str:
        if re.search(r"[<>\"';]", v):
            raise ValueError("허용되지 않는 문자가 포함되어 있습니다.")
        return v.strip()
```

---

## CORS 설정

```python
# FastAPI — 운영 환경: 명시적 Origin만 허용
from fastapi.middleware.cors import CORSMiddleware

ALLOWED_ORIGINS = [
    "https://app.example.com",
    "https://admin.example.com",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,      # "*" 절대 금지 (운영)
    allow_methods=["GET", "POST"],      # 필요한 메서드만
    allow_headers=["Authorization", "Content-Type"],
    allow_credentials=True,
    max_age=600,
)
```

---

## Secrets 관리

### 계층별 전략

| 환경 | 도구 | 비고 |
|------|------|------|
| 로컬 개발 | `.env` 파일 (`.gitignore` 등록 필수) | |
| CI/CD | GitHub Actions Secrets / GitLab CI Variables | 로그 마스킹 확인 |
| K8s 운영 | HashiCorp Vault + Vault Agent Injector | 권장 |
| K8s 대안 | Kubernetes Sealed Secrets | Vault 없을 때 |

```yaml
# Vault Agent Injector — Pod 어노테이션으로 Secrets 주입
metadata:
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "ice-breaker-api"
    vault.hashicorp.com/agent-inject-secret-config: "secret/ice-breaker/prod"
    vault.hashicorp.com/agent-inject-template-config: |
      {{- with secret "secret/ice-breaker/prod" -}}
      export OPENAI_API_KEY="{{ .Data.data.openai_api_key }}"
      {{- end }}
```

### 금지 사항

- `.env` 파일 git 커밋 금지 — `git-secrets` 또는 `pre-commit` 훅으로 강제
- 코드·주석·로그에 API 키 노출 금지
- 개발용 Secrets를 운영에 재사용 금지

---

## 컨테이너 보안

```dockerfile
# 보안 강화 Dockerfile 체크리스트
FROM python:3.11-slim               # 최소 이미지 사용
RUN useradd -m -u 1000 appuser      # non-root 사용자
USER appuser                         # root 실행 금지
COPY --chown=appuser:appuser . .    # 파일 소유권
```

```yaml
# K8s SecurityContext
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true       # 파일시스템 읽기 전용
  capabilities:
    drop: ["ALL"]                    # 모든 Linux capability 제거
```

---

## 의존성 취약점 스캔

### Python

```bash
# CI 단계에 포함
pip audit                            # 알려진 CVE 스캔
trivy fs --exit-code 1 \
  --severity HIGH,CRITICAL .        # 파일시스템 스캔
```

### Java

```bash
./gradlew dependencyCheckAnalyze     # OWASP Dependency Check
trivy image ghcr.io/org/api:latest   # 이미지 스캔
```

### CI 통합

```yaml
# GitHub Actions — 취약점 스캔 job
security-scan:
  runs-on: ubuntu-latest
  steps:
    - uses: aquasecurity/trivy-action@master
      with:
        image-ref: ghcr.io/org/ice-breaker-api:${{ github.sha }}
        severity: HIGH,CRITICAL
        exit-code: 1                 # 발견 시 빌드 실패
```

---

## 감사 로그 (Audit Log)

인증 이벤트는 반드시 별도 감사 로그로 기록한다.

```json
{
  "timestamp": "2026-05-03T10:00:00Z",
  "event_type": "AUTH_SUCCESS",
  "user_id": "user-uuid",
  "ip_address": "192.168.1.1",
  "user_agent": "Mozilla/5.0 ...",
  "resource": "POST /v1/process",
  "result": "SUCCESS"
}
```

| 이벤트 | 기록 필수 여부 |
|--------|--------------|
| 로그인 성공·실패 | 필수 |
| 토큰 발급·갱신·만료 | 필수 |
| 권한 없는 접근 시도 | 필수 |
| 관리자 작업 | 필수 |
| PII 데이터 조회 | 필수 |

---

## 미결 기술 과제

- [ ] WAF 도입 — AWS WAF / Cloudflare WAF / ModSecurity 선택
- [ ] 정기 침투 테스트 주기 수립 (연 1회 이상 권장)
- [ ] Secrets 자동 로테이션 — Vault Dynamic Secrets 적용 범위 결정
- [ ] PII 데이터 암호화 수준 결정 — 컬럼 레벨 vs 애플리케이션 레벨
- [ ] 보안 의존성 스캔 자동 PR 생성 — Dependabot / Renovate 설정
