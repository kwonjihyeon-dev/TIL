---
layout: post
title: 디자인 토큰 자동화
date: 2026-02-19
tags: [Figma, Figma Tokens Studio, Style Dictionary, GitHub Actions]
references:
    - title: 디자인 토큰 자동화 참고 사이트 - 올리브영 테크 블로그
      url: https://oliveyoung.tech/2024-12-16/Design-System-Token-Automation/
---

# 피그마에서 코드까지, 디자인 토큰 자동화 파이프라인 구축기

> Figma Tokens Studio + Style Dictionary + GitHub Actions를 활용한 Web 디자인 토큰 자동화

## 도입 배경

자사 서비스에서는 디자인 시스템의 토큰이 변경될 때마다 다음과 같은 플로우가 반복되었습니다.

-   디자이너가 피그마에서 토큰을 변경하면, 개발자가 **수동으로** 코드에 반영
-   CSS Variables, Tailwind 설정 등 **여러 파일을 일일이** 수정
-   수작업 과정에서 **오타, 누락, 불일치**가 발생할 가능성이 있음
-   변경 사항에 대한 이력 추적이 어렵고, **어떤 토큰이 왜 바뀌었는지** 파악이 힘듦

이러한 문제를 해결하기 위해, **Figma에서 토큰이 변경되면 Web 코드에 자동으로 반영되는 파이프라인**을 구축했습니다.

## 디자인 토큰 구조

현재 디자인 토큰은 **2단계 계층 구조**로 설계되어 있습니다.

### Atomic Tokens (기본값)

실제 색상 HEX 값을 가지는 기초 토큰입니다. 다른 토큰이 이 값을 참조합니다.

```json
{
    "Atomic": {
        "gray": {
            "50": { "value": "#f9f9f9", "type": "color" },
            "500": { "value": "#898990", "type": "color" },
            "900": { "value": "#0c0c0d", "type": "color" }
        },
        "green": {
            "500": { "value": "#1db177", "type": "color" }
        }
    }
}
```

### Semantic Tokens (의미 기반)

Atomic 토큰을 참조하여 **용도에 맞는 이름**을 부여한 토큰입니다.

```json
{
    "Brand": {
        "Primary": { "value": "{Atomic.green.500}", "type": "color" },
        "Secondary": { "value": "{Atomic.bluegray.600}", "type": "color" }
    },
    "Basic": {
        "Surface": {
            "1": { "value": "{Atomic.Common.white}", "type": "color" },
            "2": { "value": "{Atomic.gray.50}", "type": "color" }
        }
    }
}
```

이렇게 분리하면, Atomic 레벨에서 색상값 하나만 바꿔도 이를 참조하는 모든 Semantic 토큰에 자동으로 반영됩니다.

## 전체 자동화 흐름

```
┌──────────────┐
│   Figma      │  디자이너가 Tokens Studio에서 토큰 수정
│   (Design)   │
└──────┬───────┘
       │ Tokens Studio → Git Push
       ▼
┌──────────────────────────┐
│  design-tokens           │  tokens.json이 main 브랜치에 push
│  (Source of Truth)       │
└──────┬───────────────────┘
       │ GitHub Actions 트리거
       ▼
┌──────────────────────────────────────┐
│                                      │
│  ┌─────────────┐  ┌──────────────┐   │
│  │ Slack 알림   │  │ Web 빌드      │   │
│  │ (즉시 전송)   │  │ & PR 생성     │   │
│  └─────────────┘  └──────┬───────┘   │
│                          │            │
│                          ▼            │
│                   ┌────────────┐      │
│                   │ Web 프로젝트 │      │
│                   │   (PR)     │      │
│                   └────────────┘      │
│                                       │
└──────────────────────────────────────┘
```

## Step 1: Figma → Git 연동

Figma의 **Tokens Studio for Figma** 플러그인을 사용합니다. 디자이너가 토큰을 수정하고 "Push to Git"을 누르면, `tokens.json` 파일이 `design-tokens` 리포지토리의 `main` 브랜치에 직접 push됩니다.

이때 Tokens Studio가 생성하는 JSON은 다음과 같은 구조입니다.

```json
{
  "Color/Mode 1": {
    "Brand": { ... },
    "Basic": { ... },
    "Accent": { ... },
    "Atomic": { ... }
  },
  "$themes": [...],
  "$metadata": { ... }
}
```

현재는 Tokens Studio 무료 버전을 사용 중이라 JSON으로 파일이 생성되면 `"Color/Mode 1"`이라는 래퍼와 `$themes`, `$metadata` 메타데이터등 불필요한 부분도 일부 포함되어 있어, Style Dictionary가 바로 읽을 수 없습니다. 이를 해결하기 위해 **전처리 단계**를 추가했습니다.

## Step 2: 토큰 전처리

`scripts/preprocess-tokens.js`가 Style Dictionary 빌드 전에 실행됩니다.

```javascript
// "Color/Mode 1" 내용을 최상위로 이동
if (tokens['Color/Mode 1']) {
    const modeTokens = tokens['Color/Mode 1'];
    delete tokens['Color/Mode 1'];
    Object.assign(tokens, modeTokens);
}

// 메타데이터 제거
delete tokens.$themes;
delete tokens.$metadata;
```

**Before** (`tokens.json`):

```json
{ "Color/Mode 1": { "Brand": { "Primary": { "value": "{Atomic.green.500}" } } } }
```

**After** (`tokens.processed.json`):

```json
{ "Brand": { "Primary": { "value": "{Atomic.green.500}" } } }
```

이 전처리를 통해 Tokens Studio의 출력 형식과 Style Dictionary의 입력 형식 간 차이를 해소합니다.

## Step 3: Style Dictionary 빌드

Style Dictionary v5와 `@tokens-studio/sd-transforms`를 사용하여 `tokens.processed.json`을 Web 코드로 변환합니다.

### Web 플랫폼 설정 (`config/web.js`)

하나의 소스에서 **여러 포맷의 파일**을 동시에 생성합니다.(우리는 3가지 포맷이 필요했음)

```javascript
export default {
    source: ['tokens.processed.json'],
    preprocessors: ['tokens-studio'],
    platforms: {
        css: {
            transformGroup: 'css',
            buildPath: 'build/web/styles/',
            files: [{ destination: 'variables.css', format: 'css/variables' }],
        },
        ts: {
            transformGroup: 'js',
            buildPath: 'build/web/styles/',
            files: [{ destination: 'tokens.ts', format: 'javascript/es6' }],
        },
        tailwind: {
            transformGroup: 'tokens-studio',
            buildPath: 'build/web/tailwind/',
            files: [{ destination: 'colors.ts', format: 'custom/tailwind-colors' }],
        },
    },
};
```

### 생성 결과물

**1) CSS Variables** (`variables.css`)

```css
:root {
    --atomic-green-500: #1db177;
    --brand-primary: var(--atomic-green-500);
    --basic-surface-1: var(--atomic-common-white);
    --basic-on-surface-1: var(--atomic-gray-800);
}
```

**2) TypeScript 상수** (`tokens.ts`)

```typescript
export const BrandPrimary = '#1db177';
export const BrandSecondary = '#535c68';
export const BasicSurface1 = '#ffffff';
export const AccentGreen = '#1db177';
```

**3) Tailwind Colors** (`colors.ts`) — 커스텀 포맷

Tailwind CSS에서 바로 사용할 수 있도록, Atomic 토큰은 팔레트로, Semantic 토큰은 Atomic을 참조하는 구조로 생성합니다.

```typescript
export const themeColors = {
    gray: { 50: '#f9f9f9', 500: '#898990', 900: '#0c0c0d' },
    green: { 50: '#f3fbf8', 500: '#1db177', 900: '#0a402a' },
    // ...
} as const;

export const commonColors = {
    white: '#ffffff',
    black: '#000000',
} as const;

export const colorTheme = {
    brand: {
        primary: themeColors.green[500], // HEX가 아닌 참조!
        secondary: themeColors.bluegray[600],
    },
    accent: {
        green: themeColors.green[500],
        red: themeColors.red[500],
    },
    state: {
        success: themeColors.green[500],
        error: themeColors.red[500],
    },
    basic: {
        'surface-1': commonColors.white,
        'on-surface-1': themeColors.gray[800],
    },
} as const;
```

이 커스텀 포맷의 핵심은 **Semantic 토큰이 HEX 리터럴이 아닌 Atomic 토큰을 참조**한다는 점입니다. 이렇게 하면 Atomic 색상을 수정했을 때 Semantic 토큰도 자동으로 따라갑니다.

## Step 4: GitHub Actions 자동 배포

`tokens.json`이 `main` 브랜치에 push되면, **2개의 워크플로우가 동시에 실행**됩니다.

### 1) Slack 푸시 알림 (`figma-slack.yml`)

즉시 Slack에 "디자인 토큰이 push되었습니다" 알림을 보냅니다. 빌드가 완료되기 전에 팀에게 변경 사실을 알리는 역할입니다.

### 2) Web 빌드 & PR 생성 (`build-web.yml`)

```
tokens.json 변경 감지
  → npm run build:web (Style Dictionary)
  → git diff로 변경사항 추출
  → AWS Bedrock(Claude)로 PR 설명 자동 생성
  → Web 프로젝트 리포에 빌드 결과 복사
  → dev 브랜치 대상 PR 자동 생성
  → Slack 성공 알림
```

핵심 포인트:

-   **Cross-Repository PR**: `peter-evans/create-pull-request` 액션과 PAT(Personal Access Token)을 사용하여, 별도의 Web 프로젝트 리포지토리에 자동으로 PR을 생성합니다. 기본 `GITHUB_TOKEN`은 현재 레포 권한만 있기 때문에, `repo` 스코프가 포함된 PAT가 필요합니다.

-   **빌드 실패 시 알림**: 빌드가 실패하면 에러 로그를 포함한 Slack 알림을 즉시 전송하고, Actions 로그 링크를 함께 제공합니다.

## Step 5: 개발자 사용

PR이 Web 프로젝트의 `dev` 브랜치에 생성되면, 리뷰 후 머지만 하면 됩니다. 별도의 수동 작업이 필요 없습니다.

```typescript
// Tailwind 설정에서 바로 사용
import { colorTheme, themeColors } from '@/styles/colors';

// CSS Variables로도 사용 가능
// var(--brand-primary)
// var(--basic-surface-1)
```

## 알림 체계

모든 단계에서 Slack으로 상태를 알려줍니다.

| 시점           | 메시지                  | 포함 정보               |
| -------------- | ----------------------- | ----------------------- |
| 토큰 push 직후 | 📦 Design Tokens Pushed | 커밋 메시지, 작성자     |
| Web 빌드 성공  | ✅ Web Tokens Synced    | PR 링크                 |
| 빌드 실패      | ❌ Build Failed         | 에러 로그, Actions 링크 |

## 프로젝트 구조

```
design-tokens/
├── tokens.json                      # Figma → Tokens Studio가 push하는 소스 파일
├── tokens.processed.json            # 전처리된 중간 파일
├── package.json                     # Style Dictionary 의존성 및 빌드 스크립트
│
├── scripts/
│   └── preprocess-tokens.js         # tokens.json 전처리 (래퍼 제거)
│
├── config/
│   └── web.js                       # Web 빌드 설정 (CSS + TS + Tailwind)
│
├── build/web/
│   ├── styles/
│   │   ├── variables.css            # CSS Custom Properties
│   │   └── tokens.ts               # ES6 상수 export
│   └── tailwind/
│       └── colors.ts               # Tailwind용 컬러 객체
│
└── .github/workflows/
    ├── figma-slack.yml              # 토큰 push 즉시 Slack 알림
    └── build-web.yml            # Web: 빌드 → PR 생성 → Slack
```

## 도입 효과

### Before

1. 디자이너가 피그마에서 디자인 토큰 변경
2. 디자이너가 개발자에게 변경 내용 전달 (Slack/문서)
3. Web 개발자가 CSS, Tailwind 설정 수동 수정
4. 코드 리뷰 → 배포

### After

1. 디자이너가 피그마에서 디자인 토큰 변경 → Tokens Studio에서 Push
2. 자동으로 Web PR 생성
3. 개발자는 PR 리뷰 후 머지만

## 마무리

예제는 **색상(Color) 토큰**이지만 이런 식으로 Typography, Spacing, Border Radius 등 여러 토큰을 자동화시킬 수 있습니다.

디자인 토큰 자동화는 "디자이너와 개발자 사이의 수작업을 제거"하는 것이 핵심입니다.

각 팀의 상황에 맞게 자동화 범위를 조절하면 충분히 효과를 볼 수 있습니다.
