거주하기 프로젝트에서 건물 상세 화면 공유 작업에서 생긴 이슈를 정리해둔 페이지입니다.

관련 티켓 - REVIEW-114: [WEB] 거주후기 건물 상세 공유 기능 OneLink 동적 링크 생성 및 OG Tag 데이터 전송
작업 진행 중

Message from 개발*백엔드*김진수 in #거주후기

위 슬랙을 참고해서 요약해서 보자면,

공유하기 기능에서 목표했던 정책
사용자가 링크를 클릭하면:

앱 설치됨 → 앱 실행 후 해당 매물로 이동

앱 미설치 → 스토어로 이동

사용자에게 보여지는 링크는 짧아야 함 (단, 버튼인 경우는 긴 링크도 OK)

테스트 결과
iOS 제약으로 웹에서 만든 짧은 링크(매물 상세에서 구현된 방식 → 백엔드를 통해 직접 shortURL 생성)로는 모바일 앱 실행 불가

카카오 공유처럼 버튼에 긴 롱링크를 직접 넣으면 작동함

앱에서 생성한 짧은 링크는 정상 작동(SDK 로 shortURL 생성 가능)

웹에서 찾아본 방식은

1. 앱스플라이어 사이트에서 생성한 원링크를 기반으로 쿼리스트링 전달 후 사이트에 접속하는 방식

공유 링크 생성: 건물 상세 페이지에서 공유 버튼을 누르면, [OneLink 템플릿] 뒤에 해당 건물의 deep_link_sub1, af_sub2 (ID)와 af_sub1 (정보) 값을 쿼리 스트링으로 조합하여 최종 공유 URL을 생성 (deep_link_sub1 = 앱에 전달되는 건물 상세 id, 웹으로 접속 시 앱스플라이어에서 누락시킴)

예시 URL 조합:

[Short URL] + ?deep_link_sub1=[건물ID] &af_sub2=[건물ID] &af_sub1=[건물정보]
데이터 파싱 (Parsing): 웹페이지 로딩 시, URL에 넘어온 쿼리 스트링(deep_link_sub1, af_sub1)을 파싱하여 건물 상세 정보 로드

// apps/dongne-life/middleware.ts
// TODO: 조건은 더 확장성있게 변경할 필요 있음
if (url.searchParams.has('af_sub2') && url.searchParams.has('af_sub1')) {
const af_sub1 = url.searchParams.get('af_sub1');
const deep_link_sub1 = url.searchParams.get('af_sub2');
if (req.headers.get('user-agent')?.includes('Mobile')) {
// 모바일 기기에서는 원링크로 앱이 열리도록 리다이렉트
return NextResponse.redirect(
new URL(
`y48a036s?deep_link_sub1=${deep_link_sub1}&af_sub2=${deep_link_sub1}&af_sub1=${af_sub1}`,
'https://peterpanztown.onelink.me/nFCi/',
),
);
}
}
해당 방식으로

데스크탑에서 링크 접속 시, 정상적으로 건물 상세 진입

모바일 기기에서는 앱 설치로 이동 확인

\*\* 해당 링크 정도의 길이는 공유하기에 적절하다고 논의되었음 (기획 확인 - Message from 개발*프론트엔드*권지현 in #거주후기).

그러나 현재 개발서버는 IP 차단되어있어 크롤링봇이 메타 태그를 읽지 못하는 문제가 있는 것으로 예상됨. stage 서버 배포 후 확인 필요. → 26.01.02 /static 목업 화면 만들어서 IP 제한 해제하고 테스트 진행해서 메타태그 수집 확인

SmartScript 를 호출해서 원링크 생성 함수를 이용해 원링크 제작 → 링크 삽입하는 방식

해당 방식은 docs에도 나와있듯이 버튼에 삽입했을 때 사용하는 링크로 활용 ⇒ 즉, 전달하는 모든 파라미터를 담은 URL이고, shortURL을 생성해주는 함수는 아님.

// 원링크 생성 함수 관련 코드
const generateOneLinkParams = () => {
// const { score, displayName, dong } = getShareUrl();
const params = {
oneLinkURL: ONELINK_URL,
afParameters: {
mediaSource: { keys: ['mediaSource'], defaultValue: 'af_app_invites' },
campaign: { keys: ['c'], defaultValue: 't_pcweb_share' },
deepLinkValue: { keys: ['itemId'], defaultValue: buildingId || '' },
afCustom: [
{ paramKey: 'af_dp', keys: ['af_dp'], defaultValue: 'peterpan://main' },
{
paramKey: 'af_web_dp',
keys: ['af_web_dp'],
defaultValue: `${process.env.NEXT_PUBLIC_PETERPANZ_TOWN_URL}/building/${buildingInfo}/${buildingId}`,
},
{ paramKey: 'af_sub1', keys: ['af_sub1'], defaultValue: buildingInfo || '' },
{ paramKey: 'af_sub2', keys: ['af_sub2'], defaultValue: buildingId || '' },
{ paramKey: 'af_og_title', keys: ['af_og_title'], defaultValue: `${displayName}|${dong}거주후기` },
{
paramKey: 'af_og_description',
keys: ['af_og_description'],
defaultValue: `후기평점${score}`,
},
{
paramKey: 'af_og_image',
keys: ['af_og_image'],
defaultValue: `https://dev.img.peterpanz.com/reviews/20251216/42e21d7d1fc70d6b272e7f622e6d7235.jpeg`,
},
],
},
};
const { clickURL } = window.AF_SMART_SCRIPT.generateOneLinkURL(params);
console.log(clickURL);
return clickURL;
};
적용해본 결과, 1번 방식과 동일한 결과를 확인했지만 링크가 길어지기때문에 링크 복사 기능에 사용하기에 부적절.

원링크에서 제공하는 API를 사용해서 (OneLink API를 사용하여 대규모 캠페인에서 링크 생성) shortURL 만들어서 사용하려고 했으나, appsflyer 플랜 제한으로 불가능

추가 이슈
페이스북 인앱 브라우저에서 앱 열기 시도 → (앱이 설치되어있어도) 앱 설치 링크로 이동

페이스북의 '인앱 브라우저' 환경이 앱 실행을 위한 딥링크(Deep Link) 신호를 차단하거나 제대로 전달하지 못하기 때문이라는 추측(캠페인을 위한 원링크 생성하기, 아래 이미지 참고).

=> af_force_deeplink=true 파라미터 추가해서 강제로 열도록 변경해서 해결

스크린샷 2026-01-06 오전 10.43.46.png
Short URL + 파라미터 추가하는 방식(
원링크를 활용한 공유하기 작업 | 웹에서 찾아본 방식은
1번 방식)으로 링크 공유하기 후 데스크탑에서 해당 링크 클릭 시 공유할 때 설정한 쿼리스트링이 누락되고 앱스플라이어에서 등록된 URL로만 이동되는 문제가 있음 ( → 쿼리스트링이 누락되었으니 next 미들웨어에서 의도한 페이지로 이동할 수 없음)

// 페이스북 공유 시 흐름

1. 사용자가 원링크 공유
   https://peterpanztown.onelink.me/nFCi/y48a036s?af_sub1=...
2. AppsFlyer 서버에서 User-Agent 확인
3. 페이스북 크롤러 감지 시
   → af-preview 페이지로 301 리다이렉트
   https://peterpanztown.onelink.me/af-preview/facebook?url=...&ios_dp=...&android_dp=...
   → 최종 목적지로 302 리다이렉트
4. 게시글 공유했을 때 최종 목적지 + 페이스북 자체적으로 URL을 변경키는 걸로 보임.
   → 데스크탑: url 파라미터의 값으로 최종 리다이렉트
   → 앱: 유저 에이젼트별로 앱 이동
   해결한 방식은,
   그래서 공유하는 링크 자체를 변경해서 위 흐름을 타지 않고 ( /share/[building-info] 공유 → 미들웨어 타도록) Next Middleware에서 판단할 수 있게 플로우 변경.

참고 사이트

카카오톡 공유하기 디버깅 도구: https://developers.kakao.com/tool/debugger/sharing

페이스북 공유하기 디버깅 도구: Sharing Debugger - Meta for Developers

페이스북 크롤러 디버깅 방법(크롬 기준): 개발자 도구 → 네트워크 컨디션스 → facebookexternalhit/1.1 (+http://www.facebook.com/externalhit_uatext.php) 설정 시 페이스북 공유하기를 통해 리다이렉트되는 과정을 확인할 수 있음.
