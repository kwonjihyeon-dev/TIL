---
title: iOS Safari 바텀시트 배경 스크롤 잠금
date: 2026-04-14
tags: [iOS, Safari, Bottom Sheet, Scroll Lock, CSS, touch-action, overscroll-behavior]
references:
  - title: 모바일 스크롤 아키텍처 마이그레이션 - 내부 컨테이너 스크롤에서 페이지 전체 스크롤로
    url: https://kwonjihyeon-dev.github.io/TIL/dev-log/mobile-scroll-migration
---

# iOS Safari 바텀시트 배경 스크롤 잠금

## 문제 상황

모바일 웹에서 바텀시트가 열려 있을 때 뒤쪽 배경이 계속 스크롤되는 현상이 발생했다. 바텀시트 뒤의 콘텐츠가 움직이면 사용자 경험을 해치기 때문에 스크롤을 잠가야 했다.

## 시도했던 방법들

### 1. `overflow: hidden` on `documentElement`

```typescript
useEffect(() => {
  if (!shouldRender) return undefined;

  document.documentElement.style.overflow = 'hidden';
  document.documentElement.style.height = '100vh';

  return () => {
    document.documentElement.style.removeProperty('overflow');
    document.documentElement.style.removeProperty('height');
  };
}, [shouldRender]);
```

- **문제점**: iOS Safari에서는 터치 기반 스크롤을 CSS overflow와 별개의 메커니즘으로 처리하기 때문에, `overflow: hidden`을 설정해도 터치 스크롤이 여전히 동작한다.

### 2. `position: fixed` on `body`

```typescript
useEffect(() => {
  if (!shouldRender) return undefined;

  const scrollY = window.scrollY;
  const body = document.body;

  body.style.position = 'fixed';
  body.style.top = `-${scrollY}px`;
  body.style.left = '0';
  body.style.right = '0';
  body.style.overflow = 'hidden';

  return () => {
    body.style.removeProperty('position');
    body.style.removeProperty('top');
    body.style.removeProperty('left');
    body.style.removeProperty('right');
    body.style.removeProperty('overflow');
    window.scrollTo(0, scrollY);
  };
}, [shouldRender]);
```

- **장점**: body를 뷰포트에 고정시켜 iOS에서도 스크롤이 차단됨
- **문제점**: `position: fixed`가 overscroll 컨텍스트를 생성하여 배경을 터치하면 **pull-to-refresh(당겨서 새로고침)** 가 발생

### 3. `touchmove` preventDefault

```typescript
useEffect(() => {
  if (!shouldRender) return undefined;

  const preventTouchMove = (e: TouchEvent) => {
    if (containerRef.current?.contains(e.target as Node)) return;
    e.preventDefault();
  };

  document.addEventListener('touchmove', preventTouchMove, { passive: false });

  return () => {
    document.removeEventListener('touchmove', preventTouchMove);
  };
}, [shouldRender]);
```

- **장점**: 바텀시트 외부의 터치 스크롤만 차단, 내부 스크롤은 허용
- **문제점**: 바텀시트 내부 스크롤이 경계(맨 위/맨 아래)에 도달하면 터치 이벤트가 외부로 전파되어 배경이 함께 스크롤됨 (스크롤 체이닝)

<br />

## 해결 방법: `data-attribute` + CSS 조합

### JS: data 속성 토글

인라인 스타일 대신 `data-layout-scroll` 속성을 토글하는 방식으로 변경했다.

```typescript
useEffect(() => {
  if (!shouldRender) return undefined;

  document.body.dataset.layoutScroll = 'locked';

  return () => {
    delete document.body.dataset.layoutScroll;
  };
}, [shouldRender]);
```

### CSS: 글로벌 스타일

```css
body[data-layout-scroll='locked'] {
  max-height: 100vh;
  overflow: hidden;
  overscroll-behavior: none;
  -webkit-overflow-scrolling: none;
  touch-action: none;
}
```

<br />

## 각 CSS 속성의 역할

| 속성 | 역할 | 동작 레벨 |
|------|------|-----------|
| `max-height: 100vh` | body 높이를 뷰포트로 제한하여 스크롤 가능 영역 자체를 제거 | 레이아웃 |
| `overflow: hidden` | 콘텐츠가 넘칠 때 스크롤바를 생성하지 않음 | CSS |
| `overscroll-behavior: none` | 스크롤 체이닝과 기본 동작(pull-to-refresh, 바운스) 차단 | 스크롤 경계 |
| `-webkit-overflow-scrolling: none` | iOS Safari의 관성(모멘텀) 스크롤 비활성화 | WebKit |
| `touch-action: none` | 브라우저의 터치 제스처 처리 자체를 비활성화 | 터치 엔진 |

### 왜 이 조합이 동작하는가

이전 방법들이 실패한 근본적인 이유는 **iOS Safari의 터치 스크롤이 CSS overflow와 독립적으로 동작**하기 때문이다.

- `overflow: hidden`은 CSS 레벨에서 스크롤을 제어하지만, iOS Safari의 터치 이벤트 처리 엔진은 이를 무시하고 자체적으로 스크롤을 수행한다.
- `touch-action: none`은 **브라우저의 터치 이벤트 처리 엔진 레벨**에서 동작한다. 브라우저에게 "터치 제스처를 직접 처리하지 말라"고 지시하므로, iOS Safari가 `overflow: hidden`을 무시하더라도 터치 스크롤 자체가 차단된다.
- `overscroll-behavior: none`은 스크롤이 경계에 도달했을 때 발생하는 기본 동작을 차단한다. `position: fixed` 방식에서 발생하던 **pull-to-refresh 문제**를 이 속성이 해결한다.

```
[터치 입력]
    ├─ touch-action: none → 브라우저 터치 제스처 처리 차단 ✅
    ├─ overflow: hidden → CSS 레벨 스크롤 차단 (iOS에서 불완전)
    ├─ overscroll-behavior: none → 스크롤 경계 동작 차단 ✅
    └─ -webkit-overflow-scrolling: none → 관성 스크롤 차단 ✅
```

결국 **여러 레이어에서 중첩으로 스크롤을 차단**하는 것이 iOS Safari에서 확실한 해결책이다.

<br />

## 결론

iOS Safari의 스크롤 잠금은 단일 CSS 속성으로 해결할 수 없다. `overflow: hidden`만으로는 터치 스크롤을 막을 수 없고, `position: fixed`는 pull-to-refresh를 유발한다. `touch-action`, `overscroll-behavior`, `overflow` 를 조합하여 터치 엔진, 스크롤 경계, CSS 레이아웃 각 레벨에서 스크롤을 차단해야 한다.

<br />

## 참고

- [MDN - touch-action](https://developer.mozilla.org/en-US/docs/Web/CSS/touch-action)
- [MDN - overscroll-behavior](https://developer.mozilla.org/en-US/docs/Web/CSS/overscroll-behavior)
- [iOS Safari scrolling issues](https://bugs.webkit.org/show_bug.cgi?id=153852)
