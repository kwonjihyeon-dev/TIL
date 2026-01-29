---
layout: post
title: Hardware Acceleration
date: 2026-01-06
---

# Safari의 GPU 레이어 관리와 렌더링 문제

1. [개요](#개요)
2. [GPU 레이어 관리의 기본 개념](#gpu-레이어-관리의-기본-개념)
3. [Safari의 레이어 관리 철학](#safari의-레이어-관리-철학)
4. [Hardware Acceleration 미적용 문제](#hardware-acceleration-미적용-문제)

## 개요

Safari는 성능 최적화를 위해 GPU 레이어를 관리하는 독자적인 전략을 사용합니다.<br/>이는 메모리와 배터리를 절약하기 위한 선택이지만, **Hardware Acceleration이 제대로 적용되지않는 등** 다양한 렌더링 문제를 일으킵니다.

## GPU 레이어 관리의 기본 개념

### CPU vs GPU 렌더링

```
CPU 렌더링 (Software Compositing)
┌──────────────────────────┐
│ 메인 스레드에서 순차 처리       │
│ - 느린 처리 속도             │
│ - 낮은 메모리 사용           │
│ - 배터리 효율적              │
└──────────────────────────┘

GPU 렌더링 (Hardware Acceleration)
┌──────────────────────────┐
│ GPU에서 병렬 처리          │
│ - 빠른 처리 속도           │
│ - 높은 메모리 사용         │
│ - 배터리 소모 증가         │
└──────────────────────────┘
```

### 레이어(Compositing Layer)란?

브라우저는 성능 최적화를 위해 특정 요소를 별도의 **레이어**로 분리하여 GPU에서 처리합니다.

```
화면 구성:
┌─────────────────────────────────┐
│ Layer 1 (CPU)                   │
│  ├─ header                      │
│  └─ content                     │
├─────────────────────────────────┤
│ Layer 2 (GPU) - animated-box    │
│  - 독립적으로 변환/애니메이션          │
│  - 다른 요소 영향 없음              │
└─────────────────────────────────┘
```

### 레이어 승격(Promotion) 기준

**Chrome/Firefox (적극적 승격):**

```css
/* 자동으로 GPU 레이어로 승격되는 속성들 */
.element {
    transform: translateX(100px); /* ✅ GPU */
    opacity: 0.5; /* ✅ GPU */
    filter: blur(5px); /* ✅ GPU */
    will-change: transform; /* ✅ GPU */
}
```

**Safari (보수적 승격):**

```css
/* Safari는 더 엄격한 기준 적용 */
.element {
    transform: translateX(100px); /* ✅ GPU */
    opacity: 0.5; /* ⚠️ 경우에 따라 CPU */
    filter: blur(5px); /* ⚠️ 경우에 따라 CPU */
}

/* 명시적으로 요청해야 GPU 레이어 생성 */
.element {
    opacity: 0.5;
    transform: translateZ(0); /* ✅ 강제 GPU */
    will-change: opacity; /* ✅ 강제 GPU */
}
```

## Safari의 레이어 관리 철학

### 핵심 원칙: 성능 vs 메모리의 균형

```
Safari의 설계 목표:
┌─────────────────────────────────┐
│ 모바일 환경 최적화                   │
│  - 제한된 RAM (iPhone: ~6GB)      │
│  - 배터리 수명                     │
│  - 발열 관리                      │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ 보수적 레이어 관리 전략               │
│  1. 불필요한 레이어 생성 자제         │
│  2. GPU 사용을 신중하게 결정         │
│  3. 메모리 절약 우선                │
└─────────────────────────────────┘
```

## Hardware Acceleration 미적용 문제

### 증상

**Chrome/Firefox:** 부드러운 fade 애니메이션 ✅  
**Safari:** 버벅이거나 애니메이션이 제대로 작동하지 않음 ❌

### 원인 분석

```
Safari의 레이어 판단 프로세스:

1. CSS 파싱
   .modal-closed { opacity: 0; transition: opacity 0.3s; }

2. 레이어 승격 판단
   ├─ 3D transform 있음? → 없음
   ├─ will-change 있음? → 없음
   ├─ opacity만 있음? → ⚠️ GPU 레이어로 안 만듦
   └─ 결정: CPU에서 처리

3. Transition 실행
   CPU 렌더링:
   ├─ opacity: 0 → 0.1 → 0.2 → ... → 1.0
   ├─ 각 프레임마다 전체 페이지 repaint
   └─ 느리고 버벅이는 애니메이션
```

### 성능 영향

```javascript
// 성능 측정 예시
const performanceComparison = {
    Chrome: {
        레이어: 'GPU',
        FPS: 60,
        CPU사용률: '10%',
        렌더링방식: '레이어 합성만',
    },
    Safari_문제상황: {
        레이어: 'CPU',
        FPS: 30 - 45,
        CPU사용률: '40%',
        렌더링방식: '전체 페이지 repaint',
    },
};
```

**원리:**

```
translateZ(0)의 효과:

1. Safari가 인식: "3D transform이 있네!"
2. GPU 레이어 생성 결정
3. 이후 opacity transition도 GPU에서 처리
4. 부드러운 애니메이션 ✅

주의: translateZ(0)는 실제로 아무것도 이동하지 않음
      단지 GPU 레이어를 만들기 위한 트릭
```

#### 방법 2: `will-change` - 변경 예고

```css
.modal-closed {
    opacity: 0;
    transition: opacity 0.3s ease;
    /* 브라우저에게 변경 예고 */
    will-change: opacity;
}

/* 또는 */
.modal {
    will-change: opacity, transform;
}
```

**원리:**

```
will-change의 효과:

1. 브라우저에게 "이 속성이 변경될 예정"이라고 알림
2. 브라우저는 미리 GPU 레이어를 생성
3. 실제 변경 시 빠른 처리

주의사항:
- 모든 요소에 남발하지 말 것 (메모리 낭비)
- 애니메이션 완료 후 제거하는 것이 좋음
```

#### 방법 3: `backface-visibility` - 렌더링 최적화

```css
.modal-closed {
    opacity: 0;
    transition: opacity 0.3s ease;
    /* Safari 렌더링 최적화 */
    -webkit-backface-visibility: hidden;
    backface-visibility: hidden;
}
```

### 참고 사이트

https://www.sitepoint.com/introduction-to-hardware-acceleration-css-animations/
