---
layout: page
title: Feature Sliced Design
---

# 기능 분할 설계 - FSD(Feature Sliced Design)

프론트엔드 애플리케이션의 구조를 잡는 아키텍처 방법론

## Layer

1. App - 앱 실행을 위한 초기 설정 얘를 들면, 라우팅, 진입점, 전역 스타일, 프로바이더등이 이 폴더안에서 이루어진다.
2. Pages - 라우팅되는 페이지 컴포넌트로 구성된다.
3. Widgets - 독립적으로 작동하는 대규모 기능 또는 UI 컴포넌트, 보통 하나의 완전한 기능.
4. Features - 제품 전반에 걸쳐 재사용되는 기능 구현체로, 사용자에게 실질적인 비즈니스 가치를 제공하는 동작. (예를 들면 어떤 기능 - 로그인 구현 또는 유효성 검증)
5. Entities - 프로젝트가 다루는 비즈니스 엔티티, (예를 들어 user 또는 product)
6. Shared - 특정한 비즈니스 로직에 종속되지 않고 재사용 가능한 컴포넌트와 유틸리티로 구성 (선택사항)

레이어를 다룰 때의 중요한 점은 한 레이어의 구성 요소는 반드시 아래에 있는 레이어의 구성 요소만 알수있고 임포트할 수 있다. (pages에서는 shared를 가져와서 사용할 수 있지만 shared에서는 app에 접근할 수 없다.)

## slice

- 비즈니스 도메인별로 코드를 분할
- 높은 응집도와 낮은 결합도를 위해 같은 레이어 안에서 다른 슬라이스를 참조할 수 없다.

## segment

목적에 따라 코드를 그룹화

ui - UI와 관련된 모든 것: UI 컴포넌트, 날짜 포맷터, 스타일 등.<br/>
api - 백엔드 상호작용: request 함수, 데이터 타입, mapper 등.<br/>
model - 데이터 모델: 스키마, 인터페이스, 스토어, 비즈니스 로직.<br/>
lib - 슬라이스 안에 있는 다른 모듈이 필요로 하는 라이브러리 코드.<br/>
config - 프로젝트 설정 파일

## 예제

App과 Shared는 다른 레이어들과 달리 슬라이스를 가지지 않으며, 세그먼트로 구성.<br/>

- 📂 app (layer)
  - 📁 config (segment)
- 📂 pages
  - 📁 home (slice)
    - 📁 ui
    - 📁 api
  - 📁 settings
- 📂 shared
  - 📁 utils
  - 📁 lib

### 공개 api

slice와 segment에는 공개 api(index.js 또는 index.ts)를 통해 필요한 기능만 외부로 export 하는 방식.<br/>
app slice와 segment는 공개 api의 index 파일에 정의된 slice 기능과 컴포넌트만 사용하여 캡슐화시킨다.<br/>
공개 api에 정의되지 않은 slice 또는 segment의 내부 부분은 격리시키고, 외부와 공유될 필요가 있는 부분들은 index 파일에서만 export하여 불필요한 노출을 하지 않는 방식으로 캡슐화
