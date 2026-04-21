---
title: 모바일 스크롤 아키텍처 마이그레이션 - 내부 컨테이너 스크롤에서 페이지 전체 스크롤로
date: 2026-04-21
tags: [Mobile, Scroll, Migration, Architecture, Bottom Sheet, iOS Safari, CSS, position fixed, sticky]
references:
  - title: iOS Safari 바텀시트 배경 스크롤 잠금
    url: https://kwonjihyeon-dev.github.io/TIL/dev-log/ios-safari-bottom-sheet-scroll-lock
---

# 모바일 스크롤 아키텍처 마이그레이션: 내부 컨테이너 스크롤 → 페이지 전체 스크롤

> Fixed 플로팅 버튼 스크롤 버그 해결을 위해 "내부 컨테이너 스크롤"에서 "페이지 전체 스크롤(브라우저 기본 스크롤)" 구조로 전환하면서 발생한 일련의 이슈와 해결 과정

---

## 📌 발단: 플로팅 버튼에서 스크롤 시 html 전체가 스크롤되는 현상

### 원인
기존 구조는 **내부 컨테이너 스크롤 패턴**(페이지 내부의 특정 div가 스크롤을 담당):

```
<div class="h-dvh">
  <BottomSheetProvider class="h-full overflow-hidden">
    <div class="relative h-full">
      <Page class="h-full overflow-y-scroll">  ← 실제 스크롤 컨테이너
```

- globals.css의 `@media (hover: hover)`로 데스크톱에서만 html/body `overflow: hidden` lock
- 모바일에서는 html/body가 자유 스크롤 가능한 상태

### 추론
`position: fixed` 플로팅 버튼에 터치 후 스크롤 시:
1. 브라우저가 touch 이벤트의 스크롤 타깃을 찾음 → "가장 가까운 스크롤 가능한 조상 요소"<br/> — **scrollable ancestor**: 이 요소가 스크롤될 때 누가 움직이는가를 결정하는 조상 (DOM 트리 위로 탐색하며 `overflow: auto | scroll` 가진 첫 요소)
2. fixed 버튼은 내부 컨테이너의 자손이 아님 (position: fixed로 뷰포트 기준 배치)
3. 타깃 탐색이 html/body까지 fallthrough<br/> — **fallthrough**: 적절한 타깃을 찾지 못해 상위로 계속 밀려 올라가는 현상 (switch case의 break 없는 동작, 이벤트 버블링과 유사한 개념)
4. 모바일에선 html/body가 unlock 상태 → html이 스크롤됨
5. 결과적으로 **플로팅 버튼에서 스크롤 시 전체 페이지가 위로 밀리며 잘림**

### 결과
**페이지 전체 스크롤 구조로 마이그레이션**이 근본 해결책. 단 플로팅 버튼 없는 페이지는 내부 컨테이너 스크롤 유지 가능 (리스크/효용 균형).

---

## 1️⃣ Phase 1~2: 내부 컨테이너 스크롤 → 페이지 전체 스크롤 마이그레이션

### 원인
레이아웃 전반에 내부 컨테이너 스크롤 전제가 박혀있었음:
- 루트 layout: `<div class="h-dvh">` 모바일 wrapper
- BottomSheetProvider: 내부 `h-full overflow-hidden`
- 각 Page wrapper: `h-full overflow-y-scroll`
- 각 View: `h-dvh` 또는 `calc(100dvh)` SCSS

### 추론
- **데스크톱은 사이드바 레이아웃**이라 내부 컨테이너 스크롤 유지 필요 (사이드바 + 메인 영역)
- **모바일만 페이지 전체 스크롤로 전환**하면 데스크톱 영향 없음
- device detection context의 `isDesktop`으로 조건 분기

### 결과
**플로팅 버튼 있는 페이지만 선별 마이그레이션** (플로팅 버튼 없는 페이지는 마이그레이션 불필요 → 롤백)

핵심 변경 패턴:
```tsx
// 루트 layout (모바일)
// Before: <div className="h-dvh"><BottomSheetProvider className="h-full overflow-hidden">
// After:  <BottomSheetProvider>  // plain relative div

// BottomSheetProvider 내부 (모바일)
// Before: <div className="relative h-full">
// After:  <div className="relative">

// Page wrapper (모바일)
// Before: h-full overflow-y-scroll
// After:  isDesktop ? 'h-full overflow-y-scroll' : ''
```

---

## 2️⃣ Phase 3: iOS Safari scroll lock 강화

### 원인
기존 scroll lock 훅은 `document.body.style.overflow = 'hidden'` 한 줄만 적용.

### 추론
iOS Safari에서 `overflow: hidden`만으로는 **터치 스크롤을 완전히 막지 못함**. 실무 표준은:
- `position: fixed; top: -${scrollY}px`: body를 뷰포트에 고정 + 스크롤 위치 보존
- `width: 100%`: shrink-to-fit 방지
- `overscroll-behavior: none`: 바운스/pull-to-refresh 차단
- `touch-action: none`: 터치 제스처 차단
- 언락 시 `window.scrollTo(0, savedY)`로 원래 위치 복원

### 결과
전역 CSS + data attribute 패턴으로 통일:

```css
/* globals.css */
body[data-scroll-lock] {
  position: fixed;
  width: 100%;
  overflow: hidden;
  overscroll-behavior: none;
  touch-action: none;
  -webkit-overflow-scrolling: none;
}
```

```ts
// Lock
document.body.dataset.scrollLock = 'true';
document.body.style.top = `-${scrollY}px`;
document.body.style.paddingRight = `${scrollbarWidth}px`;

// Unlock
delete document.body.dataset.scrollLock;
document.body.style.top = '';
document.body.style.paddingRight = '';
window.scrollTo(0, savedY);
```

> 주의: 공용 UI 패키지의 lock 훅은 원본 유지. 다른 앱에 영향 주지 않기 위함.

---

## 3️⃣ Phase 4: 바텀시트 dim 사라짐 + 배경 스크롤

### 원인
Phase 1에서 BottomSheetProvider의 `h-full`을 제거했는데, 해당 div를 `container` 옵션으로 전달받은 `Dim`이:
- `position: absolute` (scoped)로 배치
- 부모(BottomSheetProvider div)의 높이에 스코핑됨
- 부모가 content 높이로 줄어들어 dim이 **뷰포트 전체를 덮지 못함**

### 추론
모바일에선:
1. `container` 주입을 생략 → Dim이 `document.body` portal + `position: fixed`로 렌더
2. 공용 Dim 훅이 `disableDocumentScroll=true` 경로로 동작 → 하지만 기존 lock 방식은 iOS 안전하지 않음
3. 바텀시트 렌더러에 **자체 scroll lock** 추가 (iOS Safari 안전한 방식)

### 결과
1. 바텀시트 열기 훅에서 모바일 시 `container` 주입 skip
2. 바텀시트 렌더러에 `data-scroll-lock` 패턴 scroll lock 추가
3. 공용 Dim 훅은 원본 유지 (타 앱 영향 차단)

```tsx
// 바텀시트 open hook
const defaultContainer = isDesktop ? (containerRef?.current ?? null) : null;

// 바텀시트 렌더러 자체 lock
useEffect(() => {
  if (!shouldRender || isDesktop) return;
  // apply data-scroll-lock lock
  return () => {
    // release + window.scrollTo(0, savedY)
    options?.onAfterClose?.();
  };
}, [isDesktop, shouldRender, options]);
```

---

## 4️⃣ Phase 5: Scroll 훅 device-aware 전환

### 원인
- scroll progress 훅: `containerRef.current.addEventListener('scroll')`
- tab state 훅: `container.scrollTop`, `anchor.offsetTop` 기반 계산
- meta sticky 훅: IntersectionObserver `root: scrollContainerRef.current`
- expandable 플로팅 버튼: `document.querySelector('.flex-1.overflow-y-auto')` 로 스크롤 컨테이너 탐색

→ 모바일이 페이지 전체 스크롤로 전환된 상태에서 모두 **scroll 이벤트를 못 잡음**

### 추론
모든 scroll 관련 훅/컴포넌트가 **device-aware**로 동작해야 함:
- 모바일: `window` + `window.scrollY` + `getBoundingClientRect().top + scrollY` (document 절대 Y)
- 데스크톱: 기존 `containerRef` + `scrollTop` + `offsetTop`

### 결과
공통 헬퍼 3개 유틸로 일원화:

```ts
// shared/model/utils/scrollTarget.ts
export const getScrollMetrics = (useWindow, container) => ({ scrollTop, clientHeight, scrollHeight });
export const getAnchorY = (el, useWindow) => useWindow
  ? el.getBoundingClientRect().top + window.scrollY
  : el.offsetTop;
export const getScrollTarget = (useWindow, container) => useWindow ? window : container;
```

각 훅에서 `isDesktop`으로 내부 분기:
- scroll observer/progress: target 자동 전환
- tab state: computeTabKey 입력 metric 통일
- meta sticky: IntersectionObserver `root: isDesktop ? ref.current : null`
- filter scrollTo: scroll target/offset 자동 전환

---

## 5️⃣ Sticky 요소 계층: 마이그레이션 시 고려사항

페이지 전체 스크롤로 전환하면 **모든 sticky 요소의 기준이 body 하나로 통합**됨. 기존에 내부 스크롤 컨텍스트 안에서 자연스럽게 쌓여있던 AppBar/Tab/FilterBar 등이 모두 같은 `top: 0`에서 만나 겹치므로 계층을 명시적으로 재설계해야 한다.

### ✅ 체크리스트

**1. sticky 요소 전수 조사**
- `sticky top-0`로 검색 → AppBar, Tab, FilterBar, 목록 헤더 등
- 데스크톱에선 각자 다른 scroll container 안에 있어 자연스레 분리되었던 요소들이, 모바일 body 스크롤에선 **하나의 scroll context(body)** 로 합쳐져 top:0에서 충돌

**2. 위→아래 순서로 누적 오프셋 계산**
- 최상단 요소(보통 AppBar): `top: 0`
- 그 아래 요소: `top: APP_BAR_HEIGHT`
- 또 그 아래 요소: `top: APP_BAR_HEIGHT + TAB_HEIGHT` (또는 해당 페이지 구조에 맞게)
- 페이지별로 Tab이 있는 곳과 없는 곳의 오프셋이 다르므로 구조 확인 필수

**3. device-aware 오프셋 분기**
- 데스크톱은 기존 내부 스크롤 컨텍스트라 `top: 0` 유지해도 자연스레 AppBar 아래 위치
- 모바일만 누적 오프셋 적용
```tsx
// Tab 요소 예시
style={{ top: isDesktop ? 0 : APP_BAR_HEIGHT }}
```
```ts
// filter state 훅 예시 (기존 공식을 device 분기)
const stickyTop = isDesktop
  ? headerOffset - APP_BAR_HEIGHT  // 내부 스크롤 기준
  : headerOffset;                   // body 스크롤 기준 (절대 Y)
```

**4. 누락된 상위 sticky 확인**
- 일부 AppBar는 `sticky`가 없고 일반 flow에 있을 수 있음 (내부 스크롤 구조에선 어차피 상단에 붙어있어 문제 없었음)
- 페이지 전체 스크롤로 바뀌면 **AppBar가 스크롤 따라 사라짐** → 명시적으로 `sticky top-0` 추가 필요

**5. z-index 재점검**
- 여러 sticky가 동시에 top 근처에 있을 때 z-index 순서가 중요
- 상단 요소일수록 높은 z-index: AppBar(200) > Tab(150) > FilterBar(140) 등
- z-index 생략된 곳은 브라우저 기본값(auto)라 예측 불가 → 명시 권장

**6. 내부 스크롤 유지 페이지와의 경계**
- 플로팅 버튼 없는 페이지(예: 등록 플로우)는 내부 컨테이너 스크롤 유지
- 이런 페이지의 sticky 오프셋은 기존 그대로 (device 분기 불필요)
- 마이그레이션 대상 페이지만 선별 적용

### 🎯 핵심 원칙

- 페이지 전체 스크롤 → sticky 기준이 body 하나로 통합됨, 계층 재설계 필수
- 모바일 오프셋은 **위 요소들의 높이 누적합**으로 계산
- device-aware `top` 값으로 데스크톱 내부 스크롤과 모바일 body 스크롤 양쪽 모두 커버
- z-index는 명시적으로 작성해 예측 가능한 스태킹 확보

---

## 6️⃣ 플로팅 버튼 positioning: 마이그레이션 시 고려사항

페이지 전체 스크롤 구조로 전환할 때 플로팅 버튼은 특히 예외적으로 확인해야 하는 영역. `position: absolute` vs `fixed`의 차이와 데스크톱/모바일 레이아웃 차이가 맞물려 쉽게 깨진다.

### ✅ 체크리스트

**1. 모든 플로팅 버튼의 `position` 속성 체크**
- `position: absolute`로 되어 있으면 → **positioned ancestor**<br/> — `position: absolute`의 좌표 원점이 어디인가를 결정하는 조상 (DOM 트리 위로 탐색하며 `position`이 `static`이 아닌 첫 요소, 없으면 뷰포트)<br/> — 기준으로 배치됨
- 기존 내부 컨테이너 스크롤 구조에선 ancestor가 고정 높이(`h-full`/`h-dvh`)라 하단 고정 정상 작동
- 페이지 전체 스크롤로 바꾸면 ancestor 높이가 content 따라 늘어남 → 버튼이 content 끝으로 밀려남

**2. 사이드바 레이아웃 있는 데스크톱 대응**
- 데스크톱에 사이드바(예: 390px 고정 폭)가 있다면 `fixed`로 바꾸면 뷰포트 전체 기준이라 **사이드바 밖 영역까지 번짐**
- 데스크톱은 `absolute` 유지(사이드바 내부 기준), 모바일만 `fixed`로 분기해야 함

```tsx
className={`${isDesktop ? 'absolute' : 'fixed'} bottom-7 right-5`}
```

**3. 이미 `fixed`인 버튼의 중앙 정렬 재확인**
- `fixed left-1/2 -translate-x-1/2`는 뷰포트 가로 중앙
- 데스크톱 사이드바 레이아웃에선 뷰포트 중앙이 사이드바 내부가 아닌 메인 영역(지도 등)과 겹칠 수 있음
- 데스크톱에서만 `absolute`로 바꿔 positioned ancestor(사이드바) 중앙에 오도록 조정

**4. IntersectionObserver·scroll listener 동반 로직 확인**
- 플로팅 버튼에 "스크롤 시 펼침/접힘" 같은 애니메이션이 걸린 경우 `document.querySelector('id 혹은 class')` 로 스크롤 컨테이너를 찾는 코드가 있을 수 있음
- 모바일이 페이지 전체 스크롤로 바뀌면 해당 셀렉터가 null → 리스너 미등록 → 애니메이션 미작동
- device 분기로 `window` 이벤트로 전환 필요

### 🎯 핵심 원칙

- 단순 `absolute`는 ancestor 높이에 종속 → 페이지 전체 스크롤에선 불안정
- 플로팅 버튼의 의미상 "뷰포트 고정"이 맞다면 기본적으로 `fixed`
- 사이드바 레이아웃이 있는 데스크톱이면 device-aware `absolute/fixed` 분기

---

## 7️⃣ 스크롤 위치 기반 UI 상태(탭 활성화·진행도 표시 등)의 false positive

### 배경: 어떤 UI가 영향을 받는가
페이지 스크롤 위치를 읽어 **현재 보고 있는 섹션을 판별**하거나 **UI 상태를 바꾸는** 로직이 있을 때 발생하는 문제. 예를 들어:
- 긴 페이지에 여러 섹션(개요·리뷰·상세 등)이 있고 상단 **탭바가 현재 섹션에 따라 하이라이트**되는 경우 (scrollspy 패턴)
- 스크롤 내려가면 **헤더 색·배경이 변하는** 효과
- "페이지 끝 도달했나"를 감지해 **무한 스크롤** 또는 **마지막 섹션 탭 강조** 처리

이런 로직은 보통 `window.scrollY`·`document.documentElement.scrollHeight`·`innerHeight`를 조합한 공식으로 판단한다. 예: `scrollY + innerHeight >= scrollHeight` → "스크롤 끝 도달".

### 문제: 바텀시트/팝업 열림이 UI 상태를 오작동시킴
페이지 전체 scroll lock이 발동되면 `position: fixed`로 body가 flow 밖으로 나가면서 scroll metrics가 왜곡됨:
- `window.scrollY` → 0
- `scrollHeight` → 뷰포트 높이 수준으로 축소
- 결과적으로 "스크롤 끝 도달" 공식이 **항상 true**로 평가됨

사용자가 단지 필터 버튼을 눌러 바텀시트를 여는 것만으로도:
- 탭이 갑자기 "마지막 섹션"으로 튐
- 무한 스크롤이 불필요하게 다음 페이지를 fetch
- 헤더 색이 스크롤 끝 상태로 고정됨

### 해결 방법
scroll metrics 기반 판단 로직에 **lock 상태 가드**를 추가:

- 오버레이(바텀시트·팝업·다이얼로그) 열림 시 body에 공통 속성(`data-scroll-lock` 등)을 부여
- scroll 기반 계산 훅은 해당 속성이 있을 때 "스크롤 끝" 같은 파생 상태를 **false로 고정**하거나 **이전 값 유지**
- 이렇게 하면 오버레이 열림 중에는 UI 상태가 얼어붙고, 오버레이 닫힌 뒤 정상 scroll metrics 복구 시점부터 다시 계산이 유효해짐

**핵심**: 계산 자체는 그대로 두되, "계산 입력이 왜곡되는 구간"은 판단에서 배제.

---

## 📚 핵심 인사이트

### 1. 내부 컨테이너 스크롤 구조는 "fixed 버튼 터치 버그"의 근본 원인
- 브라우저의 터치 스크롤 타깃 탐색이 fixed 요소를 건너뛰고 상위로 fallthrough
- html/body가 unlock인 모바일에선 이 fallthrough가 html scroll로 이어져 페이지 깨짐

### 2. `@media (hover: hover)` 가드는 데스크톱/모바일 이분법에 부적절
- 태블릿 가로모드(호버 없지만 1024px+), 하이브리드 디바이스 등 엣지 케이스
- 실제 디바이스 타입은 SSR UA 기반으로 판별하고 CSS는 무조건 적용 권장

### 3. iOS Safari scroll lock 표준
- `overflow: hidden`만으로는 부족
- `position: fixed; top: -scrollY` + `overscroll-behavior: none` + `touch-action: none` 세트
- 언락 시 `window.scrollTo(0, savedY)` 복원 필수

### 4. 페이지 전체 scroll lock은 scroll metrics를 왜곡
- `position: fixed`로 body를 잠그면 `window.scrollY`가 0으로 초기화되고 `document.documentElement.scrollHeight`도 뷰포트 수준으로 축소됨 (body가 flow 밖이라 스크롤 개념 자체가 사라지기 때문)
- 이 수치들을 읽어 판단하는 로직(탭 활성화, 스크롤 진행도, "스크롤 끝" 감지 등)이 lock 발동 순간 false positive/negative를 일으킴
- 예: `scrollY + innerHeight >= scrollHeight` 공식이 lock 상태에선 항상 true → 바텀시트 오픈만으로 "스크롤 끝에 도달했다"고 오판
- scroll 기반 상태 계산 훅은 lock 상태를 명시적으로 배제해야 안전 (`data-scroll-lock` 같은 공통 속성으로 가드)

### 6. device-aware scroll 유틸 일원화
- `getScrollMetrics`, `getAnchorY`, `getScrollTarget` 3개 헬퍼로 device 분기 중앙집중화
- 향후 scroll 기반 훅 추가 시 재사용

### 7. floating button은 `fixed` 또는 device-aware `absolute/fixed`
- 단순 `absolute`는 ancestor 높이에 종속 → 페이지 전체 스크롤에선 content 따라 밀림
- 사이드바 레이아웃 있는 경우 데스크톱 `absolute`(사이드바 내부) vs 모바일 `fixed`(뷰포트) 분기
