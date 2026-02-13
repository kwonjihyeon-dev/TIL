---
title: JavaScript 이벤트 루프 구조
date: 2026-02-12
tags: [JavaScript, Event Loop, macrotask, microtask, 비동기]
references:
    - title: React의 상태 관리와 이벤트 루프
      url: https://kwonjihyeon-dev.github.io/TIL/react/react-state-with-event-loop
---

# JavaScript 이벤트 루프 구조

## JavaScript는 싱글 스레드

JavaScript는 **하나의 호출 스택(Call Stack)** 만 가진 싱글 스레드 언어다. 한 번에 하나의 작업만 실행할 수 있다. 그럼에도 비동기 처리가 가능한 이유는 **이벤트 루프(Event Loop)** 가 있기 때문이다.

## 전체 구조

```
┌─────────────────────────────────────────────────────┐
│                    JavaScript 엔진                    │
│  ┌──────────────┐    ┌───────────────────────────┐  │
│  │  Call Stack   │    │       Memory Heap          │  │
│  │              │    │  (객체, 변수 저장)           │  │
│  │  함수 실행    │    │                             │  │
│  │  컨텍스트     │    │                             │  │
│  └──────────────┘    └───────────────────────────┘  │
└─────────────────────────────────────────────────────┘
            ↑ 실행할 작업을 꺼냄
            │
┌───────────────────────────────────────────┐
│              이벤트 루프 (Event Loop)        │
│                                             │
│  1. Call Stack이 비었는가?                   │
│  2. Microtask 큐에 작업이 있는가? → 전부 실행 │
│  3. 렌더링이 필요한가? → 렌더링               │
│  4. Macrotask 큐에서 1개 꺼내 실행            │
│  5. 1번으로 돌아감                            │
└───────────────────────────────────────────┘
            ↑ 작업을 가져옴
            │
┌──────────────────┐  ┌──────────────────┐
│  Microtask Queue  │  │  Macrotask Queue  │
│  (우선순위 높음)   │  │  (우선순위 낮음)   │
│                    │  │                    │
│  Promise.then     │  │  setTimeout        │
│  queueMicrotask   │  │  setInterval       │
│  MutationObserver │  │  MessageChannel    │
│                    │  │  I/O 콜백          │
│                    │  │  UI 이벤트 (click) │
└──────────────────┘  └──────────────────┘
```

## 이벤트 루프 실행 순서

```
[macrotask 1개 실행] → [microtask 큐 전부 소진] → [브라우저 렌더링 (필요 시)] → [macrotask 1개 실행] → ...
```

핵심 규칙:

1. **macrotask는 한 번에 1개**만 실행한다
2. **microtask는 큐가 빌 때까지 전부** 실행한다 (microtask 안에서 microtask를 추가해도 계속 실행)
3. 브라우저 렌더링은 macrotask와 macrotask 사이에 필요할 때 수행된다

## Call Stack (호출 스택)

현재 실행 중인 함수들이 쌓이는 곳이다. LIFO(후입선출) 구조로, 가장 마지막에 들어온 함수가 먼저 실행 완료된다.

```js
function a() {
    b();
    console.log('a');
}
function b() {
    console.log('b');
}
a();

// Call Stack 변화:
// [a] → [a, b] → [a] → [] (빈 스택)
// 출력: b → a
```

Call Stack이 비어야 이벤트 루프가 다음 작업(microtask 또는 macrotask)을 가져온다.

## Macrotask (Task)

브라우저 또는 Node.js 환경이 이벤트 루프에 등록하는 작업 단위다.

| API                          | 설명                            |
| ---------------------------- | ------------------------------- |
| `setTimeout` / `setInterval` | 타이머 기반 지연 실행           |
| `MessageChannel`             | 메시지 포트 간 통신             |
| `requestAnimationFrame`      | 다음 렌더링 프레임 전에 실행    |
| I/O 콜백                     | 네트워크 요청, 파일 읽기 등     |
| UI 이벤트                    | `click`, `scroll`, `keydown` 등 |

**특징**: 한 사이클에 **1개만** 실행되고, 이후 microtask 큐를 처리한다.

## Microtask

현재 실행 중인 작업이 끝난 직후, 다음 macrotask로 넘어가기 전에 실행되는 작업이다.

| API                                  | 설명                           |
| ------------------------------------ | ------------------------------ |
| `Promise.then` / `catch` / `finally` | Promise 후속 처리              |
| `queueMicrotask`                     | 명시적 microtask 등록          |
| `MutationObserver`                   | DOM 변경 감지 콜백             |
| `async/await`                        | await 이후 코드 (Promise 기반) |

**특징**: 큐에 있는 **전부를 소진**할 때까지 다음 단계로 넘어가지 않는다.

## 실행 순서 예시

### 기본 예시

```js
console.log('1: 동기');

setTimeout(() => {
    console.log('5: macrotask');
}, 0);

Promise.resolve()
    .then(() => console.log('3: microtask 1'))
    .then(() => console.log('4: microtask 2'));

console.log('2: 동기');

// 출력: 1 → 2 → 3 → 4 → 5
```

실행 흐름:

1. 동기 코드 실행: `1`, `2` 출력
2. Call Stack이 비면 microtask 큐 소진: `3`, `4` 출력
3. macrotask 큐에서 1개 실행: `5` 출력

### microtask 안에서 microtask 추가

```js
setTimeout(() => console.log('3: macrotask'), 0);

Promise.resolve().then(() => {
    console.log('1: microtask');
    queueMicrotask(() => console.log('2: 중첩 microtask'));
});

// 출력: 1 → 2 → 3
```

microtask 내에서 추가된 microtask도 **같은 사이클에서 전부 실행**된 후 macrotask로 넘어간다.

### 복합 예시

```js
console.log('1');

setTimeout(() => {
    console.log('6');
    Promise.resolve().then(() => console.log('7'));
}, 0);

Promise.resolve().then(() => {
    console.log('3');
    setTimeout(() => console.log('8'), 0);
});

queueMicrotask(() => console.log('4'));

console.log('2');

setTimeout(() => console.log('5'), 0);

// 출력: 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8
```

실행 흐름:

1. **동기**: `1`, `2`
2. **microtask 큐 소진**: `3` (Promise), `4` (queueMicrotask)
3. **macrotask 1개**: `5` (첫 번째 setTimeout... 아닌 순서대로)

> 실제로는 setTimeout 등록 순서에 따라 `5`가 먼저일 수도 있고 `6`이 먼저일 수도 있다. 핵심은 macrotask 1개 실행 후 microtask 큐를 소진하는 패턴이다.

## requestAnimationFrame의 위치

`requestAnimationFrame`(rAF)은 macrotask도 microtask도 아닌 **별도의 콜백 큐**에 속한다. 브라우저 렌더링 직전에 실행된다.

```
[macrotask] → [microtask 전부] → [rAF 콜백] → [브라우저 렌더링] → [macrotask] → ...
```

```js
setTimeout(() => console.log('3: macrotask'), 0);
requestAnimationFrame(() => console.log('2: rAF'));
Promise.resolve().then(() => console.log('1: microtask'));

// 일반적 출력: 1 → 2 → 3
// (단, rAF와 macrotask 순서는 브라우저 구현에 따라 다를 수 있음)
```

## async/await와 이벤트 루프

`async/await`는 Promise의 문법적 설탕(syntactic sugar)이다. `await` 이후 코드는 microtask로 실행된다.

```js
async function foo() {
    console.log('1: 동기');
    await Promise.resolve();
    console.log('3: microtask (await 이후)');
}

foo();
console.log('2: 동기');

// 출력: 1 → 2 → 3
```

`await` 이전까지는 동기 실행이고, `await` 이후 코드는 `Promise.then`과 동일하게 microtask 큐에 등록된다.

## 정리

| 구분      | macrotask                  | microtask                    |
| --------- | -------------------------- | ---------------------------- |
| 실행 시점 | 이벤트 루프 사이클 시작    | macrotask 실행 직후          |
| 실행 개수 | 한 사이클에 **1개**        | 큐가 빌 때까지 **전부**      |
| 우선순위  | 낮음                       | 높음                         |
| 대표 API  | setTimeout, MessageChannel | Promise.then, queueMicrotask |
| 렌더링    | macrotask 사이에 발생      | 렌더링 전에 전부 처리        |
