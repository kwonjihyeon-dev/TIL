---
title: React의 상태 관리와 이벤트 루프
date: 2026-02-12
tags: [React, Memory Heap, Fiber, setState, Event Loop, SPA Routing]
references:
    - title: JavaScript 이벤트 루프 구조
      url: https://kwonjihyeon-dev.github.io/TIL/cs/javascript-event-loop
---

# React의 상태 관리와 이벤트 루프

React는 각 컴포넌트를 **Fiber Node**라는 내부 자료구조로 관리하며, `useState`의 값은 Fiber Node의 `memoizedState`에 저장된다.

```
Memory Heap
├── React Fiber 트리 (React 내부 자료구조)
│   ├── FiberNode (컴포넌트 A)
│   │   └── memoizedState → { count: 0 }   ← useState의 state
│   ├── FiberNode (컴포넌트 B)
│   │   └── memoizedState → { name: 'hello' }
│   └── ...
├── 일반 변수, 객체
├── 클로저에 캡처된 값
└── ...
```

## setState 호출 시 일어나는 일

`setState`는 즉시 state를 바꾸지 않는다. update queue에 요청을 쌓고, React 스케줄러가 macrotask로 처리한다.

```
setState 호출 (동기)
│
├── 1. Update 객체 생성 → Fiber Node의 update queue에 추가
│       { action: newValue } → queue에 enqueue
│
├── 2. React 스케줄러에 리렌더링 등록 (MessageChannel = macrotask)
│
└── 동기 코드 계속 실행 (setState 이후 코드)

        ↓ (현재 macrotask 종료 → microtask 소진 → 렌더링 → 다음 macrotask)

[스케줄러의 macrotask 실행]
│
├── 3. update queue에서 대기 중인 업데이트를 꺼냄
│      이전 state (memoizedState) + update → 새 state 계산
│
├── 4. 새 state로 컴포넌트 함수 재실행 (reconciliation)
│      이전 Virtual DOM vs 새 Virtual DOM 비교 (diffing)
│
└── 5. 변경된 부분만 실제 DOM에 반영 (commit)
       memoizedState를 새 state로 교체
```

### setState 직후 state를 읽으면 이전 값인 이유

```jsx
const [count, setCount] = useState(0);

const handleClick = () => {
    setCount(1); // update queue에 { action: 1 } 추가 + macrotask 등록
    console.log(count); // 0 (아직 macrotask 실행 전이므로 이전 값)
};
```

`setState`는 동기 코드에서 update queue에 요청만 쌓고, 실제 state 계산과 DOM 반영은 스케줄러의 macrotask에서 실행된다. 따라서 `setState` 직후에는 아직 이전 값이다.

### 이벤트 루프 관점

```
[사용자 클릭 (macrotask)]
    → setState → update queue에 추가 + macrotask 등록
    → [microtask 소진]
    → [브라우저 렌더링] ← 아직 state 변경 전이므로 이전 화면 유지

[React 스케줄러 macrotask 실행]
    → 3. 새 state 계산
    → 4. reconciliation (diffing)
    → 5. commit (실제 DOM 반영)
    → [microtask 소진]
    → [브라우저 렌더링] ← 여기서 변경된 DOM이 화면에 반영
```

React가 commit 단계에서 실제 DOM을 수정하고, 그 다음 브라우저 렌더링 단계에서 변경사항이 화면에 **페인트**된다. DOM 수정과 화면 페인트는 별개의 단계이다.

## SPA 라우터 이동 시 과정

`router.push`도 내부적으로는 `setState`로 라우트 상태를 변경하는 것이므로 동일한 흐름을 따른다.

```
[사용자 클릭 (macrotask)]
    → router.push('/search')
    → history.pushState() 호출 (URL 변경, 동기)
    → React setState로 라우트 상태 업데이트 + macrotask 등록
    → [microtask 소진]
    → [브라우저 렌더링] ← 아직 이전 페이지 화면

[React 스케줄러 macrotask 실행]
    → 이전 페이지 컴포넌트 unmount
    → 새 페이지 컴포넌트 mount (컴포넌트 함수 실행)
    → reconciliation (diffing)
    → commit (실제 DOM 반영 — 새 페이지의 DOM으로 교체)
    → [microtask 소진]
    → [브라우저 렌더링] ← 새 페이지가 화면에 표시

[새 페이지의 useEffect (macrotask)]
    → autoFocus → .focus() 호출
```

사용자 클릭 시점과 새 페이지의 DOM 반영 사이에 **최소 2번의 macrotask 경계**가 있다.

## 정리

| 단계                          | 실행 위치                | 설명                                                             |
| ----------------------------- | ------------------------ | ---------------------------------------------------------------- |
| setState / router.push        | 동기 (현재 macrotask)    | update queue에 요청 추가, 스케줄러에 macrotask 등록              |
| state 계산 + diffing + commit | React 스케줄러 macrotask | Heap의 Fiber Node에서 이전 state를 읽고 새 state 계산 → DOM 반영 |
| 화면 페인트                   | 브라우저 렌더링          | commit에서 수정된 DOM을 실제 화면에 그림                         |
| useEffect                     | 별도 macrotask           | DOM 반영 이후 실행, 이미 여러 macrotask 경계를 지남              |
