---
layout: post
title: AppsFlyer
date: 2026-01-06
---

# Appsflyer

-   앱 마케팅 성과 측정 및 사용자 행동 분석을 위한 핵심 도구
-   하나의 링크로 OS(iOS/Android)를 자동 감지하여 앱 설치 시 앱 실행을, 미설치 시 스토어 이동 후 설치 완료 시 특정 페이지로 자동 연결(디퍼드 딥링킹)하는 통합 링크 솔루션을 제공 = 원링크

### 기획 요약

> 사용자가 원링크를 활용해서 소셜미디어로 공유 시 모바일 기기에서는 앱으로 이동(앱 미설치시 앱 설치로 이동), 데스크탑에서는 웹사이트로 이동되도록 기능 구현
>
> 1. 사용자가 링크를 클릭하면:
>     - 앱 설치됨 → 앱 실행 후 해당 매물로 이동
>     - 앱 미설치 → 스토어로 이동
> 2. 사용자에게 보여지는 링크는 짧아야 함 (단, 버튼인 경우는 긴 링크도 OK)

### 웹에서 해당 정책을 구현하기위해 찾아본 방식

1. 앱스플라이어 사이트에서 생성한 원링크를 기반으로 쿼리스트링 전달 후 사이트에 접속하는 방식<br/>
   공유 링크 생성: 특정 페이지에서 공유 버튼을 누르면, [OneLink 템플릿] 과 함께 원하는 정보를 쿼리 스트링으로 조합하여 최종 공유 URL을 생성 (deep_link_sub1 = 앱에 전달되는 값, 웹으로 접속 시 앱스플라이어에서 누락시킴)<br/><br/>
   **예시 URL 조합:&nbsp;** <br/>
   `[Short URL] + ?deep_link_sub1=[원하는 값] ...(원하는 값을 넣은 쿼리스트링)`

2. 데이터 파싱 (Parsing): 웹페이지 로딩 시, URL에 넘어온 쿼리 스트링(deep_link_sub1, af_sub1 등)을 파싱하여 건물 상세 정보 로드

    ```js
    // middleware.ts
    if (url.searchParams.has('특정 값')) {
        if (req.headers.get('user-agent')?.includes('Mobile')) {
            // 모바일 기기에서는 원링크로 앱이 열리도록 리다이렉트
            return NextResponse.redirect(new URL(원링크 URL));
        }

        return  NextResponse.redirect(이동시킬 페이지)
    }
    ```

    해당 방식으로 링크 접속 시,
    데스크탑에서 웹사이트로 이동, 모바일 기기에서는 앱 설치/앱으로 이동 확인

3. SmartScript 를 호출해서 원링크 생성 함수를 이용해 원링크 제작 → 링크 삽입하는 방식<br/>
   해당 방식은 [docs](https://dev.appsflyer.com/hc/docs/dl_smart_script_v2#about-onelink-smart-script)에도 나와있듯이 버튼에 삽입했을 때 사용하는 링크로 활용 ⇒ 즉, 전달하는 모든 파라미터를 담은 URL이고, shortURL을 생성해주는 함수는 아님.<br/><br/>
   적용해본 결과, 1번 방식과 동일한 결과를 확인했지만 링크가 길어지기때문에 링크 복사 기능에 사용하기에 부적절.<br/>
   \*\* [원링크에서 제공하는 API](https://support.appsflyer.com/hc/ko/articles/360001250345-OneLink-API%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%98%EC%97%AC-%EB%8C%80%EA%B7%9C%EB%AA%A8-%EC%BA%A0%ED%8E%98%EC%9D%B8%EC%97%90%EC%84%9C-%EB%A7%81%ED%81%AC-%EC%83%9D%EC%84%B1)를 사용해서 shortURL 만들어서 사용하는 방법도 있으나, 플랜 제한이 있음.

### 개발 진행하면서 확인한 추가 이슈

1.  페이스북 인앱 브라우저에서 앱 열기 시도 → (앱이 설치되어있어도) 앱 설치 링크로 이동
    [페이스북의 '인앱 브라우저' 환경이 앱 실행을 위한 딥링크(Deep Link) 신호를 차단하거나 제대로 전달하지 못하기 때문](https://support.appsflyer.com/hc/ko/articles/208874366-%EC%BA%A0%ED%8E%98%EC%9D%B8%EC%9D%84-%EC%9C%84%ED%95%9C-%EC%9B%90%EB%A7%81%ED%81%AC-%EC%83%9D%EC%84%B1%ED%95%98%EA%B8%B0#social-app-landing-page)이라는 추측.<br/>
    **→ af_force_deeplink=true 파라미터 추가해서 강제로 열도록 변경해서 해결**
    <br/>
    <br/>
2.  Short URL + 파라미터 추가하는 방식(위의 1번 방식)으로 링크 공유하기 후 데스크탑에서 해당 링크 클릭 시 공유할 때 설정한 쿼리스트링이 누락되고 앱스플라이어에서 등록된 URL로만 이동되는 문제가 있음 ( → 쿼리스트링이 누락되었으니 next 미들웨어에서 의도한 페이지로 이동할 수 없음)

        ```
        // 페이스북 공유 시 흐름

        1. 사용자가 원링크 공유
        ${원링크 URL}?af_sub1=...

        2. AppsFlyer 서버에서 User-Agent 확인

        3. 페이스북 크롤러 감지 시
        → af-preview 페이지로 301 리다이렉트
        ${원링크 URL}/af-preview/facebook?url=...&ios_dp=...&android_dp=...
        → 최종 목적지로 302 리다이렉트

        4. 게시글 공유했을 때 최종 목적지 + 페이스북 자체적으로 URL을 변경키는 걸로 보임.
        → 데스크탑: url 파라미터의 값으로 최종 리다이렉트
        → 앱: 유저 에이젼트별로 앱 이동
        ```

        **해결한 방식은,**<br/>

        그래서 공유하는 링크 자체를 변경해서 위 흐름을 타지 않고 ( `공유하기 리다이렉트 시킬 특정 페이지 URL` 공유 → 미들웨어 타도록) Next Middleware에서 판단할 수 있게 플로우 변경.

    <br/>
    <br/>

### ⚠️ 참고 사이트<br/>

카카오톡 공유하기 디버깅 도구: https://developers.kakao.com/tool/debugger/sharing<br>
페이스북 공유하기 디버깅 도구: https://developers.facebook.com/tools/debug<br/>
페이스북 크롤러 디버깅 방법(크롬 기준):<br/>개발자 도구 → 네트워크 컨디션스 → `facebookexternalhit/1.1 (+http://www.facebook.com/externalhit_uatext.php)` 설정 시 페이스북 공유하기를 통해 리다이렉트되는 과정을 확인할 수 있음.
