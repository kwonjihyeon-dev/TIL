---
title: A/B 테스트 적용기
date: 2026-05-12
tags: [A/B Test, Amplitude, Feature Flag, IntersectionObserver, Vue, Funnel Analysis, Experiment]
references:
  - title: Evan Miller A/B Test Calculator
    url: https://www.evanmiller.org/ab-testing/chi-squared.html
---

# A/B 테스트 적용기

> **환경/제약**
>
> - Amplitude **Growth v2** 플랜: Experiment entity (Analysis 탭에서 통계 유의성을 자동 검정해주는 기능) 불가. 플래그 페이지의 _Convert to Experiment_ 버튼이 막혀 있다. (Feature Flag만 사용)
> - 클라이언트 사이드 SPA (Vue2). variant 분기는 v-if로 처리.
> - dev/prod는 Amplitude **프로젝트 단위로 분리**되어 있어, flag도 두 프로젝트에 각각 만들어야 한다.

이 글은 위 제약 안에서 **Feature Flag 단계까지만 쓸 수 있는 환경에서 A/B 테스트를 어디까지 굴려봤나**에 대한 회고. "이상적 가이드"가 가정하는 단계 중 못 한 게 여럿이라, 본문은 "한 것"과 "못 한 것 / 다음엔 할 것"을 같이 적어둔다.

---

## 0. 가설과 사전 설계

이번 테스트 대상은 **신규 기능 진입 배너의 노출 위치**. 한 변수만 바꾼다 — 디자인은 동일, 위치만 control(페이지 상단) vs treatment(페이지 하단부 섹션 사이).

| 항목                      | 현재                                                              |
| ------------------------- | ----------------------------------------------------------------- |
| 한 번에 한 변수만 변경    | ✅ 위치만                                                         |
| 가설 한 문장으로 문서화   | ⚠️ 명시적으론 안 남김 — 다음엔 PR 본문이나 이 문서 0번에 박아두기 |
| Baseline 전환율 측정      | ✅ 측정 (~0.0216%, mount 기반) → viewport 기반으로 재측정 예정    |
| MDE / 표본 크기 사전 계산 | ✅ 합격선 0.44% (통상 배너 평균 수준) 합의                        |
| 종료 조건 합의            | ✅ viewport 이벤트 배포 후 최소 1주 운영, peeking 금지            |

> **참고 — 통상 배너 CTR 수준**
>
> | CTR       | 의미                           |
> | --------- | ------------------------------ |
> | 0.05~0.5% | 일반 디스플레이 배너           |
> | 1~2%      | 컨텐츠 내 CTA / 통상 배너 평균 |
> | 3~5%      | 잘 디자인된 맥락형 CTA         |
>
> 0.44%는 일반 배너 하단~컨텐츠 CTA 사이의 보수적 합격선. 3~5%는 위치 변경만으로 달성하긴 비현실적인 영역.

→ **이번 케이스는 운영 중에 보정을 거치며 ⚠️ 대부분을 채워 넣었지만**, 차회엔 0번 단계에서 미리 합의하는 게 의사결정이 흔들리지 않는 길.

---

## 0.5 측정 도구 점검 — viewport view 이벤트 도입

운영 12일쯤 데이터를 들여다보다 베이스라인 CTR이 **0.0216%**로 나오는 걸 확인했다. 통상 배너 CTR(0.05~0.5%)보다 두 자릿수 낮은 수치. 위치 비교 이전에 측정 자체가 의심스러웠다.

### 가설 — 분모가 부풀려져 있다

기존 view 이벤트는 배너 컴포넌트가 mount되는 순간 발화. 즉 "페이지 진입 = 1 view"였다. 그런데:

- **control(상단)**: 위에 이미지/가격/뱃지 섹션이 있어 일정 스크롤 필요. iPhone 14 Pro 같은 디바이스에선 첫 화면에 안 보이는 경우 있음
- **treatment(하단부)**: 더 많이 스크롤해야 보임. 거기까지 안 내려가면 못 봄

view 카운트 중 상당수가 _"본 게 아니라 페이지에 진입한 것"_. 진짜 본 사용자만 분모로 잡으면 CTR은 훨씬 높을 것.

### 그래도 비율 비교만 하면 안 되나 — 함정

"양쪽 다 같은 방식으로 부풀려져 있으니 비율은 보존되지 않나"라는 접근을 검토했는데, **위치별 가시성 비율이 다르면 결론의 방향까지 뒤집힐 수 있다**.

기호:

- `p_c`, `p_t` = control / treatment 위치에서 mount된 사용자 중 실제로 본 비율
- `CTR_*_mount = p_* × CTR_*_view`

```
CTR_t_mount / CTR_c_mount = (p_t / p_c) × (CTR_t_view / CTR_c_view)
```

비율이 보존되려면 `p_c == p_t`. 우리처럼 위치 차이가 크면 거의 확실히 깨지는 가정.

숫자 예시:

- `p_c = 0.7`, `p_t = 0.3` (직관 추정치)
- 실제 viewport 기준 CTR — control 1%, treatment 2% (treatment가 진짜 2배 좋음)
- mount 기반 측정 — control 0.7%, treatment 0.6% (**treatment가 나쁘게 보임**)

진짜 효과 크기가 잘려나가는 정도가 아니라 _방향이 뒤집힐 수 있는 종류의 편향_.

### 결정 — viewport 기반 새 이벤트 추가, A/B는 유지

세 옵션을 비교했다:

| 옵션                                  | 무엇을                       | 트레이드오프                             |
| ------------------------------------- | ---------------------------- | ---------------------------------------- |
| A. 그대로 진행                        | mount 기반, 비율 비교        | 위치별 편향이 결론 방향을 흔들 위험      |
| **B. viewport 이벤트 추가, A/B 유지** | 측정 도구만 정밀한 걸로 교체 | 새 이벤트 배포 이전 데이터는 분석 미사용 |
| C. A/B OFF → 측정 교체 → 재시작       | 측정 일원화                  | 일정 2~3주 추가, 다운타임                |

→ **B 선택.** A/B 실험의 "1 변수" 원칙은 **실험 조건(위치)** 에 대한 규칙이지 **측정 도구** 에 대한 게 아니다. 잣대를 더 정확한 걸로 교체하는 건 조건 변경이 아님. 다만 변경 시점 이전 데이터는 분석에 사용하지 않는다 (베이스라인 이해 용도로만 참고).

### 구현

배너 컴포넌트에 `IntersectionObserver`로 viewport 진입 + 1초 dwell 시 새 이벤트가 1회 발화. 임계값 / dwell은 IAB viewability 표준(50% 픽셀 / 1초)에 맞춤. 하단 fixed CTA 영역은 `rootMargin`으로 유효 viewport에서 제외해 "CTA에 가려진 상태에서 시간 흐름"이 카운트되지 않도록.

기존 view 이벤트는 그대로 두어 다른 분석 / 대시보드 호환성 유지. 의사결정 퍼널만 새 이벤트 기반으로 교체.

---

## 1. Amplitude 플래그 설정 — 실제로 한 것

### Step 1. Feature Flag 생성

- 메뉴: `Experiment > Feature Flags > 새로운 기능 플래그`
- 프로젝트: **Dev 프로젝트** (검증 후 Prod로 복제)
- 키: `feature-position-test` _(이 자리에 실제 flag key가 들어간다.)_
- 평가 모드: **원격(remote)** — 서버에서 평가, 클라이언트는 결과만 받음

키는 활성화 후엔 코드 곳곳에 박혀 바꾸기 어렵다. 6개월 뒤에 봐도 의미가 통하는 이름으로.

### Step 2. Variant

`control`, `treatment` 두 개. payload는 안 씀 — 단순 문자열 분기로 충분하다. 입문 단계에 payload 까지 끼우면 디버깅이 어려워진다.

### Step 3. Allocation

- Rollout 대상: **100%** (전체 트래픽)
- Variant 비율: **control 50 / treatment 50**

표본을 가장 빨리 모으는 조합. 새 경험에 트래픽 절반을 보내는 게 부담이면 90/10도 옵션이지만, 표본 모이는 시간이 길어진다.

### Step 4. Targeting

조건 **없음** — 전체 사용자 대상. 첫 실험은 단순하게. 세그먼팅을 걸면 표본이 줄고 분석이 복잡해진다.

### Step 5. ⚠️ Convert to Experiment — 못 함

| 기능                               | Feature Flag | Experiment |
| ---------------------------------- | ------------ | ---------- |
| variant 분기                       | ✅           | ✅         |
| 통계 분석 (p-value, 신뢰구간 자동) | ❌           | ✅         |
| Experiment Results 차트            | ❌           | ✅         |
| 표본 크기 추정 도우미              | ❌           | ✅         |

Growth v2 플랜에선 변환 자체가 비활성. **Primary / Secondary / Guardrail metric을 플래그에 연결**하는 단계도 함께 막힌다. 그래서 통계는 **Funnel 차트 + 외부 계산기** 조합으로 우회 (Part 4).

### Step 6. QA 테스터 등록

userId 기반으로 양쪽 variant에 별도 계정 등록:

- `control` 쪽: userId A
- `treatment` 쪽: userId B

각 계정으로 로그인 → 상세 페이지 진입 → **의도한 위치에 배너 노출되는지** 확인. 동시에 Amplitude **Live event stream**에서 `banner_view` / `banner_click` 이벤트의 `variant` 프로퍼티가 의도한 값으로 트래킹되는 지 함께 검증 (필수)

#### deviceId 대신 userId로 등록한 이유

특정 사용자를 타겟하는 게 아니고 control/treatment 분기에 배너가 정상적으로 노출되는 지 확인하는 게 목적.
<br/>deviceId 기반은 디바이스 단위로 강제 배정이라 로그인 여부와 무관하게 일관되지만 userId는 사용자가 로그인하면 지정된 값이 반영되면서 위치가 달라질 수 있지만 이번 검증 범위엔 굳이 필요 없음.

---

## 2. 프론트엔드 통합 — 실제 적용 패턴

### 2.1 SDK 노출

Amplitude Experiment SDK는 서버 레이아웃에서 한 번 초기화하고, **클라이언트 코드가 접근할 수 있도록 전역 변수에 핸들을 할당해둔다**. Vue 인스턴스 생성 전에 시작되어, 컴포넌트가 마운트될 즈음엔 핸들이 준비된 상태가 되도록.

### 2.2 트래킹 모듈 헬퍼

variant 조회는 컴포넌트가 SDK를 직접 만지는 대신, 공통 트래킹 모듈에 얇은 헬퍼 두 개를 둔다:

- `onExperimentReady(callback)` — SDK가 첫 fetch를 마친 시점에 콜백 실행. SDK 핸들이 없거나 차단된 환경에선 즉시 콜백.
- `getVariant(flagKey, fallback)` — 캐시된 variant 동기 조회. 두 번째 인자가 SDK 실패 시 fallback.

flag key는 상수 객체로 한 군데에 모은다.

```js
export const FLAG_KEYS = {
  POSITION_TEST: 'feature-position-test', // 이 자리에 실제 flag key가 들어간다
};
```

이렇게 두면 컴포넌트는 "원하는 flag key + fallback"만 알면 되고, SDK 변경에도 영향을 안 받는다.

### 2.3 컴포넌트에서 variant 읽기

상세 페이지 진입 컴포넌트의 `data()` 초기값은 안전한 기본값 `'control'`로 시작한다 (`'treatment'`로 설정해도 무관 — 어차피 다음 문단의 `isExperimentReady` 가드 때문에 SDK 응답 전엔 렌더가 안 된다). 이후 `mounted()` 시점에 `onExperimentReady`로 SDK 준비를 기다린 뒤 `getVariant`로 실제 값을 받아 덮어쓴다.

여기서 한 가지 트릭 — **`isExperimentReady`라는 별도 플래그를 두고, 그게 `true`가 될 때까지 배너 자체를 렌더하지 않는** 가드를 끼운다. 그렇지 않으면 SDK가 돌아오기 전에 초기값(`control`) 기준으로 한 번 그렸다가 곧이어 `treatment`로 자리를 옮기는 식의 **레이아웃 점프**가 사용자에게 보인다.

### 2.4 템플릿 분기 패턴

상세 페이지 템플릿엔 배너 컴포넌트가 **두 군데** 박혀 있다 — 하나는 상단(`control`), 하나는 하단부 섹션 사이(`treatment`). variant 값에 따라 둘 중 하나만 렌더된다.

```vue
<Banner v-if="variant === 'control'" :variant="variant" />
...
<Banner v-if="variant === 'treatment'" :variant="variant" />
```

단일 컴포넌트에 `position` prop을 넘기는 패턴 대신 이 방식을 고른 이유는, **두 위치 사이의 페이지 구조 자체가 너무 달라서** 컴포넌트 내부 prop으론 표현이 안 됐기 때문. 위 옵션과 아래 옵션은 그냥 "위로 옮기기/아래로 옮기기"가 아니라 주변 섹션, 여백, 등장 시점이 다 다른 별개의 자리다. 단점은 분기가 두 군데에 박혀 있어 늘리거나 줄일 때 두 군데 모두 고쳐야 한다는 점.

---

## 3. 이벤트 트래킹 — 분석을 가능하게 만든 핵심

분기만 해놓고 끝나면 결과를 못 본다. **view / click 두 이벤트에 모두 `variant` 프로퍼티를 박는** 게 이번 우회법의 핵심.

- `banner_view` _(이 자리에 실제 view 이벤트명)_ — 배너 컴포넌트 mounted 시점에 발화. mount 기반이라 가시성 편향 있음. 다른 분석 호환용으로 유지.
- `banner_visible_view` _(이 자리에 실제 visible_view 이벤트명)_ — 배너가 viewport 50% 진입 + 1초 dwell 시 1회 발화. **분석 primary는 이쪽.** `variant` 프로퍼티 포함.
- `banner_click` _(이 자리에 실제 click 이벤트명)_ — 클릭 핸들러에서 발화. 동일하게 `variant` 포함.

두 이벤트에 같은 분기 키가 박혀 있으니, Funnel 차트에서 "Step 1의 variant 값으로 사용자를 가르고 각 그룹의 view→click 전환율을 본다"가 가능해진다.

> ⚠️ Experiment 변환이 됐다면 `$exposure` 이벤트가 자동으로 user property를 세팅해 분석이 자동화됐겠지만, Growth v2에선 위와 같이 **이벤트 프로퍼티로 우회**해야 한다. 자동화의 빈자리를 트래킹 단의 규율로 메우는 것.

---

## 4. 결과 분석 — Funnel 차트 우회법

### 4.1 차트 타입은 Funnel

Segmentation 차트로 단순 클릭 수만 봐도 되지 않냐 — 안 된다. 두 그룹의 노출 수가 정확히 같지 않을 수 있어 단순 카운트 비교는 위험하다. **Funnel Analysis**가 분모/분자를 자동으로 잡아준다.

`Diagnosis` → **'차트로 보기'** 버튼 클릭.

### 4.2 두 이벤트 등록

```
Step 1 (먼저 발생):  banner_visible_view   ← 실제 viewport 노출
Step 2 (그 다음):    banner_click          ← 클릭
```

순서는 **"in this order"**. 노출 없이 클릭만 한 케이스가 끼면 분석이 깨진다.

> **참고용 3단계 차트** — 위치별 가시성 비율(`p_c`, `p_t`)이 궁금하면 별도 차트로 `banner_view → banner_visible_view → banner_click` 3단계 퍼널을 따로 구성. 1→2 전환율이 가시성 비율, 2→3이 진짜 CTR. 의사결정은 위 2단계 퍼널로만.

### 4.3 핵심 — "그룹 변환 기준 (Group Conversion By)"으로 variant 가르기

Funnel 차트엔 그룹핑 옵션이 두 가지 있는데, A/B 테스트엔 **"그룹 변환 기준"**이 정답.

| 옵션                                     | 동작                                                                                        |
| ---------------------------------------- | ------------------------------------------------------------------------------------------- |
| **그룹 변환 기준** (Group Conversion By) | Step 1 시점의 variant 값으로 사용자를 가르고, 그 그룹의 view → click 전환율을 **독립 계산** |
| 그룹화 (Group By)                        | 각 step마다 별도 그룹핑 — 같은 사용자가 step1/step2에서 다른 그룹에 속하는 모순 가능        |

설정 경로: `Step 1 → 그룹 변환 기준 → 이벤트 속성 → variant`

이렇게 하면 차트가 control / treatment **두 줄로 갈라져** 표시된다. Step 2엔 따로 그룹핑 안 걸어도 된다 — Step 1의 분기를 그대로 따라간다.

### 4.4 결과 보는 법

데이터가 1~2주쯤 쌓이면 차트가 두 줄로:

```
control    :  view 1,000  →  click  80   (CTR  8.0%)
treatment  :  view 1,000  →  click 120   (CTR 12.0%)
```

CTR이 높은 쪽이 더 많이 클릭된 위치. 그 차이가 "**우연**"이 아닌지 다음 단계로 검증.

### 4.5 통계 유의성은 수동

자동 검정이 막혔으니 p-value는 외부 계산기로.

- 외부: [Evan Miller A/B Test Calculator](https://www.evanmiller.org/ab-testing/chi-squared.html) — view/click 수만 넣으면 끝
- Python: `scipy.stats.chi2_contingency` 한 줄

표본이 충분히 쌓이고(변형마다 최소 수백~수천 건 view) p-value가 0.05 미만이면 의사결정.

---

## 5. Fallback과 OFF 동작 매트릭스

플래그를 끄거나, SDK가 실패하거나, 네트워크가 느려서 fetch 전에 마운트가 끝났을 때 — 어떤 variant가 보이나? 코드의 fallback 값과 Amplitude 설정의 조합 매트릭스.

| Amplitude 상태                | 코드 fallback | 결과                  |
| ----------------------------- | ------------- | --------------------- |
| ON, 100/0 rollout (control만) | `'control'`   | 모두 control          |
| ON, 50:50                     | `'control'`   | 50:50 분배 (정상 A/B) |
| OFF (deployment 토글 꺼짐)    | `'control'`   | 모두 control          |
| OFF                           | `'treatment'` | 모두 treatment        |
| ON 인데 SDK 차단 / fetch 실패 | fallback 값   | fallback 값만 노출    |

**핵심**: Amplitude의 "OFF"는 "서버가 정한 기본 variant"를 의미하지 않는다. release 타입 flag에는 그런 1급 설정이 없다. OFF면 SDK는 응답을 못 받고, 그 자리에 코드의 fallback이 그대로 들어간다.

→ "비활성 상태일 때 보여줄 위치를 control이 아닌 다른 곳으로 두고 싶다"면 두 가지 길:

1. **Amplitude는 ON 유지, rollout weights를 100/0으로 몰기** — UI에서만 통제. 가장 단순.
2. **Amplitude OFF + 코드 fallback 값을 원하는 variant로 교체** — 코드 배포 1회 필요.

---

## 6. 다음 적용 시 체크리스트

### 사전 (Part 0)

- [ ] 가설 한 문장으로 문서화
- [ ] Baseline 전환율 1~2주 측정
- [ ] **측정 도구가 실험 변수에 영향받는지 점검** — 예: 위치 실험에 mount 기반 view = 위치별 가시성 차이가 분모에 흡수됨
- [ ] MDE / 표본 크기 계산 (Evan Miller) + **합격선(목표 CTR) 사전 합의**
- [ ] 종료 조건을 표본 또는 기간 숫자로 합의

### Amplitude (Part 1)

- [ ] Dev 프로젝트에 flag 생성, key 명명 신중히
- [ ] Variant 정의, Allocation, Targeting
- [ ] **Prod 프로젝트에 같은 key로 복제** — 빠지면 prod에선 fallback만 나옴
- [ ] (가능하면) QA 테스터 등록해 양쪽 화면 사전 검증

### 프론트엔드 (Part 2-3)

- [ ] SDK 핸들을 전역 변수에 할당, 페이지 마운트 전 시작
- [ ] 트래킹 모듈 헬퍼(`onExperimentReady`, `getVariant`) 통해 일관되게 조회
- [ ] data 초기값 = 안전한 fallback
- [ ] `isExperimentReady` 가드로 레이아웃 점프 방지
- [ ] view / click 이벤트 모두에 `variant` 프로퍼티 첨부
- [ ] **가시성이 위치마다 다른 실험은 viewport 기반 view 이벤트 도입** (`IntersectionObserver` + threshold + dwell)

### 분석 (Part 4)

- [ ] Funnel 차트, "그룹 변환 기준 = variant"
- [ ] 외부 계산기로 p-value
- [ ] Peeking 금지 — 표본 도달 전엔 결과 안 봄

---

## 정리

Growth v2 같은 제약 안에서도 **이벤트 프로퍼티에 `variant` 박기 + Funnel 차트 + 외부 통계 계산** 패턴으로 A/B 테스트를 적용해봤다. 다만 자동화된 부분이 적은 만큼 **사전 설계(가설/표본/종료 조건)**가 더 중요해진다. 다음에 또 비슷한 테스트 할 일이 있으면 이 문서의 0번과 6번을 먼저 펼쳐볼 것.
