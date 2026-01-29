---
layout: post
title: useSyncExternalStore
---

# useSyncExternalStore

React는 useSyncExternalStore를 사용해 컴포넌트를 store에 구독한 상태로 유지하고 변경 사항이 있을 때 리렌더링한다.<br/>
컴포넌트의 최상위 레벨에서 useSyncExternalStore를 호출하여 <u>외부 데이터 저장소</u>에서 값을 불러온다.

- 외부 데이터 저장소란 ?
  - zustand, redux 와 같은 상태 라이브러리 store
  - 변경 가능한 값과 해당 값의 변경 사항을 구독하는 이벤트를 제공하는 브라우저 API (DOM, Date, 전역 변수 ... )
    <br/>

## 사용 방법

useSyncExternalStore를 사용하기 위해 두 개의 함수를 인수로 전달해야한다.

```js
// app.js
import { useSyncExternalStore } from "react";
import { todosStore } from "./todoStore.js";

function TodosApp() {
  // store에 있는 데이터의 스냅샷을 반환
  const todos = useSyncExternalStore(
    todosStore.subscribe,
    todosStore.getSnapshot
  );
  // ...
}
```

1. subscribe

- 하나의 callback 인수를 받아 store에 구독하는 함수. store를 구독하고 구독을 취소하는 함수를 반환해야한다.
- store 변경 -> callback 호출 -> React가 getSnapshot을 다시 호출 -> (필요한 경우) 컴포넌트 리렌더링

2. getSnapshot

- 컴포넌트에 필요한 store 데이터의 스냅샷을 반환하는 함수
- 저장소가 변경되지 않은 상태 : getSnapshot을 반복적으로 호출하더라도 동일한 값을 반환
- 저장소가 변경된 상태 : 반환된 값이 다르면 (`Object.is`) React는 컴포넌트를 리렌더링한다.

```js
// todoStore.js
export const todosStore = {
  addTodo() {
    todos = [...todos, { id: nextId++, text: "Todo #" + nextId }];
    emitChange();
  },
  subscribe(listener) {
    listeners = [...listeners, listener];
    return () => {
      listeners = listeners.filter((l) => l !== listener);
    };
  },
  getSnapshot() {
    return todos;
  },
};

function emitChange() {
  for (let listener of listeners) {
    listener();
  }
}
```

<br/>

[참고] https://ko.react.dev/reference/react/useSyncExternalStore
