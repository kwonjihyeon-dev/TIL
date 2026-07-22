---
title: Playwright E2E 자동화 - storageState로 로그인을 한 번만 해도 되잖아 ?
date: 2026-07-22
tags: [Playwright, E2E, storageState, Authentication, SSO, OAuth]
references:
    - title: Playwright - Authentication
      url: https://playwright.dev/docs/auth
    - title: Playwright - Trace Viewer
      url: https://playwright.dev/docs/trace-viewer
---

# Playwright E2E 자동화: 로그인을 한 번만 해도 되잖아 ? (SSO + 서버 세션, 그리고 storageState의 한계)

> 주기적으로 E2E 테스트를 돌리고 실패하면 슬랙으로 알림받는 자동화를 준비하다가,
> "매 테스트마다 로그인하는 게 아깝다 → storageState로 한 번만 하면 되지 않을까?"에서 출발한 하루치 삽질 기록.
> 결론부터 말하면 **storageState는 이 상황에선 굳이 필요 없었고, 그 이유를 파는 과정을 기록한다.**

## 배경

-   목표: 웹 서비스의 코어 기능에 대해 E2E를 CI에서 주기적으로 실행하고 실패 시 슬랙으로 알림.
-   테스트 구조: spec 파일 = 하나의 시나리오(로그인 → 리소스 등록 폼 작성 → 제출). `test.describe.configure({ mode: 'serial' })` + `beforeAll`에서 만든 `page`를 스텝들이 공유.
-   각 spec은 앞부분에서 **접속 → 로그인 → 등록 페이지 진입**을 거친 뒤 폼을 채운다.

처음 든 의문: "spec마다 로그인 API를 태우는데, 굳이 여러번 인증할 필요가 있을까 ? 그리고 병렬로 진행 시 rate limit 괜찮나? storageState로 로그인 한 번만 하고 재사용하면 안 되나?"

## storageState가 뭔가

Playwright가 **로그인 후의 쿠키/localStorage를 파일로 저장**해두고, 이후 테스트가 "이미 로그인된 상태"로 시작하게 해주는 기능.

```ts
// setup 프로젝트에서 1회 로그인 후 저장
await page.context().storageState({ path: '.auth/user.json' });

// config에서 다른 프로젝트가 재사용
{
  name: 'authed',
  dependencies: ['setup'],          // setup 먼저 실행 보장
  use: { storageState: '.auth/user.json' },
}
```

저장되는 것은 **쿠키(httpOnly 포함)** 와 **origin별 localStorage** 두 가지. sessionStorage·메모리 토큰은 안 담긴다.

## 1단계: 우리 인증이 storageState로 재사용 가능한가?

먼저 로그인 상태가 "어디에" 저장되는지 확인해야 한다. 대상 앱은 하이브리드 구조였다:

-   **자격 확인만 외부 SSO에 위임** (`sso.auth-example.com`)
-   로그인 상태/접근 제어는 **백엔드(Laravel 계열)의 서버 세션 인증** (세션 ID를 담은 httpOnly 쿠키)

storageState로 실제 캡처해보니 인증 값이 **쿠키와 localStorage 양쪽에** 담겼다. 특히 **OAuth `access_token` / `refresh_token`** 이 쿠키·localStorage 모두에 저장돼 있었는데, 뒤에서 밝혀지듯 **이 토큰이 문제의 핵심**이 된다.

**교훈 ①**: 세션이 쿠키(특히 httpOnly)에 있으면 storageState로 재사용 가능하다. SSO 왕복은 setup에서 1회만 하면 되고, spec은 결과물(세션 쿠키)만 재사용한다. sessionStorage/메모리 토큰 방식이면 이 방법이 안 통한다.

## 2단계: 실험했더니 테스트가 깨졌다 → Trace 디버깅

setup 프로젝트 + spec의 로그인 스텝 제거 후 돌리니, 로그인이 아니라 **등록 폼 단계**에서 실패했다. 실패 지점이 실행마다 옮겨다녔다(Spec A step7 / Spec B step6 / step8...). **실패 지점이 움직이면 타이밍 레이스를 의심**한다.

Trace Viewer(`npx playwright show-trace`)로 파보니 두 개의 원인이 겹쳐 있었다.

### 원인 A — 로딩 스피너 + 뒤늦게 뜨는 모달 (렌더 타이밍)

등록 페이지(`/register`)는:

-   스크립트 로드까지 시간이 소요되어 넣어 둔 로딩 스피너가 **서버 렌더 시점부터** 화면 전체를 덮고(z-index 높음, 흰 반투명), `window.load` 이벤트에서 `display:none`으로 숨겨진다(DOM 제거가 아니라 숨김).
-   로딩이 끝난 뒤 안내 모달이 **비동기로** 뜬다.

원래 spec은 진입 직후 모달을 5초만 기다렸는데(`try/catch`), **로그인을 직접 하던 시절엔 로그인에 드는 수 초가 버퍼 역할**을 해줬다. storageState로 로그인을 건너뛰자 페이지에 더 빨리 도달 → 5초 창을 놓침 → 안 닫힌 오버레이가 이후 클릭을 가로채 타임아웃.

**해결**: "없어질 것"이 아니라 "있어야 할 것"을 기다린다.

```ts
await expect(page).toHaveURL(/register/, { timeout: 15000 });

// 스피너가 사라질 때까지 = window.load 완료까지 (제거 아닌 display:none 이므로 toBeHidden)
await expect(page.locator('#로딩스피너id값')).toBeHidden({ timeout: 30000 });

// 안내 모달은 뜨면 닫고 안 뜨면 통과, 마지막에 오버레이 소멸 확인
const alert = page.locator('#모달id값');
await alert.waitFor({ state: 'visible', timeout: 5000 }).catch(() => {});
if (await alert.isVisible().catch(() => false)) {
    await alert.locator('button.confirm').click();
    await expect(alert).toBeHidden({ timeout: 5000 });
}
await expect(page.locator('#모달id값')).toBeHidden({ timeout: 5000 });
```

이 수정 후 Spec A는 완전 통과, Spec B 단독도 통과했다.

### 원인 B — refresh 토큰 재발급(single-use rotation): 앞 spec의 특정 페이지 진입이 뒤 spec을 로그아웃시킨다

그런데 **Spec A → Spec B 직렬 실행**에선 Spec B가 여전히 깨졌다. 실패 스냅샷은 등록 페이지가 아니라 **로그아웃된 홈 화면**이었다. 반면 **Spec B 단독 실행은 전부 통과.** 격리 실험(Spec A에서 "외부 전송(다른 서브도메인 앱)" 스텝만 제외)으로 좁히니, **Spec A가 그 서브도메인 앱에 진입할 때만** Spec B가 깨졌다.

앱 소스를 열어 원인을 코드로 확정했다. 인증이 **두 겹**이었다:

-   **서버 세션**(프레임워크 세션 쿠키) — 페이지를 내려줄지 결정.
-   **OAuth access/refresh 토큰**(쿠키 + localStorage) — 클라이언트가 API 호출·로그인 상태 유지에 사용.

그리고 refresh*token이 **단회용(single-use rotation)** 이었다 — 한 번 갱신에 쓰이면 그 토큰은 무효가 되고 새 refresh_token이 발급된다. (코드 주석에도 *"같은 refresh*token으로 두 번째 갱신 호출은 403"* 이라고 명시돼 있었다.)

결정적으로 **앞 spec이 흐름 도중 토큰을 자동으로 재발급하는 지점**이 있었다. 그 순간 저장돼 있던 **refresh_token(R1)이 소비되고 새 R2가 발급**되는데, 이 갱신은 **라이브 컨텍스트에만 반영될 뿐 storageState 파일엔 안 써진다.** → 파일은 계속 죽은 R1을 들고 있게 된다.

> 우리 경우 그 "갱신 지점"은 특정 하위 앱(다른 서브도메인) 진입이었다. 최초 로그인 토큰엔 없는 앱 전용 마커를 미들웨어가 보고 토큰을 강제 재발급했다. 앱마다 트리거는 다르겠지만 — **"어떤 spec이 토큰을 재발급시키는 지점이 하나라도 있으면"** 공유 스냅샷은 깨진다는 게 핵심이다.

```
setup:   로그인 → access_token(A1) + refresh_token(R1), issued_at 없음 → .auth/user.json 저장
Spec A:  A1/R1 로드 → 외부 서브도메인 앱 진입 → 미들웨어: issued_at 없음 → 강제 재발급
         → issue-token(R1) → R1 단회 소비 + R2 발급 (Spec A 라이브 컨텍스트만 갱신)
         → .auth/user.json 은 여전히 R1 (파일 자동 갱신 안 됨)
Spec B:  .auth/user.json 로드 → stale A1/R1 → /register 진입
         → 서버 세션은 유효 → 페이지는 서빙됨
         → 전역 클라 스크립트가 /api 호출에 stale A1(Bearer) 첨부 → 401
         → 자동 갱신: issue-token(R1) → 403 (R1 이미 소비) → logout() → 로그아웃 리다이렉트
         → 등록 폼 사라짐 → 실패
```

**교훈 ②**: storageState는 **특정 시점 토큰의 스냅샷**이다. refresh_token이 단회용(쓰면 재발급되고 기존 것은 무효)인 앱에서, 앞 spec이 한 번이라도 토큰을 재발급하면 파일 속 토큰은 죽는다. 게다가 전역 클라이언트 스크립트가 "토큰 갱신 실패 → 자동 로그아웃"을 하기 때문에, **서버 세션이 멀쩡해도** 뒤 spec은 로그아웃 화면으로 튕긴다. "여러 spec이 하나의 storageState를 공유" 하는 구조는 이 앱에선 반드시 깨진다.

## 3단계: 그래서 storageState가 필요했나? (아니었다)

여기서 원래 질문으로 돌아왔다. "매 테스트마다 로그인이 아까우니 storageState로 한 번만" — 그런데:

-   spec은 `serial` + `page` 공유라 **로그인이 spec당 1번**이다(스텝마다가 아니라). spec 10개면 로그인 ~10회.
-   rate limit은 "서버 용량" 문제가 아니라 **로그인 시도를 너무 자주 못 하게 막는 보안(중복 횟수 제한)**이고, 관건은 **단위 시간당 횟수**다. 직렬 실행이면 로그인이 시간축에 흩어져 안전하다.
-   즉 **"테스트당 로그인(수십~수백 회)" 걱정은 애초에 해당 없었다.** 이미 spec당 1회였고, 직렬이면 rate limit에 안 걸린다.

그리고 storageState로 더 줄이려면 "스위트당 1회(전체 실행 통틀어 1번)"여야 하는데, 그건 **원인 B(토큰 재발급)** 때문에 불가능. 결국 로그인 횟수의 하한선은 **spec당 1회**이고, 그건 inline 로그인이든 spec별 storageState든 동일하다. **storageState가 로그인 수를 더 줄여주지 못한다.**

## 여담: "앞 spec 세션을 뒤 spec이 물려받게 하면?" — 되더라도 안티패턴

직렬이면 Spec A의 세션을 Spec B가 물려받아 재로그인을 아낄 수 있지 않나? 이론적으로 방법은 있다(아래는 **직접 검증하진 않은 추론**):

-   ❌ Spec A의 "로그인 시점" 세션을 넘기면 → 도중 재발급되며 기존 게 무효화돼 이미 죽어있음(원인 B와 동일).
-   ⚠️ Spec A의 `afterAll`에서 "끝난 시점"의 live 세션을 재저장해 넘기거나, worker-scoped fixture로 live 컨텍스트 자체를 공유하면 → **이론상 될 법하다.**

하지만 **애초에 이걸 하면 안 된다.** 두 가지 이유:

1. **E2E의 목적과 안 맞는다.** "사용자가 등록하고 다시 등록한다" 같은 실제 흐름이라면, 그건 **한 사용자의 한 세션**이므로 **하나의 spec 안에서** 이어서 하면 된다(세션은 그 컨텍스트에 자연히 유지). 여러 spec이 세션을 물려받는 건 어떤 사용자 행동도 아님 → "재로그인 아끼기".
2. **테스트 독립성을 깬다.** 세션을 물려주면 spec이 **순서로 결합**되고(A 뒤에 B), A가 실패/로그아웃으로 끝나면 **뒤 spec이 줄줄이 오염**되며(진짜 원인이 가려짐), **병렬도 불가**다.

E2E는 각 spec이 **독립적으로 로그인 → 독립적으로 성공/실패**해야 "어디가 깨졌는지"를 바로 짚을 수 있다. 아끼는 로그인은 미미한데(이미 spec당 1회) 잃는 건 크다. → **물려받기는 안 한다.**

## 최종 결론: 주기 자동화는 어떻게?

"자동화 = 병렬"이 아니다. 병렬 여부는 Playwright `workers` 설정이 정한다. (예: `workers: process.env.CI ? 1 : undefined` → CI에선 직렬)

| 방식                        | rate limit | 세션 충돌         | 계정 | 시간                 |
| --------------------------- | ---------- | ----------------- | ---- | -------------------- |
| **직렬 + 단일 계정** (권장) | 안전       | 없음              | 1개  | 느림(주기 실행엔 OK) |
| 병렬 + 단일 계정            | ❌ 버스트  | ❌ 서로 세션 파기 | 1개  | 빠름                 |
| 병렬 + 계정 분리            | 안전       | 없음              | N개  | 빠름                 |

-   **주기 실행은 아무도 기다리지 않으므로** wall-clock 시간이 별로 안 중요 → **직렬 + 단일 계정 + spec당 로그인**이 가장 단순하고 안전.
-   직렬이라 한 번에 한 spec만 활성 → 각 spec이 시작 시 새로 로그인 → 자기 실행 동안 항상 live 세션 → 토큰이 재발급돼도 문제없음.
-   스위트가 커져 병렬이 필요해지면 그때 **워커 수만큼 테스트 계정을 분리**한다.

정작 주기 실행에서 더 신경 쓸 건 로그인이 아니라 **매 실행마다 생성되는 테스트 데이터 누적**이었다. → teardown(정리 스텝) 또는 백엔드와 데이터 격리 협의가 필요.

## 다음에 이런 문제를 만나면 (체크리스트)

**"로그인을 한 번만 하고 재사용하고 싶다"**

1. 세션이 어디 저장되나? (DevTools → Application) 쿠키/localStorage면 storageState 가능, sessionStorage/메모리면 불가.
2. 재인증(다른 도메인 SSO, 재로그인) 지점이 있나? 있으면 그 이후 저장한 storageState는 낡는다.
3. 앱이 로그인/인증 시 세션·토큰을 재발급(migrate/regenerate 등)하나? 하면 공유 storageState는 재인증 spec 뒤에서 깨진다.

**"E2E가 flaky한데 원인을 모르겠다"**

1. 실패 지점이 실행마다 움직이나? → 타이밍 레이스 의심.
2. Trace Viewer로 실패 시점 **스냅샷(가운데 화면)** 을 본다. Console 로그가 아니라 실제 화면.
3. 클릭이 막히면 call log에서 **무엇이 intercept pointer events** 하는지 본다(스피너/오버레이/모달).
4. "없어질 것(로더)"을 기다리지 말고 "있어야 할 것(폼 요소)"을 기다린다. `toBeHidden`은 요소가 없을 때 즉시 통과하니, 아직 안 뜬 로더엔 헛통과할 수 있음(단, 서버 렌더로 처음부터 보이는 로더면 신뢰 가능).

**"CI에서 주기 실행 = 병렬로 같은 계정 로그인인가?"**

-   아니다. 병렬은 `workers` 설정. 주기 실행은 직렬로 충분하고, 직렬이면 rate limit·세션 충돌 둘 다 회피된다.
-   rate limit은 "시간당 횟수". 직렬이면 흩어져서 안전. (실제 임계값은 백엔드에 확인해두면 확실)

## 한 줄 요약

> storageState는 "세션이 쿠키에 있고, 재인증으로 토큰이 재발급되지 않는" 경우에 사용.
> 재인증(SSO/재로그인) 때 **토큰이 재발급되며 기존 것이 무효화되는** 앱에서는 **여러 spec이 하나의 storageState를 공유할 수 없다.**
> 그리고 serial + shared page 구조라면 로그인은 이미 spec당 1회 — rate limit 걱정도, storageState 도입 필요도 크지 않다.

## 끝으로: 테스트의 목적

이 작업은 "로그인을 1회로 줄이기"라는 인프라 최적화에서 시작했지만, 결론은 "필요 없다"였다.

-   E2E의 목적은 실제 사용자 여정을 **독립적으로** 검증하는 것이다.
-   storageState·세션 공유·chaining은 그 목적을 위한 수단일 뿐이다.
-   최적화가 앱 동작(토큰 재발급 → 스냅샷 노후)과 충돌하거나 테스트 독립성(spec 순서 결합)을 해치면, 목적에 맞는지부터 따져야 한다.
