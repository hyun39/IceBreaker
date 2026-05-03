# 프론트엔드 스펙 (Flask) — Ice Breaker

> **현재 구현체**. Flask Jinja2 템플릿 + Vanilla JS 기반 단일 HTML 파일.

---

## 기술 스택

| 항목 | 선택 | 비고 |
|------|------|------|
| 렌더링 | Flask Jinja2 | `templates/index.html` |
| CSS | mvp.css | CDN (`unpkg.com/mvp.css`) |
| 로딩 스피너 | css-spinners | CDN (`jsdelivr.net`) |
| JS | Vanilla JS | 프레임워크 없음 |
| 정적 파일 | Flask `static/` | `banner.jpeg`, `demo.gif` |

---

## 파일 구조

```
templates/
└── index.html      ← HTML + inline CSS 없음 + <script> 포함 단일 파일
static/
├── banner.jpeg
└── demo.gif
```

---

## 화면 구성

### 입력 영역 (`<header>`)
- 텍스트 인풋: `name` (placeholder: "Enter name")
- 제출 버튼: "Do Your Magic" (`id="magic-button"`)

### 로딩 상태 (`#spinner`, 초기 hidden)
- 제출 즉시 표시, 응답 수신 시 숨김
- 100×100px `three-quarters-loader` 스피너

### 결과 영역 (`#result`, 초기 hidden)

| 섹션 | DOM ID | 데이터 키 |
|------|--------|----------|
| 프로필 사진 | `#profile-pic` | `data.picture_url` |
| Summary | `#summary` | `data.summary_and_facts.summary` |
| Interesting Facts | `#facts` | `data.summary_and_facts.facts[]` |
| Ice Breakers | `#ice-breakers` | `data.ice_breakers.ice_breakers[]` |
| Topics of Interest | `#topics-of-interest` | `data.interests.topics_of_interest[]` |

배열 항목은 `createHtmlList(element, items)` 함수로 `<ul><li>` 생성

---

## 인터랙션 흐름

```
폼 submit 이벤트
  → result 숨김 + spinner 표시
  → fetch("POST /process", FormData)
      → 성공: 각 DOM 요소 업데이트 → spinner 숨김, result 표시
      → 실패: throw Error (현재 UI 피드백 없음)
```

---

## 현재 구현의 한계

| 항목 | 현황 | 개선 방향 |
|------|------|-----------|
| 에러 표시 | 없음 | 에러 배너 또는 토스트 |
| 로딩 취소 | 불가 | 취소 버튼 + `AbortController` |
| 반응형 | mvp.css 기본만 | 모바일 레이아웃 최적화 |
| CDN 의존 | 2개 외부 CDN | 오프라인 필요 시 로컬 번들링 |
| 상태 관리 | DOM 직접 조작 | 컴포넌트화 검토 (→ React 전환 시) |

---

## 미결 과제

- [ ] API 오류 시 사용자에게 에러 메시지 표시
- [ ] 이미지 로딩 실패 시 placeholder 이미지 처리
- [ ] 결과 섹션 fade-in 애니메이션

---

## 변경 이력

| 날짜 | 버전 | 내용 |
|------|------|------|
| 2026-05-03 | v1.0 | 초안 작성 (ui_spec.md에서 분리) |
