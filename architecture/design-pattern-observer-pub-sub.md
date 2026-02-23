---
title: Observer vs Pub/Sub 패턴 비교
date: 2026-02-23
tags: [Design Pattern, Observer, Pub/Sub, Event Bus]
references:
    - title: 'GoF, Design Patterns: Elements of Reusable Object-Oriented Software'
      url: https://en.wikipedia.org/wiki/Design_Patterns
    - title: 'Addy Osmani, Learning JavaScript Design Patterns'
      url: https://www.patterns.dev/vanilla/observer-pattern/
    - title: 'Microsoft Azure, Publish-Subscribe pattern'
      url: https://learn.microsoft.com/en-us/azure/architecture/patterns/publisher-subscriber
---

# Observer vs Pub/Sub 패턴 비교

핵심 차이: 중간 브로커(Message Broker)의 유무

## Observer 패턴

Subject(발행자)와 Observer(구독자)가 **서로를 직접 알고 있으며**, Subject가 상태 변경 시 등록된 Observer들을 직접 호출합니다.

```
Subject ──── 직접 notify ────▶ Observer1
                          ────▶ Observer2
                          ────▶ Observer3
```

### 특징

-   Subject와 Observer가 **서로를 직접 참조**
-   **동기적(Synchronous)** 처리가 일반적
-   **같은 애플리케이션 스코프** 내에서 동작
-   결합도는 낮지만 **서로의 존재를 인식**

### 코드 예시

```typescript
interface Observer {
    update(data: any): void;
}

class Subject {
    private observers: Observer[] = [];

    subscribe(observer: Observer) {
        this.observers.push(observer);
    }

    unsubscribe(observer: Observer) {
        this.observers = this.observers.filter((o) => o !== observer);
    }

    notify(data: any) {
        // Subject가 Observer를 직접 호출
        this.observers.forEach((o) => o.update(data));
    }
}

class ConcreteObserver implements Observer {
    update(data: any) {
        console.log('Received:', data);
    }
}

const subject = new Subject();
const observer = new ConcreteObserver();

subject.subscribe(observer);
subject.notify({ message: 'Hello' }); // 직접 호출
```

### 대표 사례

-   RxJS `Subject`
-   Vue.js reactivity system
-   MobX observable

## Pub/Sub 패턴

Publisher와 Subscriber 사이에 **Event Bus(브로커)**가 존재하며, 발행자와 구독자는 서로를 전혀 알지 못합니다.

```
Publisher ──▶ [ Event Bus / Message Broker ] ──▶ Subscriber1
                                             ──▶ Subscriber2
                                             ──▶ Subscriber3
```

### 특징

-   Publisher와 Subscriber가 **서로의 존재를 모름** (완전한 분리)
-   **비동기적(Asynchronous)** 처리가 일반적
-   **다른 컴포넌트, 다른 서버 간**에도 동작 가능
-   브로커가 메시지 필터링, 라우팅, 큐잉을 담당

### 코드 예시

```typescript
type EventHandler = (data: any) => void;

class EventBus {
    private events: Record<string, EventHandler[]> = {};

    publish(event: string, data: any) {
        // Publisher는 Subscriber가 누군지 모름
        this.events[event]?.forEach((fn) => fn(data));
    }

    subscribe(event: string, fn: EventHandler) {
        // Subscriber는 Publisher가 누군지 모름
        if (!this.events[event]) this.events[event] = [];
        this.events[event].push(fn);
    }

    unsubscribe(event: string, fn: EventHandler) {
        this.events[event] = this.events[event]?.filter((f) => f !== fn);
    }
}

const bus = new EventBus();

// Subscriber — publisher를 전혀 모름
bus.subscribe('user:login', (data) => {
    console.log('User logged in:', data);
});

// Publisher — subscriber를 전혀 모름
bus.publish('user:login', { userId: 123 });
```

### 대표 사례

-   Redis Pub/Sub
-   Apache Kafka
-   RabbitMQ
-   Browser `CustomEvent` / `EventEmitter`
-   React 전역 이벤트 버스

## 비교 요약

| 구분      | Observer                     | Pub/Sub                       |
| --------- | ---------------------------- | ----------------------------- |
| 결합도    | 느슨하지만 서로 인식         | 완전히 분리 (서로 모름)       |
| 브로커    | ❌ 없음                      | ✅ Event Bus / Message Broker |
| 통신 방식 | 주로 동기                    | 주로 비동기                   |
| 동작 범위 | 같은 스코프 내               | 크로스 컴포넌트, 크로스 서버  |
| 확장성    | 상대적으로 낮음              | 높음                          |
| 복잡도    | 단순                         | 브로커 관리 필요              |
| 대표 사례 | RxJS Subject, Vue reactivity | Redis, Kafka, RabbitMQ        |

## 실전 시나리오: 주문 완료 시 여러 모듈이 반응해야 하는 경우

주문이 완료되면 다음이 동시에 일어나야 한다고 가정합니다.

1. 장바구니 비우기
2. 토스트 알림
3. 포인트 갱신
4. 주문 내역 갱신

### Observer로 구현할 경우

`order.js`가 각 모듈을 **직접 import하여 호출**합니다.

```typescript
// order.js — 모든 모듈을 직접 알고 있어야 함
import { clearCart } from '@/store/cart.js';
import { toast } from '@/store/toast.js';
import { refreshPoints } from '@/store/point.js';
import { refreshOrders } from '@/store/order-history.js';

function completeOrder() {
    // 주문 처리 로직...
    clearCart();
    toast.success('주문 완료');
    refreshPoints();
    refreshOrders();
}
```

기능이 추가될 때마다 `order.js`를 직접 수정해야 합니다.

```typescript
// 쿠폰 차감 추가 → order.js 수정
import { deductCoupon } from '@/store/coupon.js'; // 추가

function completeOrder() {
    clearCart();
    toast.success('주문 완료');
    refreshPoints();
    refreshOrders();
    deductCoupon(); // 추가 — order.js가 점점 비대해짐
}
```

`order.js`가 점점 모든 모듈에 의존하게 되고, 한 모듈의 변경이 `order.js`에 영향을 줍니다.

---

### Pub/Sub으로 구현할 경우

`order.js`는 이벤트만 발행하고, 각 모듈이 **독립적으로 구독**합니다.

```typescript
// order.js — eventBus만 알면 됨
import { eventBus } from '@/core/eventBus.js';

function completeOrder() {
    // 주문 처리 로직...
    eventBus.publish('order:completed', { orderId, items });
    // 구독자가 누군지, 몇 개인지 전혀 모름
}
```

```typescript
// cart.js
eventBus.subscribe('order:completed', () => clearCart());

// toast.js
eventBus.subscribe('order:completed', () => toast.success('주문 완료'));

// point.js
eventBus.subscribe('order:completed', () => refreshPoints());

// order-history.js
eventBus.subscribe('order:completed', () => refreshOrders());
```

기능이 추가될 때 `order.js`는 **전혀 수정할 필요 없습니다**.

```typescript
// 쿠폰 차감 추가 → coupon.js에서 구독만 추가
// order.js 수정 없음
eventBus.subscribe('order:completed', () => deductCoupon());

// 재고 갱신 추가 → stock.js에서 구독만 추가
// order.js 수정 없음
eventBus.subscribe('order:completed', () => refreshStock());
```

### 의존 방향 비교

```
// Observer
order.js ──▶ cart.js
         ──▶ toast.js
         ──▶ point.js
         ──▶ order-history.js
         ──▶ coupon.js        ← 추가될수록 order.js가 비대해짐

// Pub/Sub
order.js ──▶ [eventBus] ◀── cart.js
                        ◀── toast.js
                        ◀── point.js
                        ◀── order-history.js
                        ◀── coupon.js        ← order.js 변경 없음
```

### 패턴 선택 기준

| 구분         | Observer                      | Pub/Sub                    |
| ------------ | ----------------------------- | -------------------------- |
| 의존 방향    | 발행자 → 구독자 (직접 import) | 발행자 → 브로커 ← 구독자   |
| 기능 추가 시 | 발행자 코드 수정 필요         | 구독자만 추가하면 됨       |
| 디버깅       | 호출 흐름이 명확              | 이벤트 흐름 추적이 어려움  |
| 적합한 상황  | 관계가 고정적이고 단순할 때   | 반응 모듈이 계속 늘어날 때 |

> 💡 구독자가 고정적이고 소수라면 Observer가, 하나의 이벤트에 반응하는 모듈이 계속 늘어나는 구조라면 Pub/Sub 도입을 고려하세요.

---

### Pub/Sub의 파편화 문제

Pub/Sub은 모듈 간 결합도를 낮추는 대신 **흐름 추적이 어려워지는 트레이드오프**가 있습니다.

"주문 완료 시 무슨 일이 일어나는가?"를 파악하려면:

```
Observer  → order.js 하나만 보면 됩니다.
Pub/Sub   → "order:completed"를 구독하는 모듈을 프로젝트 전체에서 검색해야 합니다.
```

구독자가 늘어날수록 전체 흐름을 파악하기 어려워지고, 이벤트명 오타나 구독 누락 같은 실수도 **런타임에서야 발견**됩니다.

---

### 중간 지점: 유스케이스 함수 (오케스트레이터 패턴)

발행자도 구독자도 아닌, **흐름을 조율하는 별도 함수**를 두는 방식입니다.

```typescript
// usecase/completeOrder.ts — 오케스트레이터
// order.js는 주문 로직만 담당하고, 이 함수가 전체 흐름을 관리
import { clearCart } from '@/store/cart';
import { toast } from '@/store/toast';
import { refreshPoints } from '@/store/point';
import { refreshOrders } from '@/store/order-history';

export function completeOrder(orderId: string) {
    processPayment(orderId); // 주문 핵심 로직
    clearCart();
    toast.success('주문 완료');
    refreshPoints();
    refreshOrders();
}
```

```typescript
// order.js — 주문 로직만 담당, 다른 모듈을 알 필요 없음
import { completeOrder } from '@/usecase/completeOrder';

function handleOrderSubmit() {
    completeOrder(orderId);
}
```

Observer처럼 흐름이 한눈에 보이면서도, `order.js`가 구독자들을 직접 알 필요가 없어집니다.

### 세 가지 접근 방식 비교

| 접근                             | 장점                        | 단점                            |
| -------------------------------- | --------------------------- | ------------------------------- |
| Observer (직접 호출)             | 흐름이 명확, 타입 추적 가능 | 발행자가 비대해질 수 있음       |
| Pub/Sub (이벤트 버스)            | 모듈 간 완전 분리           | 흐름 파편화, 디버깅 난이도 상승 |
| 유스케이스 함수 (오케스트레이터) | 흐름 명확 + 발행자 분리     | 레이어가 하나 추가됨            |

> 💡 Pub/Sub은 "모듈이 수십 개 이상이고 이벤트에 반응하는 모듈이 계속 변동되는" 규모에서 빛을 발합니다. 그보다 작은 규모에서는 Observer나 유스케이스 함수로 흐름의 명확성과 결합도 분리를 모두 확보할 수 있습니다.

## 관계 정리

Pub/Sub은 Observer를 **확장·추상화**한 패턴입니다.

Observer 패턴의 단점인 **Subject-Observer 간 의존성 문제**를 브로커 도입으로 해결한 것이 Pub/Sub 패턴입니다.

```
Observer 패턴
  └── 확장 ──▶ Pub/Sub 패턴 (브로커 추가 → 완전한 디커플링)
```
