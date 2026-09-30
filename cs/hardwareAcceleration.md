---
layout: post
title: Hardware Acceleration
date: 2026-01-06
references:
  - title: Introduction to Hardware Acceleration CSS Animations
    url: https://www.sitepoint.com/introduction-to-hardware-acceleration-css-animations/
---

# Safari의 GPU 레이어 관리와 렌더링 문제

1. [개요](#개요)
2. [GPU 레이어 관리의 기본 개념](#gpu-레이어-관리의-기본-개념)
3. [Safari의 레이어 관리 철학](#safari의-레이어-관리-철학)
4. [Hardware Acceleration 미적용 문제](#hardware-acceleration-미적용-문제)
5. [레이어 승격이 안 될 때 요소가 사라지는 문제](#레이어-승격이-안-될-때-요소가-사라지는-문제)

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

브라우저는 성능 최적화를 위해 특정 요소를 별도의 **레이어**로 분리합니다. 승격되지 않은 요소는 부모 레이어에 같이 그려지고, 승격된 요소는 자기 레이어(텍스처)를 따로 갖습니다.

```
화면 구성:
┌─────────────────────────────────┐
│ Layer 1 (부모 레이어)              │
│  ├─ header                      │
│  └─ content                     │
├─────────────────────────────────┤
│ Layer 2 (승격된 레이어) - animated-box │
│  - 자기 텍스처를 따로 가짐            │
│  - transform/opacity 변경 시       │
│    다시 그리지 않고 합성만 함          │
└─────────────────────────────────┘

Paint(래스터)   각 레이어의 비트맵을 만듦       ← CPU 또는 GPU
Composite     모든 레이어를 겹쳐 한 장으로     ← GPU (컴포지터 스레드)
```

두 레이어 모두 GPU에서 합성됩니다. 차이는 승격된 레이어가 **따로 떨어진 텍스처**를 가진다는 점입니다. 그래서 승격된 요소의 `transform`/`opacity`가 바뀌면 부모 레이어를 다시 그리지 않고, 컴포지터가 위치와 투명도만 바꿔서 합성합니다.

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

## 레이어 승격이 안 될 때 요소가 사라지는 문제

4번은 승격되지 않아 매 프레임 다시 Paint하느라 **느린** 문제입니다.
이 섹션은 승격되지 않아 요소가 **보이지 않는** 문제입니다.

요소가 사라지는 원인은 브라우저 엔진이 "어느 레이어의 어느 영역을 다시 그릴지"를 잘못 판단한 경우입니다. 두 가지 경로가 있습니다.

### 1. 레이어 겹침 순서 문제

-   `<video>`나 `transform`이 걸린 요소는 자기 레이어를 가집니다.
-   그 위에 그려져야 하는 요소(자막, 오버레이, 버튼)가 승격되지 않으면 아래쪽 부모 레이어에 같이 그려집니다.
-   합성 순서상 부모 레이어가 video 레이어보다 아래에 있으므로, 오버레이가 video에 가려집니다.
-   엔진은 "승격된 레이어 위에 겹치는 요소는 같이 승격한다"는 겹침 판단을 합니다. 이 판단이 빠지면 요소가 사라져 보입니다.
-   오버레이에 `translateZ(0)`이나 `will-change`를 주면 자기 레이어가 생겨 video 레이어 위에서 합성되므로 다시 보입니다.

```
[승격 안 된 오버레이]
  부모 레이어 (content + 오버레이)   ← 아래
  video 레이어                      ← 위   → 오버레이가 가려짐

[오버레이 승격]
  부모 레이어 (content)
  video 레이어
  오버레이 레이어                   ← 맨 위 → 보임
```

### 2. 다시 그리기 범위 누락

-   승격되지 않은 요소가 바뀌면, 엔진은 부모 레이어에서 바뀐 영역만 다시 그립니다.
-   이 영역 계산이 틀리면 텍스처가 갱신되지 않아서, 이전 상태가 남거나 요소가 안 보입니다.
-   승격하면 그 요소의 텍스처를 따로 갱신하므로 이 계산을 거치지 않습니다.

### 주의

-   두 경로 모두 스펙에 정해진 동작이 아니라 엔진 구현(주로 WebKit)의 판단 차이나 버그에서 생깁니다. 증상이 어느 경로인지는 Safari Web Inspector의 Layers 탭에서 레이어 구성을 보고 확인합니다.
-   승격으로 우회하면 레이어마다 GPU 메모리를 점유합니다. 증상이 있는 요소에만 적용합니다. 레이어 비용은 [브라우저 렌더링 파이프라인과 컴포지팅](./browser-rendering-composite)의 "승격의 비용" 참고.
