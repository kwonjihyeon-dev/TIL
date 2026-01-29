---
layout: post
title: SVG in Safari
date: 2026-01-22
---

# Safari에서 SVG 이미지 픽셀화 문제 분석

Safari 브라우저에서 SVG 파일을 `<img src='**.svg' />`로 import할 때 `<img>` 태그와 SVG 파일 내부에 크기 속성이 모두 정의되어 있음에도 불구하고 이미지 해상도가 깨져보이는 현상이 발생합니다.

## 픽셀화 발생 원인

### 1. 벡터 vs 래스터 이미지

**벡터 이미지 (SVG)**

-   수학적 공식으로 이미지 표현
-   점, 선, 곡선의 좌표와 속성을 저장
-   확대/축소해도 항상 선명함

```svg
<circle cx="50" cy="50" r="40" />
<!-- 아무리 확대해도 완벽한 원 -->
```

**래스터 이미지 (PNG, JPEG 등)**

-   픽셀(pixel)들의 격자로 구성
-   각 픽셀은 고유한 색상 정보 보유
-   확대 시 픽셀이 보이며 계단 현상 발생

### 2. Safari의 SVG 렌더링 과정

```
1. SVG 파일 (벡터) 읽기
2. 특정 크기로 픽셀 이미지로 변환 (래스터화)
3. 래스터 이미지를 GPU에 전달
4. 낮은 해상도로 변환된 경우
   → 이후 확대해도 픽셀화된 상태로 표시
```

### 3. 하드웨어 가속 관련 이슈

Safari는 성능 최적화를 위해 SVG를 GPU로 전달하기 전에 래스터 이미지로 변환합니다. 이 과정에서:

-   디바이스 픽셀 밀도(device pixel ratio)를 고려
-   요소의 렌더링 컨텍스트 기반으로 래스터화 해상도 결정
-   특정 상황에서 해상도 계산 오류 또는 낮은 해상도로 고정

### 4. 서브픽셀(Subpixel) 정렬 문제

**서브픽셀이란?**
브라우저 레이아웃 엔진이 정밀한 계산을 위해 소수점 단위로 요소 크기를 계산하는 것:

```css
width: 100.5px; /* 0.5px는 서브픽셀 */
height: 50.75px; /* 0.75px도 서브픽셀 */
```

**문제 발생 메커니즘**

```
1. 레이아웃 엔진 계산
   계산된 SVG 크기: 100.7px × 100.7px

2. Safari의 래스터화 단계
   래스터화 해상도 선택: 100px? 101px?
   → 반올림/버림 과정에서 부정확한 선택

3. 실제 렌더링
   101px로 래스터화된 이미지를 100.7px 공간에 표시
   → 정렬 오류, 픽셀화 발생
```

**프레임별 불일치**

```
프레임 1: 100.7px → 101px로 래스터화 → 약간 흐림
프레임 2: 100.6px → 101px로 래스터화 → 0.4px 어긋남
프레임 3: 100.8px → 101px로 래스터화 → 0.2px 어긋남
→ 매 프레임마다 미세하게 다른 정렬 = 시각적 떨림/픽셀화
```

## 해결 방법

### 1. SVG 파일 내부 속성 최적화

```svg
<svg viewBox="0 0 100 100"
     width="100"
     height="100"
     preserveAspectRatio="xMidYMid meet"
     xmlns="http://www.w3.org/2000/svg">
  <!-- SVG 내용 -->
</svg>
```

### 2. CSS 크기를 정수 단위로 지정

```css
/* 서브픽셀 크기 방지 - 정수 단위 강제 */
img {
    width: 100px; /* 100.5px 같은 소수점 피하기 */
    height: 100px;
}
```

### 3. 하드웨어 가속 제어

```css
/* 하드웨어 가속 비활성화 테스트 */
img {
    transform: none;
    -webkit-backface-visibility: visible;
}

/* 또는 명시적 2D 렌더링 강제 */
img {
    transform: translateZ(0); /* 제거해보기 */
    will-change: transform; /* 제거해보기 */
}
```

### 4. flexbox/grid에서 서브픽셀 계산 주의

```css
/* 문제 상황 */
.container {
    width: 301px; /* 3등분 시 100.333px → 서브픽셀 발생 */
}

/* 개선 */
.container {
    width: 300px; /* 3등분 시 정확히 100px */
}
```

### 5. 대안적 접근

```html
// inline SVG 등으로 직접 DOM에 그릴 경우, 1. SVG가 DOM의 일부로 직접 존재 2. 브라우저 렌더링 엔진이 직접 처리 3. 렌더링
시점마다 벡터 데이터를 참조 4. 현재 필요한 크기로 직접 래스터화 5. 크기 변경 시 벡터 데이터에서 다시 래스터화 → 항상
최적 품질 유지

<!-- inline SVG 사용 -->
<svg>...</svg>

<!-- object 태그 사용 -->
<object data="icon.svg" type="image/svg+xml"></object>

<!-- CSS background-image 사용 -->
<div style="background-image: url('icon.svg')"></div>
```

## 요약

Safari에서 SVG 픽셀화 문제는:

1. **SVG가 프레임마다 래스터화**되며
2. **레이아웃 엔진의 서브픽셀 크기 계산**과 **래스터화 시 정수 픽셀로의 변환 불일치**로 인해 발생

---

### 참고사이트

[SVG 아이콘이 사파리에서 픽셀화되어보이는 이유](https://medium.com/design-bootcamp/my-svg-icons-looked-pixelated-on-safari-heres-why-5a04e8eca731)
