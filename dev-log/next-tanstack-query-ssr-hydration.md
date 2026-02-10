---
title: Next.js App Router + TanStack Query SSR Hydration + Zustand 예제
date: 2026-02-09
tags: [Next.js, TanStack Query, SSR, Hydration, Zustand, App Router]
references:
  - title: SSR Hydration 최적화 경험 정리
    url: https://kwonjihyeon-dev.github.io/TIL/dev-log/ssr-hydration-presentation
---

# Next.js App Router + TanStack Query SSR Hydration + Zustand 예제

> 서버에서 prefetch한 데이터를 클라이언트 캐시로 넘기고, 유저 인터랙션 시에만 refetch하는 구조.

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

## 구현

### 1. ListPage (Server Component)

```typescript
// app/list/page.tsx

import { Suspense } from 'react';
import { SearchParams } from '@/models/item';
import queryOptions from '@/services/item/queries';
import { Hydrate, getDehydratedQuery } from '@/utils/react-query/getDehydratedQuery';
import ItemListContainer from '../../_components/itemList/container';
import FilterBar from '../../_components/filter/container';
import Loading from './loading';

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
            <Suspense fallback={<Loading />}>
                <ItemListContainer initSearchData={initSearchData} />
            </Suspense>
        </Hydrate>
    );
}
```

### 2. ItemListContainer (Client Component)

```typescript
// app/_components/list/ItemListContainer.tsx
'use client';

import { SearchParams } from '@/models/item';
import { useItemFilter } from '@/hooks/useItemFilter';
import { useItems } from '@/services/item/useItemService';
import { useFilterStore } from '@/stores/filterStore';
import { useEffect, useState } from 'react';

export default function ItemListContainer({ initSearchData }: { initSearchData: SearchParams }) {
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
        return isFiltered ? <p>필터 조건에 맞는 항목이 없습니다.</p> : <p>항목이 없습니다.</p>;
    }

    return (
        <ul>
            {data.results.map((item) => (
                <li key={item.id}>
                    {item.title} - {item.price}원
                </li>
            ))}
        </ul>
    );
}
```

### 3. 모델 정의

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

### 4. API 서비스 레이어

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

### 5. React Query 옵션

서버/클라이언트가 **동일한 queryKey + queryFn**을 사용하도록 한곳에서 관리한다.

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

### 6. SSR Dehydration 유틸리티

```typescript
// utils/react-query/getDehydratedQuery.ts

import { QueryClient, QueryKey, dehydrate, HydrationBoundary } from '@tanstack/react-query';
import { cache } from 'react';

export const getQueryClient = cache(() => new QueryClient());

export async function getDehydratedQuery<T>({ queryKey, queryFn }: { queryKey: QueryKey; queryFn: () => Promise<T> }) {
    const queryClient = getQueryClient();
    await queryClient.prefetchQuery({ queryKey, queryFn });

    const dehydratedState = dehydrate(queryClient);
    const [target] = dehydratedState.queries.filter((q) => JSON.stringify(q.queryKey) === JSON.stringify(queryKey));

    return { queries: [target], mutations: [] };
}

export const Hydrate = HydrationBoundary;
```

### 7. Zustand 필터 스토어

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

    updateCategory: (category) => set({ category, isFiltered: true }),

    updateSort: (sortBy) => set({ sortBy }),

    reset: (type) => set({ type, category: '', sortBy: 'default', isFiltered: false }),
}));
```

### 8. 필터 → 검색 파라미터 변환 훅

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

### 9. React Query 커스텀 훅

```typescript
// services/item/useItemService.ts

import { SearchParams } from '@/models/item';
import { useSuspenseQuery } from '@tanstack/react-query';
import queryOptions from './queries';

export function useItems(searchData: SearchParams) {
    return useSuspenseQuery(queryOptions.all(searchData));
}
```

---

## 트러블슈팅: 이렇게 하면 안 된다

위 구현에 도달하기까지 마주한 문제들을 코드로 정리한다.

### 1. queryKey 불일치 → 불필요한 refetch

```typescript
// 서버 (ListPage)
const initSearchData: SearchParams = {
    type: 'list',
    sortBy: 'default',
};
// → queryKey: ['items', { type: 'list', sortBy: 'default' }]

// 클라이언트 (ItemListContainer)
const { searchData } = useItemFilter();
// Zustand 초기값 → { type: 'list', category: '', sortBy: 'default' }
// → queryKey: ['items', { type: 'list', category: '', sortBy: 'default' }]

const { data } = useItems(searchData);
// ❌ queryKey가 다르므로 cache miss → 같은 API를 다시 호출!
```

```typescript
// ✅ isClient 플래그로 hydration 시점에는 서버와 동일한 값을 사용
const [isClient, setIsClient] = useState(false);
const { data } = useItems(isClient ? searchData : initSearchData);

useEffect(() => {
    setIsClient(true);
}, []);
```

### 2. useQuery → 로딩 UI 깜빡임

```typescript
// ❌ useQuery: cache에 데이터가 있어도 첫 렌더에서 data가 undefined일 수 있음
const { data, isLoading } = useQuery(queryOptions.all(searchData));

if (isLoading) return <Loading />; // hydration 직후 잠깐 보임
return <ItemList items={data.results} />;
```

```typescript
// ✅ useSuspenseQuery: cache hit 시 즉시 data 반환, 깜빡임 없음
const { data } = useSuspenseQuery(queryOptions.all(searchData));

// data는 항상 존재 (T 타입, undefined 아님)
return <ItemList items={data.results} />;
```

### 3. Safari/iOS WebView Suspense 스트리밍 버그

```typescript
// ❌ loading.tsx의 HTML이 1KB 미만이면 Safari가 버퍼링하여 표시하지 않음
// → iOS WebView에서 흰 화면이 오래 지속
export default function Loading() {
    return <Spinner />; // HTML 크기가 너무 작음
}

// ✅ loading.tsx의 HTML이 1KB 이상이 되도록 보장
export default function Loading() {
    return (
        <div>
            <Skeleton /> {/* 충분한 크기의 로딩 UI */}
            <Skeleton />
            <Skeleton />
        </div>
    );
}
```

> [next.js#52444](https://github.com/vercel/next.js/issues/52444) — Safari에서 1KB 미만의 스트리밍 청크를 버퍼링하는 버그. Chrome에서는 정상이지만 iOS WebView에서는 fallback이 표시되지 않는다.

---

## isClient 패턴 정리

이 패턴이 필요한 이유: Zustand store의 초기값과 서버 `initSearchData`의 객체 구조가 다르면 queryKey 불일치로 불필요한 refetch가 발생한다.

```
서버 initSearchData:  { type: 'list', sortBy: 'default' }
Zustand store 초기값: { type: 'list', category: '', sortBy: 'default' }
                                      ^^^^^^^^^^^^^^^ 이 차이로 cache miss
```

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
