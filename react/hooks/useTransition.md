---
layout: page
title: useTransition
---

# useTransition

concurrent rendering 을 지원하는 react hook

### 기존 동작 방식 (동기 렌더링, synchronous rendering)

- React와 같은 UI 라이브러리나 프레임워크가 브라우저의 메인 스레드에서 모든 작업(렌더링, 업데이트, 이벤트 처리 등)을 순차적으로 처리하는 방식<br/>
  ex. 사용자의 검색으로 대량의 데이터가 렌더링되어야하는 경우, 해당 작업(렌더링)을 수행하기 위해 사용자의 입력 이벤트도 처리되지않는 경우<br/>
  이 때 사용자의 경험을 해치지 않기 위해 대안으로 <u>Debounce</u> 등을 사용한다.

### Concurrent rendering

- 메인 스레드를 차단하지 않고 백그라운드에서 상태를 업데이트할 수 있다. 대규모 렌더링 작업 중에도 사용자 입력을 즉시 처리해 사용자 경험을 향상시킬 수 있다.

### useTransition

```js
const [isPending, startTransition] = useTransition();
```

- isPending : 대기 중인 Transition 이 있는 지 체크
- startTransition : state 업데이트를 Transition 으로 표시한다.<br/>startTransition 내부에 실행되는 로직은 낮은 우선순위로 분류되어 사용자 입력과 같은 작업 이후에 처리된다.
  <br/>
  <br/>

예시 코드

```js
import { useState, useTransition } from "react";

function TabContainer() {
  const [count, setCount] = useState(0);
  const [isPending, startTransition] = useTransition();

  // props로 받은 set 함수를 실행해 state를 업데이트
  startTransition(() => {
    setCount(1); // 업데이트 예약된 state
  });
}
```
