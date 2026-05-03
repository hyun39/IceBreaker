# GOV-06 — API 설계 규칙

> **강제 대상**: 모든 REST API 엔드포인트  
> **게이트**: Spectral OpenAPI 린트 — 위반 시 CI 차단  
> **원본 참조**: `enterprise/06.02_gov_api_governance.md`, `common/backend_fastapi.md`

---

## REST 설계 필수 규칙 (MUST)

### URL 규칙
```
/v{N}/{resource-plural}/{id}/{sub-resource}

예:
  GET  /v1/analyses                    목록
  GET  /v1/analyses/{id}               단건
  POST /v1/analyses                    생성
  GET  /v1/analyses/{id}/ice-breakers  하위 리소스

금지:
  /getAnalysis     (동사 금지)
  /analysis        (단수 금지)
  /v1/Analysis     (대문자 금지)
```

### HTTP 메서드·상태 코드

| 메서드 | 용도 | 성공 코드 |
|--------|------|---------|
| GET | 조회 | 200 |
| POST | 생성 | 201 |
| PUT | 전체 수정 | 200 |
| PATCH | 부분 수정 | 200 |
| DELETE | 삭제 | 204 |

### 에러 응답 포맷 (통일 필수)

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "분석 결과를 찾을 수 없습니다.",
    "details": [
      { "field": "trade_date", "issue": "비거래일입니다." }
    ],
    "request_id": "uuid-v4"
  }
}
```

| 상황 | 코드 | error.code |
|------|------|-----------|
| 입력 검증 실패 | 422 | `VALIDATION_ERROR` |
| 리소스 없음 | 404 | `NOT_FOUND` |
| 권한 없음 | 403 | `FORBIDDEN` |
| 인증 실패 | 401 | `UNAUTHORIZED` |
| 서버 오류 | 500 | `INTERNAL_ERROR` |
| 외부 API 오류 | 502 | `EXTERNAL_API_ERROR` |

---

## API 버전 관리 규칙

- URL prefix 방식: `/v1/`, `/v2/`
- Breaking change 시 새 버전 생성
- 구버전 최소 6개월 운영 후 Deprecation 공지
- `Deprecation: date=YYYY-MM-DD` 헤더 추가

---

## OpenAPI 스펙 필수 항목

모든 엔드포인트는 OpenAPI 3.0 스펙을 유지한다.

```yaml
# 필수 항목
/v1/analyses:
  post:
    summary: "분석 요청 생성"          # 필수
    operationId: "createAnalysis"       # 필수 — camelCase
    tags: ["analysis"]                  # 필수
    requestBody:                        # 필수
      required: true
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/AnalysisRequest'
    responses:
      '201':                            # 성공 응답 필수
        description: "분석 생성됨"
      '422':                            # 에러 응답 필수
        $ref: '#/components/responses/ValidationError'
    security:                           # 인증 명시 필수
      - bearerAuth: []
```

---

## CI Spectral 린트 설정

```yaml
# .spectral.yaml
extends: ["spectral:oas"]
rules:
  operation-operationId: error       # operationId 필수
  operation-tags: error              # tags 필수
  operation-summary: error           # summary 필수
  oas3-valid-media-example: warn
```

```yaml
# GitHub Actions
- name: API Lint
  run: spectral lint docs/api/openapi.yaml --fail-severity=error
```

---

## 하위 호환성 규칙

Breaking Change 금지 (기존 버전에서):
- 필드 삭제
- 필드 타입 변경
- 필수 필드 추가
- 상태 코드 변경

허용 (하위 호환):
- 선택적 필드 추가
- 새 상태 코드 추가 (기존 유지)
- 새 엔드포인트 추가
