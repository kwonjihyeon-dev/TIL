# Next.js App Router에서 React Query SSR Hydration + Zustand 패턴 분석

> iOS WebView 환경에서 서버 prefetch한 데이터를 클라이언트에서 그대로 소비하고, 유저 인터랙션 시에만 refetch하는 구조를 만들어보자.
>
> 트러블슈팅 상세 회고는 [blog-ssr-hydration-troubleshooting-qa.md](./blog-ssr-hydration-troubleshooting-qa.md) 참고.

## 목차

1. [개요](#개요)
2. [아키텍처](#아키텍처)
3. [데이터 흐름](#데이터-흐름)
4. [트러블슈팅 요약](#트러블슈팅-요약)
5. [구현 (최종)](#구현-최종)
6. [핵심 포인트: isClient 패턴](#핵심-포인트-isclient-패턴)
7. [패턴 요약](#패턴-요약)

---

## 개요

Next.js App Router 환경의 목록 페이지를 구현한다. **주요 트래픽이 iOS 앱 내 WebView**에서 발생하는 환경이다.

요구사항:

- **SSR**: 첫 진입 시 서버에서 데이터를 미리 fetch하여 HTML에 포함
- **Hydration**: 클라이언트에서 서버 캐시를 그대로 사용 (불필요한 refetch 방지)
- **필터링**: 유저가 필터 조건을 변경하면 클라이언트에서 refetch

사용 기술 스택:

- **Next.js 14+** (App Router, Server Components)
- **@tanstack/react-query v5** (서버 prefetch + 클라이언트 캐시)
- **Zustand** (클라이언트 필터 상태 관리)

---

## 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│  ListPage (Server Component)                             │
│                                                          │
│  ① queryOptions.all()로 queryKey + queryFn 생성            │
│  ② getDehydratedQuery()로 서버에서 prefetch + dehydrate   │
│  ③ <HydrationBoundary>로 클라이언트에 전달                │
│                                                          │
│  ┌───────────────────────────────────────────────┐       │
│  │  ItemListContainer (Client Component)          │       │
│  │                                                │       │
│  │  ④ useSuspenseQuery()로 hydrated cache 소비    │       │
│  │  ⑤ Zustand useFilterStore로 필터 상태 관리      │       │
│  │  ⑥ useItemFilter()로 store → 검색 파라미터 조합 │       │
│  │  ⑦ isClient 플래그로 초기/이후 fetch 분기        │       │
│  └───────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────┘
```

---

## 데이터 흐름

```
[서버]
  prefetchQuery(initSearchData)
    → dehydrate → JSON 직렬화 → HTML에 포함

[클라이언트 - 첫 렌더]
  HydrationBoundary가 dehydratedState를 QueryClient에 복원
    → useSuspenseQuery(initSearchData) → queryKey 일치 → cache hit (fetch 없음)

[클라이언트 - 유저 인터랙션]
  Zustand filter 변경 → searchData 변경 → queryKey 변경
    → useSuspenseQuery가 새 queryKey 감지 → 자동 refetch
```

---

## 트러블슈팅 요약

최종 구현에 도달하기까지 세 가지 문제를 마주했다. 상세 분석은 [트러블슈팅 회고](./blog-ssr-hydration-troubleshooting-qa.md) 참고.

### 문제 1: Safari/iOS WebView Suspense 스트리밍 버그

Safari(iOS WebView 포함)에서 **1KB 미만의 HTML 청크를 버퍼링**하여 Suspense fallback이 표시되지 않는 문제. Chrome에서는 정상이지만 iOS WebView에서는 전체 데이터 로드가 끝날 때까지 흰 화면이 지속되었다.

> [next.js#52444](https://github.com/vercel/next.js/issues/52444) - Safari streaming bug

**해결:** loading 컴포넌트의 HTML 크기가 1KB 이상이 되도록 보장.

### 문제 2: queryKey 불일치로 인한 불필요한 refetch

서버 `initSearchData`와 클라이언트 Zustand store 초기값의 객체 구조가 미묘하게 달라 cache miss가 발생. 동일 API가 서버/클라이언트에서 2번 호출되었다.

```
서버 queryKey:     ['items', { type: 'list', sortBy: 'default' }]
클라이언트 queryKey: ['items', { type: 'list', category: '', sortBy: 'default' }]
                                               ^^^^^^^^^^^^^^^ cache miss!
```

**해결:** `isClient` 플래그로 hydration 시점에는 서버와 동일한 `initSearchData`를 강제 사용.

### 문제 3: useQuery → useSuspenseQuery

`useQuery`는 cache hit 상태에서도 첫 렌더에 `data: undefined`를 반환할 수 있어 로딩 UI 깜빡임이 발생. `useSuspenseQuery`는 cache hit 시 즉시 data를 반환하므로 깜빡임이 없다.

| | `useQuery` | `useSuspenseQuery` |
|---|---|---|
| `data` 타입 | `T \| undefined` | `T` (항상 존재) |
| SSR hydration 시 | 로딩 상태가 잠깐 노출될 수 있음 | cache hit 시 즉시 데이터 반환 |
| 로딩/에러 처리 | 컴포넌트 내부에서 직접 분기 | `<Suspense>` / `<ErrorBoundary>`에 위임 |

**해결:** `useQuery` → `useSuspenseQuery` 전환.

### 해결 전후

```
[Before]
  서버 prefetch (200ms)
  + Safari 버퍼링 (fallback 미표시 → 흰 화면)
  + 클라이언트 refetch (200ms, queryKey 불일치)
  + useQuery 깜빡임
  = ~400ms 이상 + 흰 화면

[After]
  서버 prefetch (200ms)
  + Safari fallback 즉시 표시 (1KB 이상 보장)
  + 클라이언트 cache hit (0ms)
  + 깜빡임 없음
  = ~200ms
```

---

## 구현 (최종)

위의 트러블슈팅을 거쳐 도달한 최종 구현이다.

### 1. 모델 정의

```typescript
// models/item.ts

export interface SearchParams {
  type: 'list' | 'bookmark' | 'history';
  category?: string;
  sortBy?: string;
}

export interface Item {
  id: number;
  title: string;
  price: number;
  isBookmarked: boolean;
}

export interface ListResponse {
  count: number;
  results: Item[];
}
```

### 2. API 서비스 레이어

```typescript
// services/item/itemService.ts

import { ListResponse, SearchParams } from '@/models/item';

class ItemService {
  async getItems(searchData: SearchParams): Promise<ListResponse> {
    const params = new URLSearchParams({
      type: searchData.type,
      ...(searchData.category && { category: searchData.category }),
      ...(searchData.sortBy && { sort_by: searchData.sortBy }),
    });

    const res = await fetch(`/api/items?${params}`);
    return res.json();
  }
}

export default new ItemService();
```

### 3. React Query 옵션

`queryKey`와 `queryFn`을 한곳에서 관리하여 서버/클라이언트 간 **queryKey 불일치를 방지**한다.

```typescript
// services/item/queries.ts

import { SearchParams } from '@/models/item';
import itemService from './itemService';

const queryKeys = {
  all: (search: SearchParams) => ['items', search],
};

const queryOptions = {
  all: (search: SearchParams) => ({
    queryKey: queryKeys.all(search),
    queryFn: () => itemService.getItems(search),
  }),
};

export default queryOptions;
```

### 4. SSR Dehydration 유틸리티

```typescript
// utils/react-query/getDehydratedQuery.ts

import {
  QueryClient,
  QueryKey,
  dehydrate,
  HydrationBoundary,
} from '@tanstack/react-query';
import { cache } from 'react';

export const getQueryClient = cache(() => new QueryClient());

export async function getDehydratedQuery<T>({
  queryKey,
  queryFn,
}: {
  queryKey: QueryKey;
  queryFn: () => Promise<T>;
}) {
  const queryClient = getQueryClient();
  await queryClient.prefetchQuery({ queryKey, queryFn });

  const dehydratedState = dehydrate(queryClient);
  const [target] = dehydratedState.queries.filter(
    (q) => JSON.stringify(q.queryKey) === JSON.stringify(queryKey),
  );

  return { queries: [target], mutations: [] };
}

export const Hydrate = HydrationBoundary;
```

### 5. Zustand 필터 스토어

```typescript
// stores/filterStore.ts

import { create } from 'zustand';
import { SearchParams } from '@/models/item';

type FilterState = {
  type: SearchParams['type'];
  category: string;
  sortBy: string;
  isFiltered: boolean;
};

type FilterAction = {
  updateCategory: (category: string) => void;
  updateSort: (sortBy: string) => void;
  reset: (type: SearchParams['type']) => void;
};

export const useFilterStore = create<FilterState & FilterAction>((set) => ({
  type: 'list',
  category: '',
  sortBy: 'default',
  isFiltered: false,

  updateCategory: (category) =>
    set({ category, isFiltered: true }),

  updateSort: (sortBy) =>
    set({ sortBy }),

  reset: (type) =>
    set({ type, category: '', sortBy: 'default', isFiltered: false }),
}));
```

### 6. 필터 → 검색 파라미터 변환 훅

```typescript
// hooks/useItemFilter.ts

import { SearchParams } from '@/models/item';
import { useFilterStore } from '@/stores/filterStore';

export const useItemFilter = () => {
  const { type, category, sortBy } = useFilterStore();

  const searchData: SearchParams = {
    type,
    ...(category && { category }),
    ...(sortBy && { sortBy }),
  };

  return { searchData };
};
```

### 7. React Query 커스텀 훅

```typescript
// services/item/useItemService.ts

import { SearchParams } from '@/models/item';
import { useSuspenseQuery } from '@tanstack/react-query';
import queryOptions from './queries';

export function useItems(searchData: SearchParams) {
  return useSuspenseQuery(queryOptions.all(searchData));
}
```

### 8. ListPage (Server Component)

```typescript
// app/list/_components/listPage.tsx

import { SearchParams } from '@/models/item';
import queryOptions from '@/services/item/queries';
import {
  Hydrate,
  getDehydratedQuery,
} from '@/utils/react-query/getDehydratedQuery';
import ItemListContainer from '../../_components/itemList/container';
import FilterBar from '../../_components/filter/container';

export default async function ListPage() {
  const initSearchData: SearchParams = {
    type: 'list',
    sortBy: 'default',
  };

  const { queryKey, queryFn } = queryOptions.all(initSearchData);
  const dehydratedState = await getDehydratedQuery({ queryKey, queryFn });

  return (
    <Hydrate state={dehydratedState}>
      <FilterBar type={initSearchData.type} />
      <ItemListContainer initSearchData={initSearchData} />
    </Hydrate>
  );
}
```

### 9. ItemListContainer (Client Component)

```typescript
// app/_components/itemList/container.tsx
'use client';

import { SearchParams } from '@/models/item';
import { useItemFilter } from '@/hooks/useItemFilter';
import { useItems } from '@/services/item/useItemService';
import { useFilterStore } from '@/stores/filterStore';
import { Suspense, useEffect, useState } from 'react';

export default function ItemListContainer({
  initSearchData,
}: {
  initSearchData: SearchParams;
}) {
  const [isClient, setIsClient] = useState(false);
  const { isFiltered } = useFilterStore();
  const { searchData } = useItemFilter();

  // 핵심: 초기에는 서버와 동일한 initSearchData → cache hit
  //       isClient=true 이후에는 Zustand searchData → 필터 변경 시 refetch
  const { data } = useItems(isClient ? searchData : initSearchData);

  useEffect(() => {
    setIsClient(true);
  }, []);

  if (!data) return null;

  if (data.count === 0) {
    return isFiltered ? (
      <p>필터 조건에 맞는 항목이 없습니다.</p>
    ) : (
      <p>항목이 없습니다.</p>
    );
  }

  return (
    <Suspense>
      <ul>
        {data.results.map((item) => (
          <li key={item.id}>
            {item.title} - {item.price}원
          </li>
        ))}
      </ul>
    </Suspense>
  );
}
```

---

## 핵심 포인트: isClient 패턴

```
[첫 렌더 (SSR/Hydration)]
  isClient = false
  → useSuspenseQuery(initSearchData)
  → queryKey가 서버 prefetch와 동일 → cache hit → 네트워크 요청 없음

[useEffect 실행 후]
  isClient = true
  → useSuspenseQuery(searchData)  ← Zustand 기반
  → searchData가 initSearchData와 동일 → queryKey 동일 → 여전히 cache hit

[유저가 필터 변경]
  Zustand store 업데이트 → searchData 변경 → queryKey 변경
  → useSuspenseQuery가 새 queryKey 감지 → 자동 refetch
```

Zustand store는 클라이언트 전용이므로, SSR 시점에는 store의 초기값이 사용된다. 만약 store 초기값과 서버 prefetch 시 사용한 `initSearchData`가 다르면 **queryKey 불일치**가 발생하여 hydration 직후 불필요한 refetch가 일어난다. `isClient` 플래그로 첫 렌더 시에는 서버와 동일한 `initSearchData`를 강제하여 이 문제를 방지한다.

---

## 패턴 요약

| 계층 | 역할 | 기술 |
|---|---|---|
| **Server Component** | SSR prefetch + dehydrate | `getDehydratedQuery` |
| **HydrationBoundary** | 서버 캐시 → 클라이언트 전달 | `@tanstack/react-query` |
| **Client Component** | hydrated cache 소비 + refetch | `useSuspenseQuery` |
| **Filter State** | 클라이언트 필터 상태 관리 | `zustand` |
| **Filter → Params** | store → API 파라미터 변환 | 커스텀 훅 |
| **Service Layer** | HTTP 요청 캡슐화 | 클래스 기반 Service |

### 이 패턴의 장점

1. **SSR 성능**: 서버에서 데이터를 미리 가져오므로 FCP가 빠르다
2. **중복 fetch 방지**: hydration 시 queryKey가 일치하므로 클라이언트에서 재요청하지 않는다
3. **iOS WebView 대응**: Safari 스트리밍 버그를 우회하여 fallback이 정상 표시된다
4. **관심사 분리**: 서버 prefetch / 클라이언트 상태 / API 호출이 각각 독립적으로 관리된다
5. **필터 반응성**: Zustand store 변경 → queryKey 변경 → 자동 refetch로 선언적 데이터 동기화
