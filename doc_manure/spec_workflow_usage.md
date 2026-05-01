# Spec 워크플로우 사용 가이드

## 시작하기 전에

Spec 워크플로우는 Claude Code에서 `/spec` 관련 slash command 또는 자연어로 시작합니다.
아이디어 하나만 있으면 됩니다.

---

## 전체 흐름 한눈에 보기

```
1단계: 요구사항 작성  →  사용자 승인
2단계: 설계 문서 작성  →  사용자 승인
3단계: 태스크 목록 작성  →  사용자 승인
4단계: 구현 / 테스트 실행
```

각 단계에서 Claude가 문서를 작성하면, 사용자가 검토 후 승인해야 다음 단계로 넘어갑니다.

---

## 1단계 — 새 Spec 시작하기

### 시작 방법

Claude Code에서 아이디어를 자연어로 설명하면 됩니다.

```
"사용자 로그인 기능을 만들고 싶어. 이메일/비밀번호 인증에 JWT 토큰을 사용하고,
소셜 로그인도 지원했으면 해."
```

Claude가 자동으로:
1. 기능명(kebab-case)을 결정 (예: `user-authentication`)
2. `.claude/specs/user-authentication/` 디렉토리 생성
3. 몇 명의 `spec-requirements` 에이전트를 사용할지 질문

### 에이전트 수 선택

```
Claude: "spec-requirements 에이전트를 몇 개 사용할까요? (1-128)"
```

| 선택 | 상황 |
|------|------|
| `1` | 빠르게 단일 버전 생성, 직접 수정하며 완성 |
| `3~5` | 여러 버전을 비교해 최선안 자동 선택 |
| `10+` | 복잡한 기능, 최고 품질 문서가 필요할 때 |

---

## 2단계 — 요구사항 문서 검토 및 승인

에이전트가 `requirements.md`를 생성하면 Claude가 검토를 요청합니다.

**생성 위치:** `.claude/specs/user-authentication/requirements.md`

### 문서 형식 예시

```markdown
## Requirements

### Requirement 1

**User Story:** As a user, I want to log in with email/password,
so that I can securely access my account.

#### Acceptance Criteria

1. WHEN user submits valid credentials THEN system SHALL issue a JWT token
2. IF credentials are invalid THEN system SHALL return 401 error
3. WHEN token expires THEN system SHALL require re-authentication
```

### 피드백 방법

```
"3번 요구사항에 소셜 로그인(Google, GitHub)도 추가해줘"
"비밀번호 재설정 플로우가 빠진 것 같아"
"좋아, 승인할게"  ← 이 말이 나와야 다음 단계 진행
```

---

## 3단계 — 설계 문서 검토 및 승인

요구사항 승인 후, 에이전트 수를 다시 선택합니다.

```
Claude: "spec-design 에이전트를 몇 개 사용할까요? (1-128)"
```

**생성 위치:** `.claude/specs/user-authentication/design.md`

### 문서에 포함되는 내용

- 시스템 아키텍처 다이어그램 (Mermaid)
- 컴포넌트 설계 및 인터페이스
- 데이터 모델
- 비즈니스 프로세스 흐름도
- 에러 처리 전략

### 피드백 방법

```
"Redis를 세션 캐시로 추가해줘"
"데이터 모델에서 User 테이블 구조 더 자세히 설명해줘"
"설계 좋아, 다음 단계로 가자"  ← 승인
```

---

## 4단계 — 태스크 목록 검토 및 승인

설계 승인 후, 에이전트 수를 선택합니다.

```
Claude: "spec-tasks 에이전트를 몇 개 사용할까요? (1-128)"
```

**생성 위치:** `.claude/specs/user-authentication/tasks.md`

### 문서 형식 예시

```markdown
# Implementation Plan

- [ ] 1. 프로젝트 구조 및 핵심 인터페이스 설정
  - _Requirements: 1.1_

- [ ] 2. 인증 모듈 구현
- [ ] 2.1 JWT 토큰 발급/검증 구현
  - _Requirements: 1.1, 1.3_
- [ ] 2.2 비밀번호 해싱 유틸리티 구현
  - _Requirements: 1.2_

- [ ] 3. API 엔드포인트 구현
  - _Requirements: 2.1, 2.2_
```

### 피드백 방법

```
"2.3으로 소셜 로그인 태스크도 추가해줘"
"태스크 좋아, 시작하자"  ← 승인 후 구현 단계로
```

---

## 5단계 — 구현 실행

### 기본 모드 (순차 실행)

Claude가 태스크를 하나씩 실행합니다.

```
"태스크 1 실행해줘"
→ 구현 완료 → tasks.md에서 [ ] → [x] 자동 마킹
"다음 태스크 해줘"
```

### 병렬 모드 (명시적 병렬)

의존성 없는 태스크를 동시에 실행합니다.

```
"태스크 2.1이랑 2.2 동시에 실행해줘"
```

### 자동 모드 (전체 자동 실행)

의존성을 분석해 자동으로 최적 순서로 실행합니다.

```
"모든 태스크 자동으로 실행해줘"
```

---

## 6단계 — 테스트 생성 (선택)

특정 태스크의 테스트 코드를 자동 생성합니다.

```
"태스크 2.1에 대한 테스트 작성해줘"
```

`spec-test` 에이전트가 생성하는 것:
- `auth.md` — 테스트 케이스 문서 (케이스별 목적, 단계, 예상 결과)
- `auth.test.ts` — 실행 가능한 테스트 코드 (Jest, AAA 패턴)

---

## 중간에 수정이 필요할 때

### 요구사항으로 돌아가기

```
"요구사항에 다시 비밀번호 정책 조건을 추가해야 할 것 같아"
```

→ `spec-requirements` 에이전트가 `requirements.md` 업데이트
→ 이후 설계/태스크도 순차 재승인 필요

### 설계만 수정하기

```
"설계에서 캐싱 전략을 Redis에서 Memcached로 바꿔줘"
```

→ `spec-design` 에이전트가 `design.md` 업데이트

---

## 기존 Spec 이어서 작업하기

이미 생성된 Spec이 있으면 파일 경로를 언급하세요.

```
"`.claude/specs/user-authentication/tasks.md` 열고 태스크 3부터 이어서 해줘"
"기존 user-authentication 설계 수정하고 싶어"
```

---

## 생성되는 파일 구조

```
.claude/specs/
└── user-authentication/
    ├── requirements.md    ← 요구사항 (EARS 형식)
    ├── design.md          ← 기술 설계 (Mermaid 다이어그램 포함)
    └── tasks.md           ← 구현 체크리스트
```

병렬 실행 시 평가 전 임시 파일:
```
requirements_v1.md, requirements_v2.md  →  spec-judge 평가  →  requirements.md
```

---

## 빠른 참조

| 상황 | 입력 예시 |
|------|-----------|
| 새 기능 시작 | `"장바구니 기능 만들고 싶어"` |
| 요구사항 승인 | `"좋아, 승인"` / `"approved"` / `"다음으로"` |
| 피드백 | `"3번 항목에 ~도 추가해줘"` |
| 태스크 실행 | `"태스크 2.1 실행해줘"` |
| 병렬 실행 | `"태스크 2.1, 2.2, 2.3 동시에 실행해줘"` |
| 자동 실행 | `"모든 태스크 자동으로 다 해줘"` |
| 테스트 생성 | `"태스크 2.1 테스트 코드 작성해줘"` |
| 이전 단계 수정 | `"요구사항에 ~을 추가해야 할 것 같아"` |
