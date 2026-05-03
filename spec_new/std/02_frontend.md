# STD-02 — 프론트엔드 구현 표준 (React)

> 전체 상세: [`detail/frontend.md`](./detail/frontend.md)

---

## 상태 머신 패턴 (비동기 API 호출)

```typescript
type Status = 'idle' | 'loading' | 'success' | 'error';

const [status, setStatus]   = useState<Status>('idle');
const [data, setData]       = useState<Result | null>(null);
const [error, setError]     = useState<string | null>(null);

// 렌더링
{status === 'loading' && <LoadingSpinner />}
{status === 'error'   && <ErrorBanner message={error} />}
{status === 'success' && <ResultView data={data} />}
```

---

## API 클라이언트 패턴

```typescript
// api/icebreaker.ts — 관심사 분리
async function fetchAnalysis(
  name: string,
  signal?: AbortSignal,          // 취소 지원
): Promise<AnalysisResponse> {
  const res = await fetch('/v1/analyses', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ name }),
    signal,
  });
  if (!res.ok) throw new ApiError(res.status, await res.text());
  return res.json();
}

// 컴포넌트에서 AbortController 사용
useEffect(() => {
  const controller = new AbortController();
  fetchAnalysis(name, controller.signal)
    .then(setData)
    .catch(err => { if (!err.name.includes('Abort')) setError(err) });
  return () => controller.abort();
}, [name]);
```

---

## 타입 정의

```typescript
// types/analysis.ts
interface AnalysisResponse {
  summary_and_facts: { summary: string; facts: string[] };
  interests:         { topics_of_interest: string[] };
  ice_breakers:      { ice_breakers: string[] };
  picture_url:       string | null;
}
```

---

## 에러 처리 계층

| 계층 | 처리 |
|------|------|
| API 오류 (4xx/5xx) | `catch` → `status: 'error'` → ErrorBanner |
| 네트워크 오류 | `catch(NetworkError)` → 재시도 안내 |
| 이미지 로딩 실패 | `<img onError>` → placeholder |
| 런타임 오류 | `<ErrorBoundary>` → 폴백 UI |

---

## 접근성 필수 항목

```tsx
// 스피너
<div role="status" aria-label="로딩 중">
  <LoadingSpinner />
</div>

// 폼 연결
<label htmlFor="name-input">이름</label>
<input id="name-input" name="name" ... />

// 에러 알림
<div role="alert" aria-live="polite">
  {error}
</div>
```

---

## BDD E2E 연결 (Playwright)

```typescript
// tests/e2e/analysis.spec.ts
test('이름 입력 후 결과가 표시된다', async ({ page }) => {
  await page.goto('/');
  await page.fill('input[name="name"]', 'Harrison Chase');
  await page.click('button[type="submit"]');

  await expect(page.locator('[role="status"]')).toBeVisible();         // 스피너
  await expect(page.locator('#result')).toBeVisible({ timeout: 30000 }); // 결과
  await expect(page.locator('#summary')).not.toBeEmpty();
});
```
