---
title: 검색 렌더링 최적화 과정 - memo, useTransition, 점진적 로딩까지
date: 2026-04-06
tags: [React, Performance, memo, useTransition, IntersectionObserver, Optimization]
references:
  - title: useTransition
    url: https://kwonjihyeon-dev.github.io/TIL/react/hooks/useTransition
---

# 검색 렌더링 최적화 과정

## 문제 상황

검색 화면에서 키워드를 입력하면 **입력이 느려지는 현상**이 발생했다.
예를 들어 "사당"을 검색한 뒤 지우고 "구로"를 입력하면, 키 입력 반응이 눈에 띄게 느렸다.

## 원인 분석

검색 결과가 약 1000개 이상 반환되는 경우, 키워드(`keyword`) state가 변경될 때마다 **전체 컴포넌트가 리렌더링**되면서 약 1000개의 결과가 다시 그려지고 있었다.

리액트를 사용하면 많이들 알고있겠지만, 불필요한 리렌더링으로 인해 React의 reconciliation(Virtual DOM diff) + DOM 업데이트가 **메인 스레드에서 동기적으로** 실행되어, 이 작업이 끝나기 전까지 다음 키 입력 이벤트 처리가 블로킹되는 것이 근본 원인이었다.

```tsx
// 문제 코드: keyword 변경 → 전체 리렌더링 → 약 1000개 리스트 재렌더링
export const SearchSheet = () => {
    const [keyword, setKeyword] = useState('');
    const { data: results = [] } = useSearchQuery(debouncedKeyword);

    return (
        <>
            <input onChange={(e) => setKeyword(e.target.value)} />
            <ul>
                {results.map((item) => (
                    <li>...</li> // keyword가 바뀔 때마다 약 1000개 모두 리렌더링
                ))}
            </ul>
        </>
    );
};
```

## 최적화 과정

### 1단계: 리스트 컴포넌트 분리 + React.memo

입력 필드와 결과 리스트가 같은 컴포넌트에 있어서, `keyword` state 변경이 리스트까지 리렌더링을 유발했다.

**해결**: 리스트를 `memo`로 감싼 별도 컴포넌트로 분리하여, `results`와 `debouncedKeyword`가 변경될 때만 리렌더링되도록 했다.

```tsx
const ResultList = memo(({ results, keyword, onSelect }) => {
    return (
        <ul>
            {results.map((item) => (
                <li>...</li>
            ))}
        </ul>
    );
});

export const SearchSheet = () => {
    const [keyword, setKeyword] = useState('');

    return (
        <>
            <TextFields onChange={(e) => setKeyword(e.target.value)} />
            {/* keyword가 바뀌어도 results가 같으면 리렌더링 안 됨 */}
            <ResultList results={results} keyword={debouncedKeyword} onSelect={handleSelect} />
        </>
    );
};
```

**결과**: 입력 중 리스트 리렌더링은 줄었지만, API 응답이 돌아올 때 여전히 약 1000개를 한 번에 렌더링하는 비용이 남아있었다.

### 2단계: useTransition 시도 (실패)

React 19의 `useTransition`으로 리스트 업데이트를 낮은 우선순위로 처리하려 했다.

```tsx
const [keyword, setKeyword] = useState('');
const [displayKeyword, setDisplayKeyword] = useState('');
const [, startTransition] = useTransition();

const handleKeywordChange = (value: string) => {
    setKeyword(value); // 높은 우선순위: 입력 즉시 반영
    startTransition(() => setDisplayKeyword(value)); // 낮은 우선순위: 리스트 업데이트
};

const debouncedKeyword = useDebounce(displayKeyword, 300);
```

**문제점**: `useTransition` 자체는 버벅임 해결에 효과적인 도구다. 낮은 우선순위로 표시된 렌더링 도중 키 입력이 들어오면 React가 렌더링을 **중단하고 입력을 먼저 처리**할 수 있기 때문이다.

하지만 이번에는 **적용 위치가 잘못**되었다. `useTransition`을 `displayKeyword`(debounce의 입력값)에 적용하면서, transition이 상태 업데이트 타이밍을 지연시켜 debounce의 타이머 기준점이 흔들렸다:

1. `displayKeyword = "사"` → transition으로 지연되어 debounce에 **늦게** 들어감
2. `displayKeyword = "사당"` → 마찬가지로 지연
3. debounce 입장에서는 입력 타이밍이 실제와 달라져 의도한 대로 동작하지 않음

**핵심**: `useDebounce`는 "얼마나 자주 실행할지"를 제어하고, `useTransition`은 "실행될 때 렌더링 우선순위를 어떻게 할지"를 제어한다. **역할 자체는 다르지만**, transition을 debounce의 **입력값**에 적용하면 debounce가 판단하는 시간 간격이 왜곡되어 함께 사용하기 어렵다.

올바르게 사용하려면 `useTransition`을 debounce 입력이 아닌 **결과 렌더링 쪽**에 적용해야 한다:

```tsx
// ✅ 결과 렌더링에 transition을 적용하면 효과적
const [displayResults, setDisplayResults] = useState([]);
const [, startTransition] = useTransition();

useEffect(() => {
    startTransition(() => setDisplayResults(results));
}, [results]);
```

**결론**: 이번에는 적용 위치 문제로 `useTransition`을 제거했다. 올바른 위치에 적용하면 효과가 있을 수 있으므로, 위치를 바꿔서 다시 시도해본다.

### 3단계: useTransition 올바른 위치에 적용 (효과 미미)

2단계의 실패 원인이 적용 위치였으므로, debounce 입력이 아닌 **결과 렌더링 쪽**에 `useTransition`을 적용했다.

```tsx
const { data: results = [] } = useSearchQuery(debouncedKeyword);
const [displayResults, setDisplayResults] = useState([]);
const [isPending, startTransition] = useTransition();

useEffect(() => {
  startTransition(() => setDisplayResults(results));
}, [results]);

// displayResults를 렌더링
<ResultList results={displayResults} ... />
```

**결과**: debounce와의 충돌은 해결되었지만, 약 600개 이상의 결과가 한 번에 렌더링될 때 여전히 블로킹이 발생했다. `useTransition`은 렌더링을 **중단 가능**하게 만들 뿐, 대량 DOM 생성 비용 자체를 줄이지는 못하기 때문이다.

### 4단계: 점진적 로딩 (IntersectionObserver)

근본 원인은 **약 1000개를 한 번에 DOM에 올리는 것**이었다. 화면에 보이는 만큼만 렌더링하고, 스크롤 시 다음 항목을 추가하는 방식으로 변경했다.

```tsx
const PAGE_SIZE = 50;

const ResultList = memo(({ results, keyword, onSelect }) => {
    const [visibleCount, setVisibleCount] = useState(PAGE_SIZE);
    const observerRef = useRef<HTMLDivElement>(null);

    // 결과 변경 시 초기화
    useEffect(() => {
        setVisibleCount(PAGE_SIZE);
    }, [results]);

    const loadMore = useCallback(() => {
        setVisibleCount((prev) => Math.min(prev + PAGE_SIZE, results.length));
    }, [results.length]);

    // 하단 감지
    useEffect(() => {
        const el = observerRef.current;
        if (!el) return;

        const observer = new IntersectionObserver(
            ([entry]) => {
                if (entry?.isIntersecting) loadMore();
            },
            { threshold: 0.1 }
        );

        observer.observe(el);
        return () => observer.disconnect();
    }, [loadMore]);

    const visibleResults = results.slice(0, visibleCount); // slice 비용: O(n)이지만, 참조만 복사하는 단순 메모리 연산으로 600개는 영향이 없는 수준

    return (
        <ul>
            {visibleResults.map((item) => (
                <li>...</li>
            ))}
            {visibleCount < results.length && <div ref={observerRef} className="h-1" />}
        </ul>
    );
});
```

**결과**: 초기 렌더링 시 50개만 DOM에 올라가고, 스크롤할 때마다 50개씩 추가된다. 입력 반응이 즉각적으로 개선되었다.

## 최종 구조

```
키 입력 → keyword (즉시 반영, 입력 필드만 리렌더링)
       → useDebounce(300ms) → debouncedKeyword
       → useQuery (API 요청)
       → results 변경 → ResultList (memo) 리렌더링
       → 처음 50개만 렌더링 → 스크롤 시 50개씩 추가
```

## 다시 상기시킨 부분

1. **`memo`는 리렌더링 범위를 줄이지만, 대량 DOM 생성 비용은 해결 못 한다**
2. **`useTransition`은 적용 위치가 중요하다** — `useDebounce`는 실행 빈도를, `useTransition`은 렌더링 우선순위를 제어한다. 역할은 다르지만, transition을 debounce의 입력값에 적용하면 타이밍이 왜곡된다. 결과 렌더링 쪽에 적용해야 의도대로 동작한다.
3. **`useTransition`은 렌더링을 중단 가능하게 만들 뿐, DOM 수 자체를 줄이지는 못한다** — 점진적 로딩으로 DOM 수가 충분히 줄어들면 useTransition의 효과는 체감하기 어렵다
4. **점진적 로딩이 가장 효과적** — DOM 수를 제한하는 것이 근본 해결. 라이브러리(`@tanstack/react-virtual`) 없이 IntersectionObserver만으로 충분히 구현 가능하다
5. **최적화는 단계적으로** — memo → useTransition(위치 실패 → 위치 수정 → 효과 미미) → 점진적 로딩 순서로 시도하며 각 방법의 한계를 확인한 뒤 적절한 해결책을 찾았다
