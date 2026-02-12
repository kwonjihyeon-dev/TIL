# iOS Safari 가상 키보드 정책과 해결 방법

## 문제 상황

모바일 웹에서 검색 페이지로 이동 시 input에 자동으로 포커스가 잡히고 가상 키보드가 올라와야 하는 UX를 구현해야 했다.

## iOS Safari의 가상 키보드 정책

> **`input.focus()`는 사용자의 직접적인 터치 이벤트의 동기 실행 흐름(call stack) 안에서 호출되어야만 가상 키보드가 활성화된다.**

즉, 아래 조건을 모두 만족해야 한다:

1. 사용자의 **물리적 터치(탭)** 이벤트가 발생해야 한다
2. 해당 이벤트 핸들러의 **동기 실행 흐름** 안에서 `focus()`가 호출되어야 한다
3. 비동기 작업(`setTimeout`, `Promise`, `requestAnimationFrame` 등)을 거치면 gesture 체인이 끊어진다

### 동작하는 경우 vs 동작하지 않는 경우

```
// O: 사용자 탭 → 동기적 focus → 키보드 활성화
<input onFocus={handleNavigation} />
// 사용자가 input을 직접 탭 → focus 이벤트 발생 → 키보드 활성화

// X: 사용자 탭 → 비동기 네비게이션 → mount → focus → 키보드 무시
<button onClick={() => {
  router.push('/검색 페이지');  // 비동기 네비게이션
}} />
// 이후 검색 페이지의 autoFocus → gesture 체인 끊김 → 키보드 활성화 안 됨
```

## 해결 방법: 투명 input 오버레이 패턴

아이콘 버튼 위에 투명한 `readOnly` input을 겹쳐 놓아, 사용자의 탭이 직접 input의 focus 이벤트로 이어지도록 한다.

### Before (키보드 미활성화)

```tsx
<Button onClick={moveToSearch}>
    <Icon size="md" icon="search" />
</Button>
```

사용자가 버튼을 탭하면 `onClick` → `router.push('/검색 페이지')` → 검색 페이지 마운트 → `autoFocus`로 focus 시도. 하지만 이 시점에서는 이미 터치 이벤트의 동기 실행 흐름이 끊어져 iOS Safari가 키보드를 차단한다.

### After (키보드 활성화)

```tsx
<Button className="relative !p-[14px]">
    <Icon size="md" icon="search" />
    <input readOnly className="absolute inset-0 opacity-0" onFocus={moveToSearch} />
</Button>
```

사용자가 탭하면 투명 input이 직접 포커스를 받아 iOS가 이를 "사용자가 입력 필드를 터치한 이벤트"로 인식한다. `onFocus`에서 페이지 이동이 일어나고, 검색 페이지의 input이 마운트되면서 키보드가 유지된다.

### 핵심 포인트

| 속성               | 역할                                           |
| ------------------ | ---------------------------------------------- |
| `readOnly`         | 탭 시 입력 커서가 나타나지 않도록 방지         |
| `absolute inset-0` | 아이콘 버튼 영역 전체를 덮어 탭 영역 확보      |
| `opacity-0`        | 시각적으로 보이지 않지만 포커스는 받을 수 있음 |
| `onFocus`          | `onClick` 대신 사용하여 iOS gesture 체인 유지  |

## 시도했던 다른 방법들

### 1. 검색 페이지 mount 시점에서 focus

```tsx
useEffect(() => {
    inputRef.current?.focus();
}, []);
```

-   **문제점**: 보편적으로 사용했던 방법. `useEffect`는 비동기 실행이므로 터치 이벤트의 gesture 체인이 이미 끊어진 상태. iOS Safari가 키보드를 차단함

### 2. 임시 input 동적 생성

```ts
const searchClick = () => {
    const tempInput = document.createElement('input');
    document.body.appendChild(tempInput);
    tempInput.focus(); // 키보드 활성화
    router.push('/검색 페이지');
    setTimeout(() => tempInput.remove(), 2000);
};
```

-   **문제점**: 키보드가 페이지 전환보다 먼저 나타나 UX가 어색함
-   **문제점**: `router.push`와 동적 input의 focus가 충돌하여 라우터 이동이 실패하는 경우 발생

### 2-1. requestAnimationFrame으로 라우터 지연

```ts
tempInput.focus();
requestAnimationFrame(() => {
    setTimeout(() => tempInput.remove(), 2000);
    router.push('/검색 페이지');
});
```

-   **문제점**: 라우터 이동은 가능하지만 가상 키보드가 검색 화면으로 이동 전에 미리 노출되는 상황 발생

<br />

## 결론

iOS Safari의 가상 키보드 정책은 "사용자가 직접 input을 터치해야 키보드가 뜬다"로 요약된다. 페이지 마운트 시에 자동으로 키보드를 띄우려면, **전환 트리거 자체가 input의 직접적인 터치 이벤트**여야 한다.

<br />

## 참고

### Vue vs React

같은 `focus()` 호출이라도 Vue에서는 키보드가 뜨고 React에서는 뜨지 않는 경우가 있다. 이는 각 프레임워크의 **내부 DOM 업데이트 스케줄링** 차이 때문이다.

### Vue의 DOM 업데이트

```
사용자 탭 → click handler → 상태 변경 → DOM 동기 패치 → $nextTick (microtask) → focus()
└──────────────────── 같은 microtask checkpoint ─────────────────────────────────┘
```

Vue는 상태 변경 시 **DOM을 동기적으로 패치**하고, `$nextTick`은 `Promise.resolve()` 기반 **microtask**이다. 전체 흐름이 같은 microtask checkpoint 안에서 완료되어 iOS가 gesture 체인으로 인정한다.

### React의 DOM 업데이트

```
사용자 탭 → click handler → setState → [스케줄러 대기] → reconciliation → commit → focus()
                                        └─ MessageChannel (macrotask) ─┘
```

React 18+는 내부 스케줄러가 **`MessageChannel`(macrotask)** 로 렌더링을 스케줄링한다. 상태 변경 → 실제 DOM 커밋 사이에 macrotask 경계가 생기면서 gesture 체인이 끊어진다.

### 이벤트 루프 관점

```
이벤트 루프:
[사용자 탭] → [동기 코드] → [microtask] → [렌더링] → [macrotask]
               Vue: 여기서 DOM 패치 ──→ 여기서 focus
                                                      React: 여기서 commit + focus
              ├──── gesture 체인 유지 ────┤            ├── gesture 체인 끊김 ──┤
```

### React의 `autoFocus`가 동작하지 않는 이유

React는 JSX의 `autoFocus` prop을 네이티브 HTML `autofocus` 속성으로 전달하지 않고, 커밋 단계에서 프로그래밍적으로 `.focus()`를 호출한다. 이 커밋 단계가 macrotask에서 실행되므로 gesture 체인 밖에 있다.

```tsx
// React 내부 동작
<input autoFocus />
// → 네이티브 autofocus 속성이 아닌 commitPhase에서 element.focus() 호출

// 실제 DOM element
<input autofocus />
// → DOM에 autofocus 속성이 있는 이유는 React가 JSX prop을 DOM 속성으로 매핑하는 일반적인 동작의 일부이기 때문입니다. 하지만 HTML 네이티브 autofocus는 최초 페이지 로드 시에만 브라우저가 처리합니다. SPA 라우팅으로 컴포넌트가 마운트될 때는 브라우저가 이 속성을 무시합니다.

```

### 비교 정리

|                                   | Vue                     | React 18+                  |
| --------------------------------- | ----------------------- | -------------------------- |
| DOM 패치 타이밍                   | 동기                    | 스케줄러 기반 (비동기)     |
| focus 실행 시점                   | microtask (`$nextTick`) | macrotask (MessageChannel) |
| gesture 체인                      | 유지                    | 끊김                       |
| 같은 페이지 내 input 토글 + focus | **키보드 O**            | **키보드 X**               |

<br />
Vue에서는 동기 DOM 패치 + microtask 기반 `$nextTick` 덕분에 프로그래밍적 focus로도 키보드가 활성화되는 경우가 있지만, React 18+에서는 macrotask 기반 스케줄링으로 인해 동일한 방식이 동작하지 않는다.
