# zustand

- 상태 관리 라이브러리.
- 불필요한 리렌더링 방지
- provider hell 방지

## 동작 원리

1. zustand는 발행/구독 모델(pub/sub) 기반으로 이루어져 있으며, 내부적으로 스토어 상태를 클로저로, 상태 변경을 구독할 리스너는 Set을 통해 관리한다.

```js
// vanilla.ts
const createStoreImpl: CreateStoreImpl = (createState) => {
  type TState = ReturnType<typeof createState>
  type Listener = (state: TState, prevState: TState) => void
  let state: TState
  const listeners: Set<Listener> = new Set()

  const setState: StoreApi<TState>['setState'] = (partial, replace) => {
    ...
  }

  const getState: StoreApi<TState>['getState'] = () => state

  const subscribe: StoreApi<TState>['subscribe'] = (listener) => {
    ...
  }

  const api = { setState, getState, getInitialState, subscribe }
    const initialState = (state = createState(setState, getState, api))
    return api as any
  }

export const createStore = ((createState) =>
  createState ? createStoreImpl(createState) : createStoreImpl) as CreateStore
```

2. <b>setState</b><br/>
   상태를 변경하는 setState 함수를 보면 인자가 function 타입일 경우 현재 상태를 인자로 넘겨 nextState를 정의한다. 그리고 nextState와 state가 다르다면 `Object.assign` 을 이용해서 상태를 갱신해 준다.

```js
  const setState: StoreApi<TState>['setState'] = (partial, replace) => {
    const nextState =
      typeof partial === 'function'
        ? (partial as (state: TState) => TState)(state)
        : partial
    if (!Object.is(nextState, state)) {
      const previousState = state
      state =
        (replace ?? (typeof nextState !== 'object' || nextState === null))
          ? (nextState as TState)
          : Object.assign({}, state, nextState)
      listeners.forEach((listener) => listener(state, previousState))
    }
  }
```

3. <b>subscribe</b><br/>
   상태를 구독하는 함수를 등록할 때는 subscribe 함수를 사용한다.

```js
const subscribe: StoreApi<TState>["subscribe"] = (listener) => {
  listeners.add(listener);
  // Unsubscribe
  // 구독을 해제하는 함수도 리턴
  return () => listeners.delete(listener);
};
```

<br/>

## zustand가 React 컴포넌트를 업데이트 할 수 있는 이유

vanilla.ts 를 통해 만들어진 스토어를 기반으로 리액트 컴포넌트에서 사용할 수 있도록 구현되어있다.
이 때 리액트에서 스토어의 값을 불러오기 위해 <a href='https://github.com/kwonjihyeon-dev/TIL/blob/master/react/hooks/useSyncExternalStore.md'>useSyncExternalStore</a>(zustand v4 이후 버전)를 이용한다.

```js
import { createStore } from './vanilla.ts'

// react.ts
export function useStore<TState, StateSlice>(
  api: ReadonlyStoreApi<TState>,
  selector: (state: TState) => StateSlice = identity as any,
) {
  const slice = React.useSyncExternalStore(
    api.subscribe,
    () => selector(api.getState()),
    () => selector(api.getInitialState()),
  )
  React.useDebugValue(slice)
  return slice
}

const createImpl = <T>(createState: StateCreator<T, [], []>) => {
  const api = createStore(createState)

  const useBoundStore: any = (selector?: any) => useStore(api, selector)

  Object.assign(useBoundStore, api)

  return useBoundStore
}
```

<br/>

[참고]<br/>
https://ui.toast.com/weekly-pick/ko_20210812<br/>
https://ingg.dev/zustand-work/
