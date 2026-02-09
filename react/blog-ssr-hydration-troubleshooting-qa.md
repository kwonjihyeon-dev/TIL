# SSR Hydration 최적화 트러블슈팅 회고

> Next.js App Router + React Query SSR 환경, iOS 웹뷰에서 목록 페이지의 렌더링 성능 문제를 해결한 과정

---

## 1. 원인

세 가지 근본 원인이 복합적으로 겹쳐 있었다.

### 원인 A: Safari/iOS WebView의 Suspense 스트리밍 버그

Next.js App Router의 SSR Streaming은 HTML을 청크 단위로 전송하고, `<Suspense>` fallback을 먼저 보여준 뒤 데이터가 준비되면 실제 콘텐츠로 교체하는 방식으로 동작한다. 그런데 **Safari(iOS WebView 포함)에서는 청크 크기가 약 1KB 미만일 때 브라우저가 이를 버퍼링**하여 화면에 렌더하지 않는 문제가 있었다.

> [next.js#52444](https://github.com/vercel/next.js/issues/52444) - Safari에서 Suspense fallback HTML 청크가 너무 작으면 브라우저가 버퍼링하여 표시하지 않음. 모든 데이터 로드가 완료될 때까지 흰 화면이 유지됨.

Chrome에서는 Suspense fallback(로딩 UI)이 즉시 표시되지만, iOS WebView에서는 fallback조차 보이지 않고 **전체 데이터 로드가 끝날 때까지 흰 화면**이 지속되었다. 이것이 "흰 화면이 오래 보인다"는 증상의 첫 번째 원인이었다.

### 원인 B: 서버-클라이언트 간 queryKey 불일치

React Query의 SSR Hydration은 서버에서 prefetch한 데이터를 클라이언트 cache에 복원하는 방식으로 동작한다. 이때 cache hit의 조건은 **queryKey의 정확한 일치**다. queryKey에 검색 파라미터 객체 전체를 포함하고 있었는데, 서버(Server Component)와 클라이언트(Zustand store 초기값)에서 이 객체의 구조가 미묘하게 달랐다.

```
서버 queryKey:     ['items', 'user1', { type: 'list', sortBy: 'default' }]
클라이언트 queryKey: ['items', 'user1', { type: 'list', category: '', sortBy: 'default' }]
```

Zustand store의 초기값에 `category: ''`(빈 문자열)이 포함되면서 객체 구조가 달라졌고, React Query는 이를 **서로 다른 쿼리**로 인식했다. 서버에서 미리 가져온 데이터가 cache에 있음에도 클라이언트가 이를 찾지 못해 동일한 API를 다시 호출했다.

### 원인 C: useQuery의 SSR Hydration 동작 특성

`useQuery`는 hydration 시 cache에 데이터가 있더라도 첫 렌더에서 `data: undefined`, `status: 'pending'` 상태를 반환할 수 있다. `useQuery`는 내부적으로 Promise를 throw하지 않고 상태값을 그대로 반환하기 때문에, 컴포넌트가 데이터 없는 상태로 한 번 렌더된 후 → 리렌더로 데이터를 받는 2단계 렌더링이 발생한다.

### 세 원인의 복합 작용

이 세 가지가 합쳐지면서 최악의 시나리오가 만들어졌다:

```
① Safari 스트리밍 버그 → Suspense fallback이 표시되지 않음 → 흰 화면
② queryKey 불일치 → 서버 prefetch 데이터를 못 찾음 → 클라이언트에서 재요청
③ useQuery 동작 → 재요청 중에도 data=undefined → 로딩 UI 표시
④ ①+③ → 로딩 UI마저 Safari에서 버퍼링될 수 있음 → 흰 화면이 더 길어짐

결과: 서버 prefetch 시간 + 클라이언트 재요청 시간 + Safari 버퍼링 시간
     = 흰 화면 또는 로딩바가 비정상적으로 오래 노출
```

---

## 2. 어떤 상황에서 어떤 문제가 발생했는지

### 상황

Next.js App Router 기반 서비스의 목록 페이지. **주요 트래픽이 iOS 앱 내 WebView**에서 발생하는 환경이다. Server Component에서 React Query `prefetchQuery`로 데이터를 미리 가져오고, `HydrationBoundary`로 클라이언트에 전달한 뒤, Client Component에서 동일 데이터를 소비하는 구조. 클라이언트 측에서는 Zustand로 필터 상태를 관리하고, 필터 조건에 따라 목록을 다시 조회하는 기능이 포함되어 있다.

### 문제 현상

**iOS WebView에서 페이지 진입 시 체감되는 흐름:**

```
① 서버가 데이터를 가져옴
② HTML 스트리밍 시작 → Safari가 청크를 버퍼링 → 흰 화면 지속
③ Hydration 시작
④ queryKey 불일치 → 클라이언트에서 동일 API 재요청
⑤ useQuery가 data=undefined 반환 → 로딩 UI 표시 시도
⑥ Safari 버퍼링으로 로딩 UI도 지연 표시
⑦ 데이터 도착 → 콘텐츠 렌더
```

SSR을 적용하기 전(클라이언트 fetch만 하던 때)보다 **오히려 느려진** 상황이었다.

### 디버깅의 어려움

특히 문제를 파악하기 어려웠던 이유는 **iOS WebView 환경 특성** 때문이었다:

- **Chrome DevTools를 직접 사용할 수 없다.** Safari Web Inspector를 통해 디버깅해야 하는데, WebView 내부의 네트워크 탭이나 렌더링 프로파일러 접근이 제한적이다.
- **Chrome에서는 재현되지 않는다.** Safari 스트리밍 버그(원인 A)는 Chrome에서 정상 동작하므로, 개발 환경에서 문제를 확인하기 어렵다.
- **여러 원인이 겹쳐 있어 개별 원인을 분리하기 어렵다.** 흰 화면이 길다는 증상만으로는 Safari 버그인지, queryKey 문제인지, useQuery 문제인지 구분이 안 되었다.

### 구체적 문제 세 가지

| 문제                    | 증상                                      | 영향                                     |
| ----------------------- | ----------------------------------------- | ---------------------------------------- |
| Safari 스트리밍 버그    | fallback이 표시되지 않고 흰 화면          | iOS WebView 사용자 전체에 영향           |
| queryKey 불일치         | 동일 API가 서버/클라이언트에서 2번 호출됨 | 렌더링 시간 2배, 서버 fetch가 무의미해짐 |
| useQuery hydration 동작 | 첫 렌더에서 `data=undefined` → 깜빡임     | 로딩 UI가 추가로 끼어들어 체감 지연      |

---

## 3. 해결 방법과 선택 근거

### 해결 A: Safari 스트리밍 버그 대응 - loading 컴포넌트 크기 확보

[next.js#52444](https://github.com/vercel/next.js/issues/52444)의 임시 해결책을 적용했다. Safari는 1KB 미만의 HTML 청크를 버퍼링하므로, **Suspense fallback(loading 컴포넌트)의 HTML 크기가 1KB 이상**이 되도록 보장했다.

**선택 근거:**

- Safari/WebKit 측 버그이므로 근본적인 수정은 브라우저 업데이트를 기다려야 한다
- Next.js 팀도 Upstream 이슈로 분류하여 자체 수정 계획이 없는 상태였다
- fallback 컴포넌트의 크기를 키우는 것은 기능에 영향 없이 적용 가능한 우회책이다

### 해결 B: `isClient` 플래그로 queryKey 일치 보장

```typescript
const [isClient, setIsClient] = useState(false);
const { searchData } = useFilter(); // Zustand 기반

const { data } = useItems(
  isClient ? searchData : initSearchData, // hydration 시에는 서버와 동일한 key 사용
);

useEffect(() => {
  setIsClient(true);
}, []);
```

**선택 근거:**

- Zustand는 클라이언트 전용 상태이므로, 서버에서 알 수 없는 값이 queryKey에 포함될 수밖에 없다
- hydration이 완료되기 전까지만 서버와 동일한 `initSearchData`를 강제하면 cache hit가 보장된다
- `useEffect`는 hydration 이후에 실행되므로, `isClient` 전환 시점이 자연스럽게 hydration 완료 시점과 일치한다

### 해결 C: `useQuery` → `useSuspenseQuery` 전환

```typescript
// Before
return useQuery(queryOptions.all(searchData));

// After
return useSuspenseQuery(queryOptions.all(searchData));
```

**선택 근거:**

- `useSuspenseQuery`는 cache hit 시 **즉시 data를 반환**하고, cache miss 시에는 **Promise를 throw**하여 `<Suspense>` boundary에 로딩 처리를 위임한다
- 컴포넌트 코드에서 `isLoading` / `isError` 분기가 제거되어 로직이 단순해진다
- SSR hydration 시 서버 prefetch 데이터가 cache에 있으면 Suspense를 트리거하지 않으므로 **깜빡임이 원천 차단**된다
- `data`의 타입이 `T | undefined`에서 `T`로 좁혀져 타입 안전성도 향상된다

---

## 4. 시도했던 여러 접근 방법

### 접근 1: Zustand 초기값을 서버 initSearchData와 일치시키기

```typescript
// filterStore.ts 초기값을 서버와 완전히 동일하게 맞추려는 시도
export const useFilterStore = create((set) => ({
  type: 'list',
  // category를 아예 제거하면?
  sortBy: 'default',
}));
```

**결과:** 필터 기능 자체가 깨짐. store에 `category` 필드가 없으면 필터 업데이트 시 에러 발생. 또한 필터 훅이 store 값을 조합하여 `searchData`를 만드는 과정에서 항상 동일한 구조가 나온다는 보장이 없었다. 근본적 해결이 아니었다.

### 접근 2: queryKey에서 searchData 객체를 제거하고 개별 값만 사용

```typescript
const queryKeys = {
  all: (search: SearchParams) => ['items', search.type, search.sortBy ?? null],
};
```

**결과:** 빈 문자열 문제는 해결됐지만, 필터 조건이 추가될 때마다 queryKey 정의를 수동으로 수정해야 했다. 유지보수 비용이 높고 실수 가능성이 컸다. 또한 이 방법으로는 `useQuery`의 hydration 깜빡임 문제(원인 C)는 해결되지 않았다.

### 접근 3: `staleTime: Infinity`로 클라이언트 refetch 억제

```typescript
const queryOptions = {
  all: (search) => ({
    queryKey: queryKeys.all(search),
    queryFn: /* ... */,
    staleTime: Infinity, // 절대 stale하지 않으므로 refetch 안 함
  }),
};
```

**결과:** queryKey가 일치하는 경우에는 refetch를 방지하지만, queryKey 자체가 다른 상황에서는 효과가 없었다. 새로운 queryKey에 대해서는 staleTime과 무관하게 fetch가 발생한다. 근본 원인(queryKey 불일치)을 해결하지 못했다.

### 접근 4: Safari 버그 우회를 위한 loading 컴포넌트 패딩 추가

```typescript
// loading.tsx - 보이지 않는 패딩으로 1KB 이상 확보
export default function Loading() {
  return (
    <>
      <Skeleton />
      {/* Safari streaming bug workaround: next.js#52444 */}
      <div style={{ display: 'none' }}>{'x'.repeat(1024)}</div>
    </>
  );
}
```

**결과:** Safari에서 fallback이 표시되기 시작했다. 흰 화면 → 로딩 UI는 보이게 되었지만, queryKey 불일치로 인한 이중 요청과 useQuery의 깜빡임 문제는 여전히 남아 있었다. 부분적 해결.

### 접근 5 (최종): loading 크기 확보 + `isClient` 플래그 + `useSuspenseQuery`

세 원인을 각각 정확히 해결하는 조합.

**결과:** Safari에서도 fallback이 정상 표시되고, queryKey 일치가 보장되며, hydration 시 즉시 data를 반환하여 깜빡임 없음. 필터 변경 시에는 자연스럽게 새 queryKey로 refetch.

---

## 5. 최종 선택한 방법을 결정하게 된 이유

### Safari 스트리밍 우회를 선택한 이유

- **브라우저 버그이므로 어플리케이션 레벨에서 근본 수정이 불가능**하다. WebKit 팀이 수정하기 전까지는 우회가 유일한 방법이다.
- Next.js 팀도 해당 이슈를 Upstream으로 분류했으며, React 팀의 fizz(서버 렌더러) 수정을 기다리는 상황이었다.
- loading 컴포넌트의 크기를 확보하는 것은 **부작용이 없고 적용이 간단**하다.

### `isClient` 플래그를 선택한 이유

- **서버/클라이언트 상태 소스가 본질적으로 다르다**는 사실을 인정하는 접근이다. Zustand는 클라이언트 전용이고, 서버에서 그 초기값을 알 방법이 없다. 둘을 억지로 맞추려 하기보다, hydration 시점에만 서버 값을 사용하고 이후에 클라이언트 값으로 전환하는 것이 가장 자연스러웠다.
- **기존 코드 변경이 최소화**된다. queryKey 구조나 store 구조를 건드리지 않고, 컴포넌트에서 삼항 연산자 하나만 추가하면 된다.
- **다른 페이지에도 동일 패턴을 적용**할 수 있다. 목록 컨테이너 컴포넌트는 여러 페이지에서 공유하는 구조이므로, 일관된 패턴이 중요했다.

### `useSuspenseQuery`를 선택한 이유

- **SSR Hydration과의 궁합**이 `useQuery`보다 근본적으로 좋다. cache hit 시 Promise를 throw하지 않고 즉시 data를 반환하므로, 불필요한 렌더 사이클이 없다.
- **컴포넌트 로직이 단순해진다.** `isLoading`, `isError`, `data === undefined` 분기가 전부 제거되고, 로딩은 `<Suspense>`, 에러는 `<ErrorBoundary>`로 위임된다. 관심사 분리가 자연스럽다.
- **React 18+의 Concurrent Features 방향성과 일치**한다. React 팀은 데이터 fetching을 Suspense 기반으로 전환하는 것을 권장하고 있고, TanStack Query v5에서 `useSuspenseQuery`를 별도 훅으로 분리한 것도 이 방향성을 반영한 것이다.

---

## 6. 성과 및 배운 점

### 성과

#### 렌더링 성능

```
[Before]
  서버 prefetch
  + Safari 버퍼링 대기: 가변 (fallback 미표시)
  + 클라이언트 refetch
  + 깜빡임으로 인한 체감 지연
  = 총 체감 시간: 서버 요청에 따른 중복 API 호출 시간 + 흰 화면

[After]
  서버 prefetch
  + Safari fallback 즉시 표시 (1KB 이상 보장)
  + 클라이언트: cache hit (0ms)
  + 깜빡임 없음
  = 총 체감 시간
```

- 클라이언트 측 불필요한 API 호출 **완전 제거** (요청 수 2회 → 1회)
- iOS WebView에서 Suspense fallback **정상 표시**
- Hydration 후 깜빡임(flicker) **완전 제거**
- 페이지 진입 체감 속도 **약 2배 이상 개선**

### 배운 점

1. **대상 환경에서 직접 테스트해야 한다.** Chrome에서 완벽하게 동작해도 iOS WebView에서 전혀 다르게 동작할 수 있다. Safari/WebKit의 스트리밍 처리 방식은 Chrome과 근본적으로 다르며, 이런 차이는 개발 환경에서 발견하기 어렵다.

2. **queryKey는 "참조"가 아니라 "구조"로 비교된다.** React Query는 내부적으로 deep comparison을 수행한다. 빈 문자열(`''`), `undefined`, 누락된 필드 등 사소한 차이도 cache miss를 유발한다.

3. **`useQuery`와 `useSuspenseQuery`는 이름만 다른 게 아니라, 렌더링 모델 자체가 다르다.**

   - `useQuery`: 상태를 반환하는 훅. 컴포넌트가 모든 상태(loading/error/success)를 직접 처리.
   - `useSuspenseQuery`: Promise를 throw하는 훅. 컴포넌트는 success 상태만 처리하고, 나머지는 React boundary에 위임.

4. **SSR의 성능 이점은 "서버에서 가져온 데이터를 클라이언트가 그대로 쓸 때"만 실현된다.** 서버에서 아무리 빨리 가져와도, 클라이언트가 그 cache를 못 찾아 다시 요청하면 SSR은 오버헤드일 뿐이다.

5. **여러 문제가 겹치면 원인 분리가 핵심이다.** "흰 화면이 오래 보인다"는 하나의 증상 뒤에 Safari 버그, queryKey 불일치, useQuery 동작 특성이라는 세 가지 독립적인 원인이 있었다. 한꺼번에 해결하려 하지 않고, 각 원인을 분리하여 하나씩 대응한 것이 효과적이었다.

---

## 7. 해결 후의 변화나 개선된 지표

### 네트워크 관점

| 지표                        | Before                  | After        |
| --------------------------- | ----------------------- | ------------ |
| 페이지 진입 시 API 호출 수  | 2회 (서버 + 클라이언트) | 1회 (서버만) |
| 클라이언트 측 불필요한 요청 | 매 페이지 진입마다 발생 | 0            |
| API 서버 부하               | 2x (동일 요청 중복)     | 1x           |

### 렌더링 관점

| 지표                      | Before                  | After         |
| ------------------------- | ----------------------- | ------------- |
| iOS WebView fallback 표시 | 미표시 (흰 화면)        | 정상 표시     |
| 컴포넌트 렌더 사이클      | 2회 (pending → success) | 1회 (success) |
| 로딩 UI 깜빡임            | 있음                    | 없음          |

### 코드 유지보수 관점

| 지표                  | Before                              | After                  |
| --------------------- | ----------------------------------- | ---------------------- |
| 컴포넌트 내 상태 분기 | `isLoading`, `isError`, `!data` 3개 | `data.count === 0` 1개 |
| data 타입             | `T \| undefined`                    | `T`                    |
| 로딩 UI 관리 위치     | 각 컴포넌트 내부                    | `<Suspense>` boundary  |
| 에러 UI 관리 위치     | 각 컴포넌트 내부                    | `<ErrorBoundary>`      |

### 사용자 체감

- iOS WebView에서 **흰 화면 시간 대폭 감소**
- 페이지 진입 시 콘텐츠가 즉시 표시됨
- 로딩바가 불필요하게 노출되던 현상 해소
- 필터 변경 시에만 자연스럽게 로딩 → 결과 표시 흐름 유지

---

## 8. 해당 경험을 통해 얻은 교훈, 재발 방지 방안

### 교훈

#### SSR은 "도입"이 아니라 "연결"이 핵심이다

SSR prefetch를 추가하는 것 자체는 어렵지 않다. 진짜 어려운 것은 **서버에서 가져온 데이터가 클라이언트에서 정확히 매칭되도록** 보장하는 것이다. `dehydrate → hydrate` 파이프라인의 어느 한 지점이라도 어긋나면 SSR은 오히려 성능을 악화시킨다.

#### 실제 서비스 환경과 개발 환경은 다르다

Chrome 개발 환경에서 정상 동작하더라도, 실제 사용자가 접하는 iOS WebView에서는 완전히 다른 문제가 발생할 수 있다. 특히 SSR Streaming처럼 **브라우저의 HTML 파싱/렌더링 파이프라인에 깊이 의존하는 기능**은 브라우저 간 차이가 크다. 실제 서비스 환경(디바이스, 브라우저, 네트워크)에서의 테스트가 필수적이다.

#### "동작은 하지만 최적이 아닌" 코드를 의심하라

`useQuery`로도 화면은 정상적으로 나왔다. 데이터도 잘 보였다. 하지만 Network 탭을 열어보면 불필요한 요청이 발생하고 있었고, 렌더링 프로파일러를 보면 불필요한 렌더 사이클이 존재했다. "되니까 괜찮다"가 아니라, 의도한 대로 동작하는지 검증하는 습관이 필요하다.

#### 라이브러리의 API가 여러 개인 데는 이유가 있다

React Query가 `useQuery`와 `useSuspenseQuery`를 별도로 제공하는 것은 단순한 편의 기능이 아니다. 두 훅은 **렌더링 모델이 근본적으로 다르며**, SSR Hydration 시의 동작도 완전히 다르다. 비슷해 보이는 API가 여러 개 존재한다면, 각각의 설계 의도와 차이를 이해해야 올바른 선택을 할 수 있다.
