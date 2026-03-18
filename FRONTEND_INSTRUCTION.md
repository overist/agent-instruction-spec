---
applyTo: "**/*.tsx"
---

# 프론트엔드 설계 규칙

## 핵심 규칙

모든 코드에 예외 없이 적용한다.

**구조**

- 모든 컴포넌트/페이지 내부는 `// ** Hooks → States → Functions → Lifecycle → Render` 섹션 주석으로 구분한다.
- 모든 페이지 하단에 `guard`와 `layout` 정적 속성을 반드시 선언한다.
- 도메인 타입은 `src/types/index.ts`에 중앙 관리한다. 컴포넌트 Props는 인라인 정의.
- 불필요한 래핑 컴포넌트를 만들지 않는다. 시맨틱 HTML에 Tailwind 클래스를 직접 적용한다.

**스타일링**

- Tailwind CSS 유틸리티 클래스만 사용한다. CSS 모듈, styled-components, MUI `sx` 금지.
- `globals.css`의 `@theme` 디자인 토큰을 우선 사용한다 (`bg-surface`, `text-foreground` 등).
- 인라인 임의값(`bg-[#171717]`)보다 토큰 우선. 임의값이 2회 반복되면 `@theme`에 승격한다.
- 조건부 클래스 결합은 `cn()` 유틸리티를 사용한다.
- 반응형은 Tailwind 접두사(`sm:`, `md:`, `lg:`) 사용. JS 분기는 조건부 렌더링이 필요할 때만.

**데이터**

- 조회(GET)는 SWR 훅, 명령(POST)은 axios 직접 호출.
- 인증 상태는 Redux(persist), 시스템 메타는 SWR→Redux, 페이지 데이터는 SWR.
- `getServerSideProps`/`getStaticProps` 사용하지 않는다. 클라이언트 사이드 데이터 페칭만.

**컴포넌트**

- 모든 사용자 클릭 버튼은 `ThrottledButton`을 사용한다. 네이티브 `<button>` 직접 사용 금지.
- 에러는 `handleAlertError`(toast) 또는 `handleRedirectError`(toast + 리다이렉트) 헬퍼 사용.
- 토스트는 `react-hot-toast`의 `toast.error()`, `toast.success()`.

**코드**

- import는 절대 경로(`src/...`) 기본. 같은 디렉토리 내에서만 상대 경로 허용.
- 컴포넌트는 `default export`, 유틸리티/상수는 `named export`.

---

## 프로젝트 구조

```
src/
├── @core/
│   ├── lib/            # 유틸리티, 설정, 헬퍼 (fetcher, error, convert, constant 등)
│   └── provider/       # 앱 프로바이더 (AuthGuard, LayoutProvider, SysInfoProvider)
├── api/                # 도메인별 API 모듈 (.ts: 순수 함수, .tsx: SWR 훅 포함)
├── components/         # 재사용 UI — 기능별 하위 디렉토리
│   ├── button/         # ThrottledButton, BackButton 등
│   ├── container/      # 기능 컨테이너
│   ├── dialog/         # 모달/다이얼로그
│   ├── header/
│   ├── footer/
│   ├── image/          # 이미지 (로딩/에러 상태 포함)
│   ├── input/
│   ├── layout/         # MainLayout, BlankLayout
│   └── loader/         # Spinner, SkeletonLoader, LoadingCover
├── hooks/              # 커스텀 훅 (.tsx)
├── pages/              # Next.js Pages Router
├── store/              # Redux Toolkit 슬라이스
├── styles/             # globals.css (@theme 디자인 토큰)
├── types/              # 공유 타입 (index.ts)
└── assets/             # 정적 에셋
```

---

## 코드 템플릿

### 페이지

```tsx
const SomePage = () => {
  // ** Hooks
  const router = useRouter()
  const dispatch = useDispatch<AppDispatch>()

  // ** States
  const [isLoading, setIsLoading] = useState(false)

  // ** Functions
  const handleSubmit = async () => { ... }

  // ** Lifecycle
  useEffect(() => { ... }, [])

  // ** Render
  return <div className="flex flex-col gap-4">...</div>
}

const guard: guardType = 'user'   // 'user' | 'all' | null
const layout: layoutType = 'main' // 'main' | 'blank' | null

SomePage.guard = guard
SomePage.layout = layout
export default SomePage
```

| guard | 동작 |
|-------|------|
| `'user'` | 세션 검증 필수. 실패 시 로그아웃 |
| `'all'` | 세션 체크하지만 미인증도 허용 |
| `null` | 인증 검사 없음 |

| layout | 동작 |
|--------|------|
| `'main'` | Header + Footer 전체 레이아웃 |
| `'blank'` | Header만 최소 레이아웃 |
| `null` | blank과 동일 |

데이터는 클라이언트 사이드에서 SWR 또는 API 직접 호출. 페이지는 `components/`의 컴포넌트를 조합하여 구성.

### API 모듈

파일 확장자: SWR 훅 포함 → `.tsx`, 순수 함수만 → `.ts`

```tsx
// src/api/{domain}.tsx
import axios from 'axios'
import { fetcher } from 'src/@core/lib/fetcher'
import useSWR from 'swr'

const baseUrl = '/api/{domain}/'

// 조회 SWR 훅: get + 리소스명 + SWR
export const getItemsSWR = (params) => {
  return useSWR({ url: baseUrl + 'getItems', params }, fetcher)
}

// 명령 API: 동사 + 리소스명
export const createItem = (params) => {
  return axios.post(baseUrl + 'createItem', params)
}
```

- axios 직접 import. 별도 인스턴스(interceptor, 공통 헤더) 없음.
- 모듈 레벨 `baseUrl` 상수 정의, endpoint는 문자열 결합.
- 순수 API 함수는 axios Promise를 그대로 반환 (`.then` 없음).
- SWR 훅은 `src/@core/lib/fetcher.ts`의 fetcher 사용.
- 인증은 서버 세션 쿠키 기반. 프론트에서 헤더 주입하지 않음.

### Redux 슬라이스

```ts
// src/store/{name}.ts
export const revalidateThunk = createAsyncThunk(
  '{name}/revalidate',
  async (_, { rejectWithValue }) => {
    const { data: res } = await getData()
    if (res.statusCode === 200) return res.data
    return rejectWithValue(res)
  }
)

export const someSlice = createSlice({
  name: '{name}',
  initialState: { data: null },
  reducers: {
    setData: (state, action) => { state.data = action.payload },
  },
  extraReducers: (builder) => {
    builder.addCase(revalidateThunk.fulfilled, (state, action) => {
      state.data = action.payload
    })
  }
})
```

| 데이터 | 도구 | 영속화 |
|--------|------|--------|
| 인증 상태 | Redux + redux-persist | localStorage |
| 시스템 메타 | SWR → Redux | 없음 (새로고침 시 재조회) |
| 페이지 데이터 | SWR 훅 | 없음 |
| 명령형 API | axios 직접 호출 | — |

### 커스텀 훅

- 단일 책임: 하나의 관심사만 다룬다.
- 반환값은 객체 또는 원시값. dispatch 함수를 노출하지 않는다.
- 사이드이펙트(API, dispatch)는 훅 내부에서 처리하고 결과만 반환한다.
- 파일 확장자: `.tsx`

### 컴포넌트

파일명: PascalCase (`ThrottledButton.tsx`). 레이아웃/로더는 kebab-case (`main-layout.tsx`).

모든 컴포넌트는 섹션 주석 규칙을 따른다.

**ThrottledButton** — 모든 사용자 클릭 버튼의 기본:

```tsx
export const ThrottledButton = ({ onClick, variant = 'default', className = '', ...props }: ButtonProps) => {
  const throttledClick = useThrottle(onClick)
  return (
    <button
      {...props}
      onClick={throttledClick}
      className={cn(
        'rounded-lg px-4 py-2 font-medium transition-colors disabled:bg-muted disabled:text-muted-foreground',
        variant === 'contained' && 'bg-primary text-primary-foreground',
        variant === 'outlined' && 'border border-border text-foreground',
        variant === 'default' && 'text-foreground',
        className
      )}
    />
  )
}
```

**Dialog LaunchButton** — 다이얼로그는 `LaunchButton` render prop으로 열기 트리거를 외부 주입:

```tsx
const SomeDialog = ({ LaunchButton }) => {
  const [open, setOpen] = useState(false)
  return (
    <>
      {open && (
        <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/50">
          <div className="bg-surface rounded-lg p-6 w-full max-w-md">{/* 내용 */}</div>
        </div>
      )}
      <LaunchButton onClick={() => setOpen(true)} />
    </>
  )
}

// 사용
<SomeDialog LaunchButton={({ onClick }) => <ThrottledButton onClick={onClick}>열기</ThrottledButton>} />
```

**ConfirmDialog** — `open`, `setOpen`, `handleConfirm`을 prop으로 외부 제어. `children`으로 메시지 주입.

**이미지** — 로딩 중 `SkeletonLoader`, 에러 시 fallback 표시 (`onError` 감지).

---

## 프로바이더 계층 (`_app.tsx`)

```
[외부 서비스 프로바이더 — 필요 시]
  └─ Provider (Redux)
    └─ PersistGate (redux-persist — auth 슬라이스 영속화)
      └─ LayoutProvider (Component.layout 기반 레이아웃 선택)
        └─ SWRConfig (revalidateOnFocus, revalidateOnReconnect, shouldRetryOnError: false)
          └─ SysInfoProvider (시스템 정보 SWR 조회 → Redux, 로딩 중 Spinner)
            └─ AuthGuard (Component.guard + useSessionCheck 기반 인증)
              └─ <Component />
```

---

## 스타일링 & 디자인 시스템

### @theme 디자인 토큰 (`globals.css`)

새 프로젝트마다 `@theme` 블록의 값만 교체하면 전체 앱의 톤이 바뀌도록 설계한다.

```css
/* src/styles/globals.css */
@import url('https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/variable/pretendardvariable-dynamic-subset.min.css');
@import 'tailwindcss';
@plugin "@tailwindcss/typography";

@theme {
  --color-gray-50:  #f8f9fa;
  --color-gray-100: #f1f3f5;
  --color-gray-200: #e9ecef;
  --color-gray-300: #dee2e6;
  --color-gray-400: #ced4da;
  --color-gray-500: #adb5bd;
  --color-gray-600: #868e96;
  --color-gray-700: #495057;
  --color-gray-800: #343a40;
  --color-gray-900: #212529;
  --color-gray-950: #212529;

  --color-background: var(--color-gray-50);
  --color-foreground: var(--color-gray-900);
  --color-surface: #ffffff;
  --color-muted: var(--color-gray-500);
  --color-border: var(--color-gray-200);

  --color-primary: #2563eb;
  --color-primary-foreground: #ffffff;
  --color-secondary: var(--color-gray-100);
  --color-secondary-foreground: var(--color-gray-900);
  --color-destructive: #ef4444;
  --color-destructive-foreground: #ffffff;

  --font-sans: 'Pretendard Variable', 'Roboto', 'Helvetica', 'Arial', sans-serif;
  --font-brand: 'KOHIBaeum', sans-serif;
}
```

시맨틱 토큰(`bg-surface`, `text-foreground`)을 우선 사용. 미세 조정 시에만 gray 스케일 직접 사용. 새 색상 필요 시 먼저 `@theme`에 토큰 추가.

| DO | DON'T |
|----|-------|
| `bg-surface`, `text-foreground` | `bg-[#ffffff]`, `text-[#212529]` |
| `bg-gray-100`, `text-gray-600` | `bg-[#f1f3f5]`, `text-[#868e96]` |
| `sm:pt-2 md:pt-4 lg:pt-8` | `useMediaQueries()`로 스타일 분기 |
| `p-4`, `gap-2`, `mt-6` | `style={{ padding: '1rem' }}` |
| `hover:bg-primary/90` | `onMouseEnter` + setState |

### cn() 유틸리티

```tsx
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'
export function cn(...inputs: ClassValue[]) { return twMerge(clsx(inputs)) }
```

### 반응형

모바일 퍼스트. 기본 = 모바일, `sm:` `md:` `lg:`로 확장.

```tsx
// Tailwind 접두사로 해결
<div className="flex flex-col sm:flex-row gap-2 sm:gap-4">
  <div className="w-full sm:w-1/2">...</div>
</div>

// JS 분기가 반드시 필요한 경우만 훅 사용
const { isSm } = useMediaQueries()
return isSm ? <MobileDrawer /> : <DesktopNav />
```

### 글로벌 기본 스타일

```css
@layer base {
  html, body {
    @apply font-sans text-foreground bg-background text-base;
    max-width: 1440px;
    margin: 0 auto;
  }
  * {
    @apply box-border select-none;
    -webkit-tap-highlight-color: transparent;
  }
}
```

---

## 컨벤션

**Prettier**: `singleQuote: true`, `semi: false`, `trailingComma: "none"`, `tabWidth: 2`, `printWidth: 120`

**ESLint**: `no-explicit-any: warn`, `no-unused-vars: warn`, `exhaustive-deps: warn`

**TypeScript**: `strict: false`, `baseUrl: "."` (절대 경로 `src/...`), `target: es6`

**환경 변수**: 클라이언트 `NEXT_PUBLIC_*`, 서버 전용은 접두사 없이.

**에러 처리** (`src/@core/lib/error.ts`): `handleAlertError`(toast 표시), `handleRedirectError`(toast 후 401→`/login`, 그 외→`/500`).

**토스트**: `react-hot-toast` — `_app.tsx`에서 `<Toaster position="top-center" />` 전역 설정.
