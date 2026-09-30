---
title: Next.js App Router 동적 렌더링과 페이지 전환 지연
date: 2026-09-29
tags: [Next.js, App Router, RSC, Prefetch, Dynamic Rendering, Suspense, cacheComponents]
references:
  - title: Nextjs가 느린 이유 1 - 동적 RSC와 prefetch (velog)
    url: https://velog.io/@k-svelte-master/nextjs-slow-1-dynamic-rsc-prefetch
  - title: Next.js Link 문서
    url: https://nextjs.org/docs/app/api-reference/components/link
---

# Next.js App Router 동적 렌더링과 페이지 전환 지연

> 최근 Nextjs가 느린 이유 1 - 해당 글을 읽으면서 내가 이해하기 쉽게 정리한 개념들을 순서대로 기록하고(1부), 운영 중인 서비스에 대입해 봤다(2부).
> 참고 글은 Next 16.x 기준이다. `cacheComponents`, `partialPrefetching`을 제외한 내용은 Next 15에도 해당한다.

---

# 1부. 개념 정리

## 1. RSC가 줄이는 것

### 헷갈렸던 점

- 라우트별로 코드 스플리팅을 하면 번들 문제는 해결된 것 아닌가?
- SSR을 하면 서버가 화면을 그려 주니까, SPA보다 브라우저가 할 일이 적은 것 아닌가?

### 정리

**코드 스플리팅은 현재 라우트가 아닌 라우트의 코드를 초기 번들에서 제외한다.** `/posts`에 들어갈 때 `/settings` 코드는 받지 않는다. `/posts` 화면을 이루는 컴포넌트(헤더, 목록, 카드, 푸터 등)의 코드는 전부 받는다.

**코드를 받은 뒤에도 브라우저 작업이 남는다.**

1. 컴포넌트 함수를 전부 실행한다.
2. 그 결과로 React 엘리먼트 객체 트리(가상 DOM)를 만든다.
3. 실제 DOM과 맞춘다. SSR 이후라면 이 과정이 하이드레이션이다.

이 작업은 네트워크와 관계없이 메인 스레드의 CPU 시간을 쓴다. 60fps로 화면을 갱신하려면 한 번에 쓸 수 있는 시간은 약 16.7ms(1000ms ÷ 60)다. 이 작업이 길게 이어지면 그동안 프레임이 빠지고, 스크롤이 끊기거나 클릭에 반응하지 않는다.

**SSR도 브라우저 작업량은 SPA와 거의 같다.** SSR은 HTML을 먼저 보여 줄 뿐이고, 버튼이 눌리려면 SPA와 같은 양의 컴포넌트 코드를 받아 전부 실행해야 한다.

| | SPA | SSR + 하이드레이션 |
|---|---|---|
| 첫 화면이 보이는 시점 | JS 실행 후 | HTML 도착 즉시 |
| 브라우저에서 컴포넌트 함수 실행 | 전부 | 전부 |
| 가상 DOM 트리 생성 | 전부 | 전부 |
| 실제 DOM 생성 | 브라우저가 새로 만듦 | 서버 HTML로 만든 DOM 재사용 |
| 클릭 가능한 시점 | 실행 완료 후 | 하이드레이션 완료 후 |

SSR이 SPA보다 브라우저에서 덜 하는 작업은 실제 DOM 생성이다. 컴포넌트 함수 실행과 가상 DOM 생성 작업은 동일하다.

**RSC는 훅과 이벤트 핸들러가 없는 컴포넌트를 서버에서 실행하고 결과만 브라우저로 보낸다.**
- 그 컴포넌트의 코드가 번들에서 빠진다. (다운로드 감소)
- 브라우저에서 그 컴포넌트 함수를 실행하지 않는다. (실행 감소)
- 단, 서버가 보낸 결과(`div`, `ul` 같은 태그 트리)를 브라우저에서 React 트리로 만드는 과정은 브라우저에서 실행된다.

---

## 2. RSC Payload

서버 컴포넌트를 실행한 결과를 직렬화한 데이터다. HTML이 아니라 React가 정한 줄 단위 텍스트 형식이다.

```
1:I["./app/posts/like-button.tsx",["static/chunks/app/posts/page.js"],"LikeButton"]
2:["$","main",null,{"children":[["$","h1",null,{"children":"글 목록"}], ..., ["$","$L1",null,{"postId":1}]]}]
```

| 내용 | 예시 |
|---|---|
| 서버 컴포넌트가 렌더한 태그와 props | `2:` 줄 |
| 클라이언트 컴포넌트 참조 (모듈 위치, 청크 목록, export 이름). 코드는 없음 | `1:I[...]` |
| 클라이언트 컴포넌트에 넘길 props | `{"postId":1}` |
| 나중에 채울 자리 (Suspense 안쪽 등) | `$L1` |

- 서버 컴포넌트가 클라이언트 컴포넌트에 넘기는 props는 직렬화할 수 있어야 한다. 일반 함수는 넘길 수 없고 Server Action만 참조로 넘길 수 있다.
- 서버 컴포넌트 자리는 이 Payload가 도착해야 채워진다. 채워지지 않은 자리가 있으면 React는 새 트리를 화면에 올리지 않는다.

| 이동 방식 | 받는 것 |
|---|---|
| URL 직접 접근, 새로고침 | HTML + HTML 안에 들어 있는 Payload (`self.__next_f.push(...)`) |
| `<Link>` 클릭, `router.push()` | Payload만. 요청 헤더에 `rsc: 1` |

---

## 3. `<Link>` prefetch는 자동인가, 목적지는 어떻게 아는가

`<Link>`를 쓰면 기본으로 prefetch가 켜져 있다. 따로 설정하지 않아도 된다.

1. `<Link>`가 마운트되면 DOM 요소를 IntersectionObserver에 등록한다.
2. 뷰포트에 들어오면 `href`의 URL로 prefetch를 요청한다. `href`에는 값이 채워진 URL(`/posts/1`)이 들어오므로 목적지를 따로 해석하지 않는다.
3. 서버에 그 URL의 RSC Payload를 요청한다. 헤더에 `rsc: 1`, `next-router-prefetch: 1`이 붙는다.
4. Payload 안의 클라이언트 컴포넌트 참조(`1:I[...]`)에서 필요한 청크 목록을 알아내고 받는다.

App Router는 어떤 청크가 필요한지 서버 응답을 받아서 안다. Pages Router는 빌드 때 만든 라우트-청크 목록(`_buildManifest.js`)을 브라우저가 미리 가지고 있었다.

- `next dev`에서는 prefetch를 하지 않는다. 프로덕션 빌드(`next build && next start`)에서만 동작한다. 로컬 개발 서버에서 느낀 전환 속도는 운영과 다르다.
- `<link rel="preload">`는 현재 페이지에 곧 필요한 리소스를 받는 브라우저 기능과 별개인 "prefetch" 기능이다.

---

## 4. `router.push()`는 prefetch가 되는가

되지 않는다. 호출하는 순간 요청을 시작한다.

| 경우 | prefetch |
|---|---|
| `<Link>` | 뷰포트에 들어오면 자동 |
| `router.push()` | 없음. 같은 URL을 다른 `<Link>`가 이미 prefetch했다면 그 캐시를 씀 |
| `<Link prefetch={false}>` | 없음 |
| 뷰포트에 들어오기 전에 클릭 | 아직 시작 안 됨 |
| `<a href>` | 클라이언트 이동이 아니라 페이지 전체를 새로 불러옴 |

미리 받으려면 `router.prefetch()`를 따로 부른다.

```tsx
const router = useRouter();

useEffect(() => {
  router.prefetch('/posts');
}, [router]);

const handleClick = () => {
  track('view_all_click');
  router.push('/posts');
};
```

단순 이동인데 `onClick` + `router.push()`로 작성했다면 `<Link>`로 바꾸는 게 낫다. `<Link>`는 prefetch, 새 탭 열기(Cmd+클릭), `<a>` 태그 접근성을 기본으로 지원한다. 트래킹은 `<Link onClick>`로 가능하다.

---

## 5. 동적 렌더링 판정

### 두 가지 "동적"

- **동적 세그먼트**: `[id]`, `[...slug]` 폴더 규칙. URL 일부를 변수로 받는다는 뜻이고 렌더링 방식과는 관계없다. `generateStaticParams`를 쓰면 정적 라우트다.
- **동적 렌더링**: 빌드 때가 아니라 요청 때 렌더하는 것. `next build` 결과에서 `○ (Static)`, `ƒ (Dynamic)`으로 구분된다.

### 판정 조건

서버 컴포넌트가 **렌더 중에** 아래 API를 호출하면, `cacheComponents`가 없을 때 그 라우트 전체가 동적 렌더링이 된다.

- `cookies()`, `headers()`
- `searchParams` props
- `connection()`, `draftMode()`
- `fetch(url, { cache: 'no-store' })`
- `export const dynamic = 'force-dynamic'`

- 판정 단위는 컴포넌트가 아니라 라우트(URL)다. 최상위 레이아웃이든 말단 컴포넌트 하나든, 호출하는 곳이 있으면 라우트 전체가 동적이다.
- 파일에 코드가 있다고 판정되는 게 아니라, 빌드 때 렌더해 보면서 실제로 호출되면 동적 렌더링으로 판정한다.
- Server Action, Route Handler(`route.ts`) 안에서 호출하는 것은 페이지 판정에 영향이 없다.

### 동적 라우트의 prefetch와 캐시

1. **prefetch**: 공통 레이아웃과 `loading.js` 경계까지만 미리 받고, 페이지 본문의 Payload는 받지 않는다. `loading.js`가 없으면 본문 쪽으로 받는 게 거의 없다. 청크 목록도 Payload로 오기 때문에, 본문에 있는 클라이언트 컴포넌트의 청크도 클릭 후에 알게 된다(동작 원리에서 추론한 내용. 버전별로 Network 탭 확인 필요).
2. **Client Cache**: `experimental.staleTimes.dynamic` 기본값이 0초다(14.2에서는 30초, 15에서 0초로 바뀜). 방문했던 페이지도 다시 가면 새로 받는다.

> `staleTimes`는 미리 받는 기능이 아니다. 미리 받는 건 prefetch고, `staleTimes`는 받아 둔 Payload를 브라우저 메모리에 보관하는 시간이다.

```
클릭 → RSC 요청 → (네트워크 왕복 + 서버 렌더 + 서버 안의 API 대기) → 응답 → 화면 변경
       └──────────────── 이 동안 이전 화면이 멈춰 있음 ────────────────┘
```

### URL 직접 접근

prefetch와 관계없이 항상 서버에서 받는다. 정적과 동적의 차이는 받느냐가 아니라, 빌드 때 만든 HTML을 내주느냐 요청마다 서버가 렌더하느냐다.

---

## 6. 동적 렌더링 판정과 "페이지 본문을 다시 호출하는 것"의 차이

동적 렌더링 판정은 Payload를 만드는 시점을 정하고, 렌더 구간은 이번 이동에서 서버가 실행하는 세그먼트 범위를 정한다.

| | 동적 렌더링 여부 | 이번 이동에서 렌더하는 구간 |
|---|---|---|
| 정하는 것 | Payload를 만드는 시점 (빌드 때 / 요청 때) | 서버가 실행하는 레이아웃과 페이지의 범위 |
| 판정 단위 | 라우트 전체 | 세그먼트 |
| 정해지는 때 | 빌드 때 | 이동할 때마다 (출발과 도착 URL이 갈라지는 지점) |

클라이언트 이동에서는 공유하는 레이아웃을 서버가 다시 렌더하지 않고 갈라지는 지점 아래만 렌더한다.

그래서 이런 상황이 생긴다. 공통 레이아웃의 `cookies()`는 이번 이동에서 실행되지 않는다. 하지만 그 호출 때문에 라우트가 동적으로 판정되었고, 그 결과 페이지 본문을 미리 만들어 둘 수 없다. 요청 시점에 렌더하는 것은 동적 렌더링 판정 때문이고, 렌더 범위가 본문으로 좁혀지는 것은 출발·도착 URL의 세그먼트 비교 때문이다.

---

## 7. Suspense로 감싸면 바깥이 정적으로 남는가

```tsx
export default function PostDetail() {
  return (
    <>
      <PostHeader />
      <PostBody />
      <Suspense fallback={<LikeSkeleton />}>
        <LikeCount /> {/* cookies() 호출 */}
      </Suspense>
    </>
  );
}
```

`cacheComponents`가 없으면 남지 않는다. Suspense는 응답을 보내는 순서를 나누지만, 정적/동적 판정은 라우트 단위로 한다.

| Suspense가 나누는 것 | Suspense가 나누지 않는 것 |
|---|---|
| 응답을 보내는 순서 (바깥 먼저, 안쪽 나중) | 정적/동적 판정 (라우트 전체 기준) |
| 느린 API 하나가 나머지를 막지 않게 함 | 바깥도 요청 때 렌더되고, 미리 받거나 캐시할 수 없음 |

- 첫 조각도 요청이 와야 만들어지므로 클릭 후 왕복이 한 번 필요하다. fallback도 그 응답 안에 있어서 왕복 전에는 그릴 수 없다.
- `loading.js`는 그 세그먼트의 `page`를 Next.js가 자동으로 Suspense로 감싼 것이고 prefetch 대상이다. 페이지 안에 직접 둔 Suspense는 prefetch 대상이 아니다.

### 서버 렌더링 판정과 관계없는 Suspense·dynamic 패턴

| 패턴 | 다루는 문제 |
|---|---|
| 서버 컴포넌트를 Suspense로 감싸 느린 섹션 격리 | 응답 순서. 이 용도로는 동작한다 |
| `next/dynamic` + `ssr: false` (차트 라이브러리 등) | JS 번들 크기. RSC 판정과 관계없다 |
| 클라이언트 컴포넌트 안의 Suspense | 클라이언트 데이터 로딩. 서버 렌더링 판정과 관계없다 |

---

## 8. 서버 데이터가 필요한 본문은 Suspense가 필수인가, 아무 내용도 없는 빈 화면이 보이는가

아무 내용도 없는 빈 화면은 보이지 않는다. React는 비어 있는 트리를 화면에 올리지 않는다. Suspense 경계가 있으면 fallback이 보이고, 없으면 이전 화면을 그대로 두고 기다린다.

아무 내용도 없는 빈 화면이 보인다면 앱 코드가 그렇게 그린 것이다. 예: 클라이언트 컴포넌트가 `useQuery`의 `isLoading` 동안 `null`을 반환.

`cacheComponents`를 켜면 요청 시점 데이터를 읽는 컴포넌트마다 셋 중 하나를 정해야 한다. 정하지 않으면 빌드 에러("Uncached data outside of Suspense")가 난다.

| 선택 | 클릭했을 때 |
|---|---|
| `<Suspense>`로 감싸기 | 정적 셸과 fallback이 즉시 보이고 본문은 도착하면 교체 |
| `'use cache'` 붙이기 | 결과가 캐시되어 정적 셸에 포함. 사용자별 데이터(쿠키 기반)에는 보통 쓸 수 없음 |
| 기다리겠다고 명시 | 이전 화면이 멈춘 채 응답을 기다림 |

"네트워크 요청 없이 보여줄 수 있는 것부터 바로 보여준다"를 원하면, 서버 데이터가 필요한 본문은 Suspense로 감싸야 한다.

---

## 9. `loading.tsx`와 `cacheComponents`를 쓰면 흐름이 어떻게 바뀌는가

용어 정리부터.
- 클릭할 때 보내는 요청은 prefetch가 아니다. prefetch는 클릭 전(뷰포트 진입 시)에 보낸다.
- 클라이언트 이동에서 서버가 돌려주는 건 HTML이 아니라 RSC Payload다.
- `loading.tsx`와 `cacheComponents`는 따로 동작한다. 하나만 있어도 즉시 전환된다.

**둘 다 없음**
```
뷰포트 진입 → prefetch (본문 없음)
클릭 → 이동 요청 → 서버 렌더 + API 대기 → Payload 도착 → 화면 전환
       └──────────── 이 동안 이전 화면이 멈춰 있음 ────────────┘
```

**`loading.tsx`**
```
뷰포트 진입 → prefetch (loading.tsx까지)
클릭 → 즉시 전환 (공통 레이아웃 유지 + loading 화면)
     → 이동 요청 → Payload 도착 → 본문으로 교체
```

**`cacheComponents` + Suspense**
```
뷰포트 진입 → prefetch (정적 셸 + fallback)
클릭 → 즉시 전환 (정적 셸 + fallback)
     → 동적 부분만 요청 → 도착 → fallback 교체
```

| | `loading.tsx` | `cacheComponents` |
|---|---|---|
| 클릭 직후 보이는 것 | 공통 레이아웃 + loading 화면 (페이지 영역 전체가 loading) | 공통 레이아웃 + 페이지의 정적인 부분 + fallback |
| 클릭 후 요청 범위 | 페이지 세그먼트 전체 | Suspense 안쪽 동적 부분만 |
| 사용 조건 | Next 15에서 사용 가능 | Next 16, 데이터를 읽는 컴포넌트마다 경계 설정 |

**즉시 전환은 prefetch가 되어 있어야 한다.** `router.push()`처럼 prefetch가 안 된 이동은 `loading.tsx`가 있어도 서버 응답의 첫 조각(레이아웃 + loading 화면)이 와야 전환된다. API 대기보다는 먼저 오지만 왕복 한 번은 기다린다(스트리밍 순서에서 추론한 내용). `cacheComponents`도 셸을 클릭 후에 받아야 한다.

### `cacheComponents`의 부작용

- 비동기 작업을 암묵적으로 캐시하지 않는다. 8번의 세 가지 중 하나를 정해야 한다.
- 최근에 떠난 화면을 unmount하지 않고 React `<Activity>`로 숨긴다(`display: none`, 최대 3개). `<video>`가 계속 재생되거나, 화면을 떠났다고 가정한 코드가 어긋날 수 있다.
- 참고 글 저자는 미들웨어 비호환, 인증 세션을 읽는 레이아웃의 Suspense 경계, Activity 전환 문제로 이 옵션을 여러 번 켰다 껐다고 했다.

### `partialPrefetching` (16.3)

`/posts/1`, `/posts/2`가 같은 `/posts/[id]` 셸을 각각 따로 prefetch하던 것을, 공유하는 부분(App Shell)은 한 번만 받도록 바꾼다. 셸을 크게 만들려면 `params`를 페이지 최상단에서 `await`하지 말고 필요한 자식에서 읽는다.

### 참고 글의 추가 제안

- **Serwist 서비스 워커**: 정적 셸과 번들을 디스크에 CacheFirst로 캐시한다. RSC 요청 `rsc: 1`, prefetch 요청 `next-router-prefetch`, 응답 `x-nextjs-stale-time` 헤더로 구분한다. 배포 ID가 바뀌면 이전 캐시를 지우는 로직이 필요하다.
- **인터셉팅 라우트**: 목록에서 이동할 때(`(.)posts/[id]`)는 목록에 있는 요약을 먼저 그리고 본문만 클라이언트에서 불러온다. 직접 진입, 새로고침, 봇은 원래 라우트에서 `generateMetadata`와 스트리밍 SSR로 전체를 그린다.

---

## 10. `useLinkStatus`

`next/link`에서 제공하는 React 훅이다(Next 15.3+).

```tsx
import Link, { useLinkStatus } from 'next/link';

function PendingIndicator() {
  const { pending } = useLinkStatus();
  return pending ? <Spinner size="sm" /> : null;
}

<Link href={`/posts/${id}`}>
  <span>{title}</span>
  <PendingIndicator />
</Link>
```

- `<Link>`의 하위 컴포넌트에서 호출한다. `<Link>`를 렌더하는 컴포넌트 자신이 부르면 안 된다.
- 원인과 관계없이 이동이 끝나지 않으면 `pending: true`다. 동적 라우트의 서버 왕복, 서버 컴포넌트 안의 API 대기, 청크 다운로드 모두 해당한다.
- **이동 대기 시간은 줄지 않는다. 대기 중이라는 표시를 링크 옆에 띄운다.**
- prefetch가 되어 바로 전환되면 표시가 거의 보이지 않는다. 빠른 응답에서 깜빡임을 줄이려면 표시에 짧은 지연 애니메이션을 둔다.
- `router.push()`에는 쓸 수 없다.

| 방법 | 멈춘 시간 | 클릭 직후 보이는 것 |
|---|---|---|
| 아무것도 안 함 | 그대로 | 반응 없음 |
| `useLinkStatus` | 그대로 | 이전 화면 + 링크 옆 로딩 표시 |
| `loading.tsx` | 이전 화면이 멈추지 않음. 본문 도착 시간은 그대로 | 공통 레이아웃 + loading 화면 |
| `cacheComponents` | 이전 화면이 멈추지 않음. 클릭 후 요청 범위도 줄어듦 | 정적 셸 + fallback |

---

## 11. 이 글의 해결책은 언제 의미가 있는가

"1초 이상"처럼 시간 하나로 나누기 어렵다. 글을 두 부분으로 보면 답이 다르다.

- **동작 원리**: 지연 크기와 관계없이 적용된다. 전환이 느려졌을 때 prefetch 응답, `rsc` 요청 시간, 서버 컴포넌트 안의 API 대기 중 어디를 확인할지 알 수 있다.
- **해결책(`cacheComponents` 등)**: 왕복 지연이 체감될 때 의미가 있다. 저자는 "서울 리전이 아니면 왕복만 300ms를 넘기기도 한다"는 환경에서 체감했다.

해결책의 효과가 커지는 조건:
- 서버가 사용자와 멀다 (저자: Cloudflare에서 한국 트래픽이 LA·홍콩으로 연결)
- 서버 컴포넌트 안에서 기다리는 API가 느리다
- 목록 → 상세 이동이 반복된다 (피드, 상품 목록)
- 모바일 네트워크, 저사양 기기 사용자가 많다

`loading.tsx`, `cacheComponents` 모두 본문이 도착하는 시간을 줄이지 않는다. 클릭 직후 보여 주는 화면을 바꾼다. 응답이 짧으면 `loading.tsx`는 loading 화면이 잠깐 보였다 사라지는 깜빡임을 추가한다. 판단 기준은 프로덕션 빌드의 Network 탭에서 잰 `rsc` 요청 시간이다.

---

# 2부. 우리 서비스에서는?

> 실제 코드는 가리고 구조가 같은 예시로 바꿨다. 경로와 이름은 예시다.

## 현재 구조

**공통 인증 레이아웃에서 쿠키를 읽는다**

```tsx
// app/(protected)/layout.tsx (예시)
export default async function ProtectedLayout({ children }) {
  const cookieStore = await cookies();
  const accessToken = cookieStore.get('access_token')?.value;
  const auth = await fetchSession();

  if (!auth.isAllowed) return <Forbidden />;

  return (
    <Providers>
      <AuthInitializer accessToken={accessToken} user={auth.user} />
      {children}
    </Providers>
  );
}
```

- 인증과 권한을 레이아웃에서 한 번 확정하고 하위는 store로 구독하는 구조다.
- 이 레이아웃 아래 모든 라우트가 동적 렌더링이다. 페이지마다 따로 정할 수 없다.

**상세 페이지는 서버에서 API를 await한 뒤 렌더한다**

```tsx
// app/(protected)/posts/[id]/page.tsx (예시)
export default async function PostDetailPage({ params }) {
  const { id } = await params;
  const cookieHeader = (await cookies()).toString();

  const queryClient = new QueryClient();
  await queryClient.prefetchQuery({
    queryKey: ['post', id],
    queryFn: () => fetchPost(id, cookieHeader),
  });

  if (!queryClient.getQueryData(['post', id])) redirect('/posts');

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostDetailView id={id} />
    </HydrationBoundary>
  );
}
```

- 서버 prefetch → dehydrate → 클라이언트 캐시에 주입하는 구조다. 클라이언트에서 다시 요청하지 않는다.
- 대신 클릭 후 서버가 이 API를 기다리는 시간만큼 이전 화면이 멈춘다.

**설정**
- `loading.tsx`가 없다.
- `staleTimes`, `cacheComponents` 설정이 없다. `staleTimes.dynamic`은 기본값 0초다.
- Next 15.x다.

## 놓친 것

- **공통 레이아웃의 `cookies()` 하나로 모든 하위 라우트가 동적이다.** 페이지 단위로 정적 렌더링을 선택할 수 없다.
- **`loading.tsx`가 없어서 prefetch 응답에 본문이 없다.** 링크마다 prefetch 요청은 나가지만, 클릭 후 전환에 쓸 수 있는 내용이 들어 있지 않다.
- **같은 상세를 다시 열어도 매번 서버에 요청한다.** `staleTimes.dynamic` 0초 때문이다.
- **"전체보기" 같은 버튼 이동이 `router.push()`라 prefetch 대상이 아니다.**
  ```tsx
  <Button
    onClick={() => {
      track('view_all_click');
      router.push('/posts');
    }}
  >
    전체보기
  </Button>
  ```
- **로컬 개발 서버에서 본 전환 속도로 판단하면 안 된다.** 개발 서버는 prefetch를 하지 않는다.

## 지금 적용할 수 있는 부분

**`router.push()` 이동에 prefetch 추가**

변경이 가장 작은 방법은 기존 버튼을 두고 prefetch만 거는 것이다.

```tsx
const router = useRouter();

useEffect(() => {
  router.prefetch('/posts');
}, [router]);
```

또는 `<Link>`로 바꾸고 트래킹을 `onClick`으로 옮긴다. 이미 목록 항목 링크는 이 형태다.

```tsx
<Link href="/posts" onClick={() => track('view_all_click')}>
  전체보기
</Link>
```

디자인 시스템 버튼을 링크로 렌더할 수 있는지(`asChild` 같은 기능)는 확인이 필요하다.

지금은 전환 지연이 문제가 되는 수준이 아니라서, 이 항목은 이후 `loading.tsx`나 `cacheComponents`를 도입할 때 같이 적용해야 효과가 난다. 동적 라우트에서는 prefetch만 걸어도 본문을 받지 못하기 때문이다.

## API가 느려지면 대응할 지점

변경 범위가 작은 순서다.

1. **`useLinkStatus`**: 링크 옆에 이동 대기 표시를 띄운다. 멈춘 시간은 그대로지만 "눌렸다"는 반응을 주고 중복 클릭을 줄인다. 링크 컴포넌트 하나만 바뀐다.
2. **`loading.tsx` + 지연 노출**: 클릭 즉시 전환된다. loading 화면을 `opacity: 0`에서 시작해 CSS `animation-delay` 뒤에만 보이게 하면 빠른 응답에서는 깜빡이지 않는다. 이때 `router.push()` 이동은 prefetch를 먼저 걸어야 즉시 전환된다.
3. **서버에서 기다리는 API 개선**: 멈춘 시간 대부분이 서버 컴포넌트 안의 API 대기라면, 전환 방식보다 이 API 응답 시간을 먼저 본다.

## 현재 적용하지 않는 것

**`cacheComponents`**
- Next 16 업그레이드가 필요하다.
- 공통 레이아웃이 최상단에서 쿠키를 읽고 권한 분기를 한다. 정적 셸이 생기려면 이 구조를 Suspense 경계에 맞게 다시 짜야 한다.
- 미들웨어에서 토큰 처리를 하고 있어서, 참고 글 저자가 겪은 미들웨어·인증 레이아웃 문제와 같은 영역에 해당한다.
- 전환 지연이 문제가 되지 않는 지금은 변경 범위에 비해 얻는 게 적다.

**`loading.tsx` (지금 당장)**
- 응답이 빠른 상태에서는 loading 화면이 잠깐 보였다 사라지는 깜빡임이 생길 수 있다.

## 다시 검토할 상황

- 서버 컴포넌트 안에서 기다리는 API가 느려진 경우
- 목록 → 상세를 반복해서 여는 화면을 새로 만드는 경우 (예: 모바일 피드)
- 호스팅 위치가 사용자와 멀어지거나, 모바일 네트워크 사용자가 많은 서비스에 같은 구조를 쓰는 경우
