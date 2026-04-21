---
title: iOS Safari Sticky Hover 이슈
date: 2026-04-16
tags: [iOS, Safari, hover, active, media query, Bottom Sheet, CSS]
---

# iOS Safari Sticky Hover 이슈

## 문제 상황

바텀시트에서 최하단 리스트 아이템을 클릭하면, 바텀시트가 닫힌 후 뒤에 있던 CTA 버튼에 hover 배경색이 적용되었다.

## 원인

iOS Safari에서 `:hover`는 터치 시 적용되고, 다른 곳을 터치하기 전까지 유지되는 "sticky" 특성이 있다. 터치 대상이 DOM에서 제거되면 같은 좌표의 다른 요소로 `:hover`가 전이된다.

1. 사용자가 리스트 아이템 터치 → 브라우저가 `:hover` 적용
2. 바텀시트 닫힘 → 리스트 아이템이 DOM에서 제거
3. iOS Safari가 같은 좌표에 있는 다른 요소(CTA)에 `:hover`를 전이
4. CTA 버튼에 `:hover` 배경색 표시

## 시도했지만 효과 없었던 방법들

| 방법 | 실패 이유 |
|---|---|
| `pointer-events: none` (body) | `:hover`는 브라우저 엔진 레벨 → CSS pointer-events로 차단 불가 |
| JS capture 핸들러 (`stopPropagation`) | `:hover`는 JS 이벤트가 아닌 CSS pseudo-class → JS로 차단 불가 |
| `transform: translateZ(0)` (컴포지팅 레이어 통합) | iOS Safari의 hover 전이와 무관 |
| `touchmove preventDefault` | 스크롤 차단 용도, hover 전이와 무관 |

## 해결

`:hover`를 `@media (hover: hover)`로 감싸서 터치 기기에서는 적용되지 않도록 변경하고, 터치 피드백은 `:active`로 대체했다.

```scss
// Before
&:hover {
  @apply bg-green-700;
}

// After
@media (hover: hover) {
  &:hover {
    @apply bg-green-700;
  }
}

&:active {
  @apply bg-green-700;
}
```

### 동작 차이

| | `:hover` | `:active` |
|---|---|---|
| 데스크톱 (마우스) | `@media (hover: hover)` 안에서 동작 | 클릭 시 동작 |
| 터치 (iOS Safari) | 적용 안 됨 → sticky hover 해결 | 터치 시 동작, 손가락 떼면 즉시 해제 |

- `@media (hover: hover)`: 마우스처럼 진정한 hover가 가능한 기기에서만 매칭
- `:active`: 터치/클릭 중에만 적용, 해제 시 즉시 사라짐 (sticky 없음)

## 수정 범위

자체 디자인 시스템의 Button, ListItem 컴포넌트 SCSS에서 `:hover` 스타일을 `@media (hover: hover)` + `:active`로 분리.

## 핵심 교훈

- iOS Safari에서 `:hover`는 터치 시 적용되고 sticky하게 유지됨
- 터치 대상이 DOM에서 제거되면 같은 좌표의 다른 요소로 전이됨
- **모바일 대응 시 `:hover`는 터치 피드백이 필요하면 `:active`를 `@media (hover: hover)`로 감싸서 예외 케이스에 대한 고려가 필요함**
