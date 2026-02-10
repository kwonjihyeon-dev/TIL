---
title: SSR Hydration 최적화 경험 정리
date: 2026-02-09
tags: [Next.js, TanStack Query, SSR, Hydration, iOS WebView, 트러블슈팅]
references:
    - title: Next.js App Router + TanStack Query SSR Hydration + Zustand 예제
      url: https://kwonjihyeon-dev.github.io/TIL/dev-log/next-tanstack-query-ssr-hydration
---

# SSR Hydration 최적화 경험 정리

## 1. 배경: 목록 화면을 개발해야 했다

목록 페이지를 새로 개발하게 되었습니다. 필터 조건에 따라 목록을 조회하고, 사용자가 필터를 변경하면 다시 조회하는 전형적인 목록 화면입니다.

---

## 2. 기존 문제: 서버/클라이언트 상태 분리가 안 되어 있었다

기존 구조에서 가장 힘들었던 점은 **상태의 역할 구분이 되어 있지 않았다**는 것입니다.

하나의 상태 관리 영역에 성격이 전혀 다른 데이터가 섞여 있었습니다:

-   **서버 데이터**: API에서 가져온 목록 응답
-   **초기값**: 페이지 진입 시의 기본 검색 조건
-   **유저 입력값**: 필터 변경, 정렬 변경 등 사용자 인터랙션

이 세 가지가 하나의 store에 뒤섞여 있으니, "지금 이 데이터가 서버에서 온 건지, 초기값인지, 유저가 바꾼 건지"를 코드에서 구분하기가 어려웠습니다. 데이터를 새로 가져와야 하는 시점, 캐시를 써도 되는 시점, 초기화해야 하는 시점을 판단하는 게 복잡했습니다.

---

## 3. 기술 선택: TanStack Query + SSR Hydration 전략

이런 상태 분리 문제를 해결하기 위해, 그리고 목록 페이지인 만큼 SSR의 이점을 살리기 위해 TanStack Query를 선택했습니다.

### 왜 TanStack Query인가

첫째, **서버 상태와 클라이언트 상태를 명확히 분리**할 수 있습니다. TanStack Query가 서버 데이터의 캐싱/갱신/무효화를 전담하고, Zustand 같은 클라이언트 상태 관리는 필터 조건 같은 UI 상태만 담당하면 됩니다. 기존에 하나의 store에 뒤섞여 있던 것을 역할별로 나눌 수 있었습니다.

둘째, **SSR을 위한 공식 API**가 있습니다. `dehydrate`/`HydrationBoundary` API를 통해 서버에서 prefetch한 데이터를 클라이언트 캐시로 자연스럽게 전달할 수 있습니다. 직접 구현하면 캐시 매칭, 중복 요청 방지, 갱신 로직을 다 만들어야 하는데, TanStack Query는 이걸 이미 해결해 놓았습니다.

셋째, **경험이 있는 라이브러리**였습니다. 팀에서 이미 사용해본 경험이 있어 학습 비용을 줄일 수 있었습니다.

### SSR + Hydration 전략

목록 페이지는 첫 진입 시 빈 화면 없이 콘텐츠가 바로 보이는 게 중요합니다. 그래서 SSR + Hydration 전략을 수행하기로 했습니다.

의도한 흐름은 이렇습니다:

```
서버에서 목록 데이터를 미리 fetch
→ HTML에 데이터를 포함하여 전송
→ 클라이언트는 서버 캐시를 그대로 사용 (추가 fetch 없음)
→ 사용자가 필터를 변경할 때만 새로 fetch
```

---

## 4. 문제 발생: 오히려 느려졌다

그런데 실제로 iOS WebView에 올려보니, **의도와 정반대의 결과**가 나왔습니다.

### iOS WebView에서 드러난 문제

개발 과정에서 Chrome 브라우저로 확인할 때는 문제를 인지하지 못했습니다. SSR이 정상 동작하는 것처럼 보였고, 데이터도 잘 표시되었습니다.

하지만 iOS 앱 WebView에 얹으니, 사용자에게 보이는 흐름은 이랬습니다:

**패턴 A**: 흰 화면이 비정상적으로 오래 노출 → 콘텐츠가 한번에 표시

**패턴 B**: 흰 화면 → 비정상적으로 오래 노출되는 로딩바 → 콘텐츠

두 가지 패턴으로 화면이 그러졌는데 이 경우 둘 다 **SSR을 도입한 의미가 없는 수준**이었습니다.

서버에서 데이터를 미리 가져왔는데도 흰 화면, 그리고 로딩바까지 오래 노출되고 디버깅해보니 클라이언트에서 같은 API를 다시 호출하는 상황이 발생했습니다.

### 디버깅이 쉽지 않았다

특히 어려웠던 점은 **iOS WebView에서의 디버깅**이었습니다.

-   Chrome DevTools를 바로 쓸 수 없고, Safari Web Inspector를 통해야 하는데 접근이 제한적입니다
-   Chrome에서는 재현 자체가 안 되는 문제가 있었습니다 (Safari 고유 버그)
-   "흰 화면이 길다"는 하나의 증상 뒤에 여러 원인이 겹쳐 있어서, 뭐가 문제인지 분리하기 어려웠습니다

---

## 5. 트러블슈팅: 원인을 분리하고 하나씩 해결했다

결국 "흰 화면이 오래 보인다"는 하나의 증상 뒤에 **세 가지 독립적인 원인**이 있다는 걸 파악했습니다. 한꺼번에 해결하려 하지 않고, 각 원인을 분리해서 하나씩 대응했습니다.

### 원인 1: Safari/iOS WebView의 Suspense 스트리밍 버그

Next.js의 SSR Streaming은 HTML을 청크 단위로 전송하는데, Safari에서는 **1KB 미만의 청크(파일)는** 화면에 표시하지 않는 버그가 있었습니다. Suspense fallback(로딩 UI)의 HTML이 너무 작아서 Safari가 이를 무시하고 있었던 것입니다.

Chrome에서는 정상적으로 로딩 UI가 먼저 보이는데, iOS WebView에서는 전체 데이터 로드가 끝날 때까지 아예 흰 화면이 유지되었습니다.

이건 Next.js GitHub과 WebKit 버그 트래커에서도 언급된 내용이었습니다.

**해결**: loading 컴포넌트의 HTML 크기가 1KB 이상이 되도록 수정했습니다.

### 원인 2: queryKey 불일치로 인한 이중 요청

TanStack Query의 hydration은 queryKey가 정확히 일치해야 cache hit가 발생합니다. 그런데 서버에서 사용한 검색 파라미터와 클라이언트의 Zustand store 초기값이 미묘하게 달랐습니다.

```
서버:     { type: 'list', sortBy: 'default' }
클라이언트: { type: 'list', category: '', sortBy: 'default' }
```

`category: ''` 하나 때문에 queryKey가 달라졌고, 서버에서 미리 가져온 데이터가 캐시에 있는데도 클라이언트가 이를 찾지 못해 같은 API를 다시 호출하고 있었습니다.

**해결**: `isClient` 플래그를 도입해서, hydration 시점에는 서버와 동일한 `initSearchData`를 강제로 사용하도록 했습니다. `useEffect` 실행 후에 Zustand 기반 값으로 전환하면, hydration 시 cache hit가 보장되고 이후 필터 변경 시에는 자연스럽게 refetch가 일어납니다.

### 원인 3: useQuery의 hydration 동작 특성

queryKey 문제를 해결한 뒤에도 로딩 UI가 잠깐 깜빡이는 현상이 남아 있었습니다.

`useQuery`는 cache에 데이터가 있어도 첫 렌더에서 `data: undefined`를 반환할 수 있습니다. 반면 `useSuspenseQuery`는 cache hit 시 즉시 data를 반환합니다. 두 훅은 이름만 다른 게 아니라 **렌더링 모델 자체가 다릅니다.**

-   `useQuery`: 상태를 반환. 컴포넌트가 loading/error/success를 직접 처리
-   `useSuspenseQuery`: cache miss 시 Promise를 throw. 컴포넌트는 success만 처리하고 나머지는 Suspense/ErrorBoundary에 위임

SSR hydration에서는 서버에서 prefetch한 데이터가 cache에 있으므로, `useSuspenseQuery`가 즉시 data를 반환하여 불필요한 렌더 사이클이 사라집니다.

**해결**: `useQuery` → `useSuspenseQuery`로 전환했습니다.

---

## 6. 결과

세 가지 원인을 각각 해결한 뒤:

-   클라이언트 측 불필요한 API 호출 제거
-   iOS WebView에서 Suspense fallback UI **정상 표시**
-   Hydration 후 깜빡임 제거

처음에 의도했던 SSR + Hydration의 이점을 적용할 수 있었습니다.

---

## 7. 배운 것

이 경험에서 가장 크게 배운 것은 세 가지입니다.

**첫째, 사용하는 기술을 정확히 이해하기.** SSR Hydration은 서버에서 prefetch한 데이터를 클라이언트 캐시로 넘겨주는 구조이고, TanStack Query는 이 과정에서 queryKey가 정확히 일치해야 cache hit가 발생합니다. 또한 `useQuery`와 `useSuspenseQuery`는 렌더링 모델 자체가 다릅니다. 이런 동작 원리를 정확히 이해하지 않으면, SSR을 도입하고도 이점을 전혀 살리지 못하는 상황이 생길 수 있습니다.

**둘째, 개발 환경과 실제 서비스 환경은 다릅니다.** Chrome에서 완벽해도 iOS WebView에서 완전히 다른 문제가 발생할 수 있습니다. 특히 SSR Streaming처럼 브라우저의 렌더링 파이프라인에 의존하는 기능은 브라우저 간 차이가 큽니다.

**셋째, 여러 문제가 겹쳐 있을 때는 원인 분리가 핵심입니다.** "흰 화면과 로딩 스켈레톤이 오래 보인다"는 증상 뒤에 Safari 버그, queryKey 불일치, useQuery 동작 특성이라는 세 가지 독립적인 원인이 있었습니다. 한꺼번에 해결하려 하면 뭐가 효과가 있었는지 알 수 없고, 각 원인을 분리해서 하나씩 대응한 것이 결과적으로 효과적이었습니다.
