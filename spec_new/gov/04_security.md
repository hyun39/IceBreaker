# GOV-04 — 보안 필수 규칙

> **강제 대상**: 모든 서비스·코드·배포  
> **게이트**: SAST HIGH/CRITICAL 발견 시 CI 차단, 컨테이너 보안 체크  
> **원본 참조**: `enterprise/04.02_gov_zero_trust_security.md`, `common/security.md`

---

## Zero Trust 원칙 (MUST)

| 원칙 | 강제 방법 |
|------|---------|
| 암묵적 신뢰 없음 | 모든 요청은 JWT 검증 (Keycloak) |
| 최소 권한 | Kubernetes SecurityContext — non-root, readOnlyRootFilesystem |
| 지속 검증 | JWKS 캐시 TTL 1시간, 만료 토큰 즉시 거부 |
| 암호화 전송 | TLS 1.3 필수 — HTTP 리다이렉트 강제 |

---

## 필수 보안 체크리스트

### 코드 레벨
- [ ] 모든 입력값에 Pydantic / Bean Validation 적용
- [ ] SQL은 ORM 파라미터 바인딩만 사용 (raw SQL 금지)
- [ ] 응답에 보안 헤더 포함 (`HSTS`, `X-Frame-Options`, `CSP`)
- [ ] CORS `allow_origins=["*"]` 운영 환경 금지

### Secrets 관리
- [ ] `.env` 파일 `.gitignore` 등록 확인
- [ ] API 키·비밀번호 코드·주석·로그에 절대 노출 금지
- [ ] CI/CD: GitHub Secrets 또는 Vault 사용

### 컨테이너
- [ ] `USER appuser` — non-root 사용자 실행
- [ ] `readOnlyRootFilesystem: true`
- [ ] `allowPrivilegeEscalation: false`
- [ ] `capabilities.drop: ["ALL"]`

### 의존성
- [ ] PR마다 `trivy` 또는 `pip audit` / OWASP Dependency Check 실행
- [ ] HIGH/CRITICAL CVE 발견 시 즉시 업데이트 또는 예외 등록

---

## OWASP Top 10 필수 대응

| 순위 | 항목 | 최소 대응 |
|------|------|---------|
| A01 | Broken Access Control | Keycloak RBAC + 메서드 레벨 권한 |
| A02 | Cryptographic Failures | TLS 1.3, AES-256 저장 암호화 |
| A03 | Injection | ORM·Pydantic 필수 |
| A05 | Security Misconfiguration | 보안 헤더·기본 자격증명 제거 |
| A06 | Vulnerable Components | CI 취약점 스캔 필수 |
| A07 | Auth Failures | Keycloak Brute Force 활성화 |
| A09 | Logging Failures | PII 마스킹 후 로그, 감사 로그 필수 |
| A10 | SSRF | 외부 URL 화이트리스트 |

---

## 감사 로그 필수 항목

다음 이벤트는 반드시 구조화 로그로 기록한다:
- 로그인 성공·실패
- 권한 없는 접근 시도
- PII 데이터 조회
- 관리자 작업

```json
{
  "timestamp": "ISO8601",
  "event_type": "AUTH_FAILURE",
  "user_id": "...",
  "ip_address": "...",
  "resource": "POST /v1/process",
  "result": "DENIED"
}
```

---

## CI 보안 게이트

```yaml
# .github/workflows/security.yml
- name: Trivy 이미지 스캔
  uses: aquasecurity/trivy-action@master
  with:
    severity: HIGH,CRITICAL
    exit-code: 1           # 발견 시 빌드 실패

- name: Secrets 스캔
  uses: gitleaks/gitleaks-action@v2
  # 코드에 하드코딩된 비밀값 감지 시 차단
```
