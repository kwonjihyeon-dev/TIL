---
layout: post
title: Error Boundary
---

## Error Boundary

에러가 발생한 부분 대신 오류 메시지와 같은 fallback UI를 표시할 수 있는 특수 컴포넌트<br/>
<u>선언적 프로그래밍</u>(상태의 변화에 따라 UI를 직접 조작하지 않고, 어떤 상태에 따라 어떤 UI가 보여져야 하는지에만 집중하는 것) 기반

## 사용법

오류에 대한 응답으로 state를 업데이트하고 사용자에게 오류 메시지를 표시할 수 있는 static getDerivedStateFromError를 작성해야한다.

```js
// error boundary 컴포넌트
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    // state를 업데이트해야 다음 렌더링에 fallback UI가 보여질 수 있음
    return { hasError: true };
  }

  // 오류를 기록하는 등의 추가 로직을 추가할 수도 있다 (선택)
  componentDidCatch(error, info) {
    // Example "componentStack":
    //   in ComponentThatThrows (created by App)
    //   in ErrorBoundary (created by App)
    //   in div (created by App)
    //   in App
    logErrorToMyService(error, info.componentStack);
  }

  render() {
    if (this.state.hasError) {
      // 오류 발생 시 보여줄 UI
      // return <>로딩중...</> 과 같이 커스텀 가능
      return this.props.fallback;
    }

    return this.props.children;
  }
}

// 사용 시
<ErrorBoundary fallback={<p>Something went wrong</p>}>
  <Profile />
</ErrorBoundary>;
```

## 주의사항

ErrorBoundary에서 감지할 수 있는 에러

1. 동기적 렌더링 중 발생한 에러
2. 라이프사이클 메서드에서 발생한 에러
3. React 이벤트 핸들러에서 발생한 에러

setTimeout이나 Promise와 같은 비동기 작업은 JavaScript의 콜백 큐에서 실행되므로(try 내부에 있을 경우 성공한 것으로 간주) React의 렌더링 흐름과 별개로 처리되기 때문.

```js
setTimeout(() => {
  // 내부에서 에러를 감지하도록 설정
  try {
    throw new Error("Something went wrong");
  } catch (error) {
    console.error("Caught error in setTimeout:", error);
  }
}, 1000);
```
