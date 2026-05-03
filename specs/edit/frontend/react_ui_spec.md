# 프론트엔드 스펙 (React) — Ice Breaker

> **미구현 설계안**. Flask 템플릿을 React SPA로 전환할 경우의 스펙.

---

## 기술 스택 (제안)

| 항목 | 선택 | 비고 |
|------|------|------|
| 프레임워크 | React 18 | Vite 번들러 |
| 언어 | TypeScript | 타입 안전성 확보 |
| 스타일 | Tailwind CSS | 또는 shadcn/ui 컴포넌트 |
| HTTP 클라이언트 | fetch API 또는 Axios | |
| 상태 관리 | React useState / useReducer | 전역 상태 불필요 수준 |
| 백엔드 연동 | Flask API (`POST /process`) | CORS 설정 필요 |

---

## 제안 파일 구조

```
frontend/
├── src/
│   ├── App.tsx
│   ├── components/
│   │   ├── SearchForm.tsx       ← 이름 입력 + 제출
│   │   ├── LoadingSpinner.tsx   ← 로딩 상태
│   │   ├── ProfileCard.tsx      ← 프로필 사진 + 이름
│   │   ├── SummarySection.tsx   ← Summary + Facts
│   │   ├── IceBreakerList.tsx   ← Ice Breakers 목록
│   │   └── InterestList.tsx     ← Topics of Interest 목록
│   ├── types/
│   │   └── icebreaker.ts        ← API 응답 타입 정의
│   └── api/
│       └── icebreaker.ts        ← fetch 함수 모음
├── public/
└── vite.config.ts
```

---

## 컴포넌트 구성

### `<SearchForm />`
- props: `onSubmit: (name: string) => void`, `isLoading: boolean`
- 입력 중 버튼 비활성화, 로딩 중 취소 버튼 표시

### `<ProfileCard />`
- props: `pictureUrl: string`
- 이미지 로딩 실패 시 placeholder 표시 (`onError` 핸들러)

### `<SummarySection />`
- props: `summary: string`, `facts: string[]`

### `<IceBreakerList />`
- props: `items: string[]`

### `<InterestList />`
- props: `items: string[]`

---

## 타입 정의 (`types/icebreaker.ts`)

```typescript
interface SummaryAndFacts {
  summary: string;
  facts: string[];
}

interface IceBreakers {
  ice_breakers: string[];
}

interface Interests {
  topics_of_interest: string[];
}

interface IceBreakerResponse {
  summary_and_facts: SummaryAndFacts;
  ice_breakers: IceBreakers;
  interests: Interests;
  picture_url: string;
}
```

---

## 상태 관리

```
App 컴포넌트 상태:
  status: 'idle' | 'loading' | 'success' | 'error'
  data: IceBreakerResponse | null
  errorMessage: string | null
```

- `status` 기반으로 Spinner / 결과 / 에러 배너 조건부 렌더링
- `AbortController`로 진행 중인 요청 취소 지원

---

## API 연동 (`api/icebreaker.ts`)

```typescript
// 예시 구조
async function fetchIceBreaker(
  name: string,
  signal?: AbortSignal
): Promise<IceBreakerResponse> {
  const formData = new FormData();
  formData.append("name", name);

  const res = await fetch("/process", {
    method: "POST",
    body: formData,
    signal,
  });

  if (!res.ok) throw new Error(`서버 오류: ${res.status}`);
  return res.json();
}
```

---

## Flask와의 차이점

| 항목 | Flask 구현 | React 구현 |
|------|-----------|-----------|
| 렌더링 방식 | 서버 템플릿 | CSR (클라이언트) |
| 상태 관리 | DOM 직접 조작 | React 상태 |
| 에러 처리 | 없음 | `status: 'error'` + 에러 배너 |
| 취소 기능 | 없음 | `AbortController` |
| 타입 안전성 | 없음 | TypeScript 인터페이스 |
| 테스트 | 어려움 | 컴포넌트 단위 테스트 가능 |
| 빌드 배포 | 파일 1개 | 빌드 산출물 → Flask static 서빙 또는 별도 호스팅 |

---

## 전환 시 백엔드 작업

- `app.py`에 CORS 헤더 추가 (`flask-cors`) — 개발 환경에서 다른 포트 사용 시
- 정적 파일 서빙 방식 결정: Flask `static/` 또는 별도 CDN

---

## 미결 과제

- [ ] 기술 스택 최종 확정 (Tailwind vs shadcn/ui)
- [ ] 빌드 산출물 서빙 전략 결정 (Flask 통합 vs 분리 배포)
- [ ] 컴포넌트 라이브러리 선택 및 디자인 시스템 정의
- [ ] 접근성(ARIA) 요건 정의

---

## 변경 이력

| 날짜 | 버전 | 내용 |
|------|------|------|
| 2026-05-03 | v1.0 | 초안 작성 (미구현 설계안) |
