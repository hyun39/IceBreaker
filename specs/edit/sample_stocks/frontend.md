# S&P 500 Daily 주가 분석 — 프론트엔드 스펙 (Frontend)

> 참조 공통 스펙: `common/frontend.md`, `common/api_gateway.md`, `common/auth_keycloak.md`

---

## 목적

ODS 원본 데이터부터 LLM Agent가 생성한 Sector 트렌드 분석까지  
**레이어 전체를 단일 SPA에서 탐색·조회**할 수 있는 분석 화면을 제공한다.

---

## 렌더링 방식

| 항목 | 선택 | 근거 |
|------|------|------|
| 렌더링 | CSR (React SPA) | 인증 필요, 복잡한 상호작용·필터 |
| 상태 관리 | Zustand | 가벼운 전역 상태 (날짜·Sector 선택) |
| 차트 | Recharts (MIT 오픈소스) | 주가 시계열·바 차트, 무료 라이선스 |
| 스타일 | Tailwind CSS | |
| API 클라이언트 | Axios + AbortController | 요청 취소, 에러 처리 |

---

## 화면 구성

```
[앱 레이아웃]
  ├─ Header: 날짜 선택기(DatePicker), Sector 필터, 레이어 탭
  └─ 본문
       ├─ [탭: ODS] 종목별 원시 가격 테이블
       ├─ [탭: DW]  Sector 집계 테이블 + 지표 바 차트
       ├─ [탭: Mart — Key Index] Sector KPI 카드 + 테이블
       └─ [탭: Mart — Trend Analysis] LLM 트렌드 카드 목록
```

---

## 화면별 상세

### 1. ODS 탭 — 종목별 원시 데이터

**목적**: 특정 날짜의 전 종목 OHLCV 원본 확인

| 컴포넌트 | 설명 |
|---------|------|
| 날짜 선택기 | 거래일 달력 (공휴일 비활성) |
| Sector 드롭다운 | 전체 / 개별 Sector 필터 |
| 종목 검색 | ticker 또는 회사명 검색 |
| 테이블 | ticker, company, sector, open, high, low, close, adj_close, volume |
| 정렬 | 컬럼 클릭으로 오름/내림차순 |
| 페이지네이션 | 50행 단위 |

```typescript
// API
GET /api/v1/ods/prices?trade_date=2026-05-01&sector=IT&page=1&size=50

// Response
{
  "trade_date": "2026-05-01",
  "total": 503,
  "items": [
    {
      "ticker": "AAPL",
      "company_name": "Apple Inc.",
      "sector": "Information Technology",
      "open": 185.20,
      "high": 187.50,
      "low": 184.80,
      "close": 186.90,
      "adj_close": 186.90,
      "volume": 58234000
    }
  ]
}
```

---

### 2. DW 탭 — Sector 집계

**목적**: Sector별 일별 집계 지표 확인

| 컴포넌트 | 설명 |
|---------|------|
| 날짜 선택기 | 단일 날짜 선택 |
| Sector 비교 바 차트 | 수익률(`sector_return_pct`) 기준 11개 Sector 비교 |
| 집계 테이블 | sector, ticker_count, avg_close, total_volume, advance/decline, return_pct, top_gainer, top_loser |
| Sector 순위 배지 | 수익률 상위 3 Sector 강조 표시 |

```typescript
// API
GET /api/v1/dw/sector?trade_date=2026-05-01

// Response
{
  "trade_date": "2026-05-01",
  "sectors": [
    {
      "sector_name": "Information Technology",
      "sector_code": "IT",
      "ticker_count": 65,
      "avg_close": 486.32,
      "total_volume": 2341870000,
      "advance_count": 51,
      "decline_count": 14,
      "sector_return_pct": 1.23,
      "sector_rank_by_return": 1,
      "top_gainer_ticker": "NVDA",
      "top_loser_ticker": "INTC"
    }
  ]
}
```

---

### 3. Mart 탭 — Key Index

**목적**: Sector별 주요 지표(MA, RSI, 거래량 이상) 카드 형태로 조회

| 컴포넌트 | 설명 |
|---------|------|
| 날짜 선택기 | 단일 날짜 |
| Sector KPI 카드 (11개) | sector명, return_pct, rsi_14, advance_ratio, volume_vs_ma20 |
| MA 추이 라인 차트 | 선택된 Sector의 최근 60일 MA5/MA20/MA60 |
| RSI 게이지 | 과매수(>70) / 과매도(<30) 색상 구분 |
| 거래량 이상 배지 | `volume_vs_ma20 > 1.5` 이면 "거래 급증" 표시 |

```typescript
// API — 단일 날짜 전체 Sector
GET /api/v1/mart/key-index?trade_date=2026-05-01

// API — 단일 Sector 시계열 (차트용)
GET /api/v1/mart/key-index/trend?sector_code=IT&days=60

// Response (시계열)
{
  "sector_code": "IT",
  "series": [
    {
      "trade_date": "2026-05-01",
      "avg_close": 486.32,
      "ma5_close": 480.1,
      "ma20_close": 470.5,
      "ma60_close": 455.2,
      "rsi_14": 62.4,
      "advance_ratio": 78.5,
      "volume_vs_ma20": 1.15
    }
  ]
}
```

---

### 4. Mart 탭 — Trend Analysis (LLM Agent 결과)

**목적**: Agent가 생성한 Sector 트렌드 분석 카드 확인

| 컴포넌트 | 설명 |
|---------|------|
| 날짜 선택기 | 단일 날짜 |
| 분석 준비 상태 표시 | 분석 미완료 시 "분석 진행 중" spinner (폴링 또는 SSE) |
| Sector 트렌드 카드 (11개) | sector명, sentiment 배지(bullish/bearish/neutral), trend_summary |
| 카드 확장 | 클릭 시 key_drivers, risk_factors, recommendation 표시 |
| sentiment 색상 | bullish=초록, bearish=빨강, neutral=회색 |
| 메타 정보 | llm_model, prompt_version, created_at |

```typescript
// API
GET /api/v1/mart/trend-analysis?trade_date=2026-05-01

// Response
{
  "trade_date": "2026-05-01",
  "is_complete": true,         // Agent 실행 완료 여부
  "analyses": [
    {
      "sector_name": "Information Technology",
      "sector_code": "IT",
      "sentiment": "bullish",
      "trend_summary": "AI 반도체 수요 확대에 힘입어 IT 섹터가 강세를 보였습니다.",
      "key_drivers": [
        {"driver": "AI 수요", "detail": "NVDA 실적 기대감으로 반도체 전반 상승"}
      ],
      "risk_factors": [
        {"factor": "금리 불확실성", "detail": "연준 금리 결정 전 관망세 가능"}
      ],
      "recommendation": "단기 모멘텀 유효하나 금리 이벤트 전 비중 조절 고려.",
      "llm_model": "gpt-4o",
      "prompt_version": "v1.0",
      "created_at": "2026-05-01T09:05:32Z"
    }
  ]
}
```

**분석 미완료 폴링 처리:**

```typescript
// 분석 미완료 시 30초 간격 폴링
const { data, isLoading } = useQuery({
  queryKey: ['trend', tradeDate],
  queryFn: () => fetchTrendAnalysis(tradeDate),
  refetchInterval: (data) =>
    data?.is_complete ? false : 30_000,
});
```

---

## API 에러 처리

| HTTP 코드 | 상황 | UI 처리 |
|-----------|------|---------|
| 404 | 해당 날짜 데이터 없음 | "선택한 날짜의 데이터가 없습니다" 안내 |
| 202 | Agent 아직 실행 중 | "분석 진행 중..." 배너 + 폴링 |
| 429 | Rate limit | 재시도 안내 토스트 |
| 500 | 서버 오류 | 에러 배너 + 새로고침 안내 |

---

## API 엔드포인트 목록

| Method | Path | 설명 |
|--------|------|------|
| GET | `/api/v1/ods/prices` | ODS 종목별 일별 가격 |
| GET | `/api/v1/dw/sector` | DW Sector 집계 (단일 날짜) |
| GET | `/api/v1/mart/key-index` | Mart Key Index (단일 날짜) |
| GET | `/api/v1/mart/key-index/trend` | Mart Key Index 시계열 |
| GET | `/api/v1/mart/trend-analysis` | LLM 트렌드 분석 (단일 날짜) |
| GET | `/api/v1/meta/trading-dates` | 조회 가능한 거래일 목록 |

---

## 상태 관리 (Zustand)

```typescript
interface StockStore {
  selectedDate: string;          // 선택된 거래일
  selectedSector: string | null; // 선택된 Sector (null = 전체)
  activeTab: 'ods' | 'dw' | 'mart_index' | 'mart_trend';
  setDate: (date: string) => void;
  setSector: (sector: string | null) => void;
  setTab: (tab: string) => void;
}
```

---

## 성능 기준

| 항목 | 목표 |
|------|------|
| ODS 테이블 초기 로딩 | 2초 이내 |
| DW 집계 차트 렌더링 | 1초 이내 |
| Mart 카드 목록 렌더링 | 1초 이내 |
| Trend Analysis 폴링 간격 | 30초 |
| LCP (최대 콘텐츠 렌더링) | 2.5초 이내 |

---

## 컴포넌트 구조 및 라우팅

```
src/
├── main.tsx                   ← Keycloak 초기화 + React mount
├── App.tsx                    ← React Router 라우팅
├── pages/
│   ├── DashboardPage.tsx      ← 메인 (탭 컨테이너, 날짜/Sector 헤더)
│   └── NotFoundPage.tsx
├── components/
│   ├── layout/
│   │   ├── Header.tsx         ← 날짜 선택기, Sector 필터, 탭
│   │   └── Disclaimer.tsx     ← 면책문구 고정 바
│   ├── ods/
│   │   └── OdsPriceTable.tsx
│   ├── dw/
│   │   ├── SectorBarChart.tsx ← Recharts BarChart
│   │   └── SectorTable.tsx
│   ├── mart/
│   │   ├── KeyIndexCard.tsx
│   │   ├── KeyIndexTrendChart.tsx  ← Recharts LineChart (MA, RSI)
│   │   ├── TrendAnalysisCard.tsx
│   │   └── TrendAnalysisSkeleton.tsx
│   └── common/
│       ├── SentimentBadge.tsx
│       ├── Pagination.tsx
│       └── ErrorBanner.tsx
├── hooks/
│   ├── useOdsPrices.ts
│   ├── useDwSector.ts
│   ├── useMartKeyIndex.ts
│   └── useTrendAnalysis.ts    ← 폴링 로직 포함
├── api/
│   ├── client.ts              ← Axios 인스턴스 + 토큰 자동 주입
│   ├── ods.ts
│   ├── dw.ts
│   └── mart.ts
├── store/
│   └── stockStore.ts          ← Zustand
└── types/
    ├── ods.ts                 ← TypeScript interface (api_contract.md 기반)
    ├── dw.ts
    └── mart.ts
```

### React Router 경로

| 경로 | 컴포넌트 | 설명 |
|------|---------|------|
| `/` | `DashboardPage` (ODS 탭) | 기본 진입 |
| `/?tab=dw` | `DashboardPage` (DW 탭) | 탭 상태 URL 동기화 |
| `/?tab=mart_index` | `DashboardPage` (Key Index 탭) | |
| `/?tab=mart_trend` | `DashboardPage` (Trend 탭) | |

### TypeScript 핵심 타입 (`src/types/mart.ts`)

```typescript
export interface DriverBilingual {
  driver: string;
  detail_ko: string;
  detail_en: string;
}

export interface TrendAnalysisItem {
  sector_name: string;
  sector_code: string;
  sentiment: 'bullish' | 'bearish' | 'neutral';
  trend_summary_ko: string;
  trend_summary_en: string;
  key_drivers: DriverBilingual[];
  risk_factors: DriverBilingual[];
  recommendation_ko: string;
  recommendation_en: string;
  disclaimer_ko: string;
  disclaimer_en: string;
  llm_model: string;
  prompt_version: string;
  created_at: string;
}

export interface TrendAnalysisResponse {
  trade_date: string;
  is_complete: boolean;
  total_sectors: number;
  completed_sectors: number;
  analyses: TrendAnalysisItem[];
}
```

### 면책문구 표시

```typescript
// components/layout/Disclaimer.tsx
export function Disclaimer() {
  return (
    <div className="fixed bottom-0 w-full bg-yellow-50 border-t border-yellow-300
                    text-xs text-yellow-800 px-4 py-1 text-center z-50">
      본 분석은 투자 자문이 아니며 참고 목적으로만 활용하시기 바랍니다.
      (This analysis is not investment advice and is for reference purposes only.)
    </div>
  );
}
```

- Trend Analysis 탭에서는 카드마다 `disclaimer_ko`/`disclaimer_en`도 추가 표시

---

## 미결 기술 과제

- [x] 차트 라이브러리: **Recharts** 확정
- [ ] Trend Analysis 실시간 알림 방식: **폴링(30초)** 우선 적용, SSE는 v2 검토
- [ ] 모바일 반응형 지원 범위
- [ ] ODS 테이블 가상화(Virtual Scroll) 적용 여부 (500행 이상)
