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

JSON에는 토큰 외에도 set 래퍼(`Color/Mode 1`, `Typography`)와 `$themes` / `$metadata` 같은 부가 정보가 함께 들어옵니다. Style Dictionary 가 그대로 읽으면 토큰 path 가 `Color/Mode 1.Atomic.green.500` 처럼 한 단계 더 깊어져 참조가 깨지므로, **전처리 단계**를 먼저 두어 정리합니다.

## Step 2: 토큰 전처리

`scripts/preprocess-tokens.js`가 Style Dictionary 빌드 전에 실행됩니다.

### 왜 전처리가 필요한가

Tokens Studio는 토큰을 **token set 단위**로 묶어서 내보냅니다. 디자이너가 정의한 모든 토큰은 어떤 set 안에 들어가야 하고, 같은 set 안에서는 root 경로로 서로를 참조합니다 — 예를 들어 `{Atomic.green.500}`은 "현재 활성화된 set들의 root에서 `Atomic.green.500`을 찾으라"는 뜻입니다.

> 참고로 set 이름이 `Color/Mode 1` 처럼 슬래시(`/`)를 포함하는 경우가 있는데, Tokens Studio에서 슬래시는 Sets 패널의 **폴더 계층 표시용**이라고 합니다. 정확히는 모르지만, 어떤 식으로 명명되었든 결과적으로 set 래퍼가 토큰들 위에 한 겹 더 씌워진 형태가 되어, Style Dictionary 가 토큰 참조(`{Atomic.green.500}` 같은)를 그대로는 해석하지 못합니다. 그래서 자동화 전에 **전처리로 불필요한 부분을 제거하는 과정**을 먼저 진행했습니다.

### 메타데이터로 자동 평탄화

Tokens Studio는 set 목록을 `$metadata.tokenSetOrder` 에 친절하게 적어주기 때문에, 이 배열을 그대로 읽어서 처리하면 됩니다. 디자이너가 새 set(Spacing, Shadow 등)을 추가해도 이 스크립트는 수정할 필요가 없습니다.

```javascript
// $metadata.tokenSetOrder 의 순서대로 모든 set 을 root 로 평탄화
const tokenSetOrder = tokens.$metadata?.tokenSetOrder ?? [];

tokenSetOrder.forEach((setName) => {
    if (tokens[setName]) {
        // 주의: delete 를 먼저 해야 함.
        //  set 내부에 set 이름과 같은 하위 그룹이 있으면(예: Typography set 안에
        //  composite Typography 그룹이 있는 경우) Object.assign 후 delete 하면
        //  그 하위 그룹까지 같이 지워집니다.
        const setContents = tokens[setName];
        delete tokens[setName];
        Object.assign(tokens, setContents);
    }
});

// Tokens Studio가 추가한 메타데이터 제거
delete tokens.$themes;
delete tokens.$metadata;
```

**Before** (`tokens.json`):

```json
{
  "Color/Mode 1": {
    "Brand": { "Primary": { "value": "{Atomic.green.500}" } }
  },
  "Typography": {
    "fontWeight": { "fontWeightBold": { "value": "Bold" } },
    "Typography": {
      "Tbd40": {
        "value": {
          "fontWeight": "{fontWeight.fontWeightBold}",
          "fontSize": "{fontSize.fontSize-40}"
        }
      }
    }
  },
  "$metadata": {
    "tokenSetOrder": ["Color/Mode 1", "Typography"]
  }
}
```

**After** (`tokens.processed.json`):

```json
{
  "Brand": { "Primary": { "value": "{Atomic.green.500}" } },
  "fontWeight": { "fontWeightBold": { "value": "Bold" } },
  "Typography": {
    "Tbd40": {
      "value": {
        "fontWeight": "{fontWeight.fontWeightBold}",
        "fontSize": "{fontSize.fontSize-40}"
      }
    }
  }
}
```

### 적용 시 유의점

- **평탄화가 누락되면** 빌드 시 `tries to reference {fontWeight.fontWeightBold}, which is not defined` 같은 에러가 납니다. 즉 참조가 깨졌다는 신호.
- **메타데이터(`$themes`, `$metadata`) 제거**는 Style Dictionary가 이를 토큰으로 오해하는 것을 방지하기 위함입니다.
- **다크모드 도입 시점에는 추가 로직이 필요**합니다. 같은 키(예: `Atomic.gray.500`)를 가진 set 이 둘 이상이 되면(`Color/Mode 1`, `Color/Mode 2`) 단순 평탄화는 뒤 set 이 앞 set 을 덮어쓰니까요. mode 별로 빌드를 분기하거나, `$themes` 배열을 읽어 활성 mode 의 set 만 평탄화하는 식으로 풀면 됩니다.

## Step 3: Style Dictionary 빌드

Style Dictionary v5와 `@tokens-studio/sd-transforms`를 사용하여 `tokens.processed.json`을 Web 코드로 변환합니다.

### Web 플랫폼 설정 (`config/web.js`)

하나의 토큰 소스에서 **여러 포맷의 파일**을 동시에 생성합니다. 우리는 다음 4가지가 필요했습니다.

| Platform | 출력 | 사용처 |
|---|---|---|
| `css` | `variables.css` | CSS 변수로 직접 사용 |
| `ts` | `tokens.ts` | TS 상수 import |
| `tailwind` | `colors.ts`, `fonts.ts`, `fontSize.ts` 등 | Tailwind config 주입 |
| `scss` | `color.scss`, `typography.scss` | SCSS 기반 프로젝트용 |

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
            files: [
                { destination: 'colors.ts', format: 'custom/tailwind-colors' },
                { destination: 'fonts.ts',  format: 'custom/tailwind-fonts' },
                // fontSize.ts / lineHeight.ts / fontWeight.ts / letterSpacing.ts
            ],
        },
        scss: {
            transformGroup: 'tokens-studio',
            buildPath: 'build/web/scss/',
            files: [
                { destination: 'color.scss',      format: 'custom/scss-colors' },
                { destination: 'typography.scss', format: 'custom/scss-typography' },
            ],
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

### Typography 토큰 — 두 계층 다루기

Typography 는 Color 보다 한 단계 복잡합니다. **Atomic 토큰**(개별 fontSize / lineHeight / fontWeight / letterSpacing)과, 이 atomic 들을 조합한 **Composite 토큰**(텍스트 스타일 preset, 예: `Tbd40` = 40px / 52px / Bold)이 두 계층으로 존재합니다.

원본 토큰은 대략 이렇게 생겼습니다.

```json
{
  "fontSize":   { "fontSize-40":  { "value": "40", "type": "fontSizes" } },
  "lineHeight": { "lineHeight-52": { "value": "52", "type": "lineHeights" } },
  "fontWeight": { "fontWeightBold": { "value": "Bold", "type": "fontWeights" } },
  "Typography": {
    "Tbd40": {
      "value": {
        "fontWeight": "{fontWeight.fontWeightBold}",
        "fontSize":   "{fontSize.fontSize-40}",
        "lineHeight": "{lineHeight.lineHeight-52}"
      }
    }
  }
}
```

이를 두 갈래의 산출물로 풀어줍니다.

**1) Composite 토큰 → 클래스 묶음** (`fonts.ts`, `typography.scss`)

디자이너가 정의한 텍스트 스타일을 **클래스 한 방으로 적용**할 수 있게 하는 형태입니다. preset 의 일관성을 강제하고 싶을 때 적합합니다.

```typescript
// fonts.ts (Tailwind addComponents 용)
export const typographyTheme = {
  '.tbd-40': {
    fontSize: '40px',
    lineHeight: '52px',
    fontWeight: '700',
    letterSpacing: '-0.3px',
  },
  '.tmd-22': { fontSize: '22px', lineHeight: '32px', fontWeight: '500', /* ... */ },
  // ...
} as CSSRuleObject;
```

```scss
/* typography.scss (SCSS 기반 프로젝트용) */
.tbd-40 {
  font-size: 40px;
  line-height: 52px;
  font-weight: 700;
  letter-spacing: -0.3px;
}
```

**2) Atomic 토큰 → 개별 export** (`fontSize.ts`, `lineHeight.ts`, `fontWeight.ts`, `letterSpacing.ts`)

Tailwind 의 `theme.extend.fontSize` 등에 그대로 주입해 **유틸리티 조합**으로도 쓸 수 있도록 별도 export 합니다.

```typescript
// fontSize.ts
export const fontSize = { '10': '10px', '12': '12px', /* ... */ '40': '40px' } as const;
```

#### 변환 시 알아둘 두 가지

- **클래스명 변환**: 토큰명 `Tbd40` → 클래스 `.tbd-40` (prefix 다음 하이픈 + lowercase). 한 번에 weight 와 size 를 식별할 수 있는 약어 컨벤션을 클래스명에 그대로 녹여 두면, 디자인 시스템의 명명 규칙이 코드에서도 그대로 보입니다.

- **fontWeight 매핑**: Tokens Studio 에서 fontWeight 값은 문자열(`"Bold"`, `"Medium"`, `"Regular"`)로 저장됩니다. CSS 는 숫자(`700`, `500`, `400`)를 요구하므로 빌드 시점에 매핑이 필요합니다.

  ```javascript
  const FONT_WEIGHT_MAP = { Bold: '700', Medium: '500', Regular: '400' };
  ```

  > 디자이너가 토큰 값에 직접 숫자(`"700"`)를 정의해두면 매핑 코드를 제거할 수 있습니다.

### 도메인이 늘어나면 — Config 모듈화

토큰 도메인이 늘어나면(Color → Typography → Spacing → Shadow…) 단일 `config/web.js` 가 빠르게 비대해집니다. 한 파일에 카테고리 분류·렌더링·포맷 등록·SD 설정이 다 들어가서 수정 한 번에 책임이 여럿 얽히게 되죠. **도메인별 모듈로 쪼개고, 진입점은 등록 호출만 담당**하도록 분리하면 깔끔합니다.

```
config/
└── web/
    ├── index.js          # SD config + 포맷 등록 호출만
    ├── colors.js         # color 도메인 (수집 + 렌더 + 포맷 등록)
    ├── typography.js     # typography 도메인
    └── shared.js         # 공통 유틸 (헤더 코멘트 등)
```

```javascript
// config/web/index.js
import StyleDictionary from 'style-dictionary';
import { register } from '@tokens-studio/sd-transforms';
import { registerColorFormats } from './colors.js';
import { registerTypographyFormats, excludeTypography } from './typography.js';

register(StyleDictionary);
registerColorFormats(StyleDictionary);
registerTypographyFormats(StyleDictionary);

export default { /* source, preprocessors, platforms */ };
```

새 도메인을 추가할 때는 모듈 하나를 만들고 진입점에 한 줄 추가하면 됩니다 — 진입점은 도메인 로직을 알 필요가 없습니다.

## Step 4: GitHub Actions 자동 배포

`tokens.json` 이 `main` 브랜치에 push 되면 **2개의 워크플로우가 동시에 실행**됩니다.

### 1) Slack 푸시 알림 (`figma-slack.yml`)

즉시 Slack 에 "디자인 토큰이 push 되었습니다" 알림을 보냅니다. 빌드 결과를 기다리지 않고 팀에게 변경 사실부터 알리는 역할입니다.

### 2) Web 빌드 & PR 생성 (`build-web.yml`)

```
tokens.json 변경 감지
  → npm run build:web (Style Dictionary)
  → git diff 로 변경사항 추출
  → AWS Bedrock(Claude) 로 PR 설명 자동 생성(선택사항)
  → 대상 프로젝트 레포에 빌드 결과 복사
  → 통합 브랜치 대상 PR 자동 생성
  → Slack 성공 알림
```

**트리거 조건** — push 외에도 수동 실행 옵션을 함께 두는 게 좋습니다.

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'tokens.json'
  workflow_dispatch:
```

`workflow_dispatch` 를 추가해 두면 GitHub Actions 탭의 `Run workflow` 버튼으로 언제든 같은 흐름을 다시 돌릴 수 있어, 워크플로우 변경 후 검증할 때 편합니다.

### 핵심 포인트

- **Cross-Repository PR**: `peter-evans/create-pull-request` 액션과 PAT(Personal Access Token)을 사용해, 별도의 Web 프로젝트 리포지토리에 자동으로 PR 을 생성합니다. 기본 `GITHUB_TOKEN` 은 현재 레포 권한만 있기 때문에, `repo` 스코프가 포함된 PAT 가 필요합니다.

- **여러 프로젝트로 분기 배포**: 같은 토큰이 서로 다른 코드베이스로 가야 할 때(예: Tailwind + TS 위주의 프로젝트 / SCSS 위주의 레거시 프로젝트), 한 워크플로우 안에서 **두 프로젝트에 각각 PR 을 생성**할 수 있습니다. checkout / 복사 / `peter-evans/create-pull-request` 단계를 프로젝트 수만큼 반복하면 되고, 각 프로젝트의 통합 브랜치(`dev`, `Design-tokens` 등)를 base 로 지정합니다.

- **머지 방식 복사 (`cp -r`)**: 빌드 산출물을 대상 프로젝트에 복사할 때 디렉토리를 통째로 옮기되, **동일 경로의 직접 만든 파일은 보존되고 산출물과 같은 이름의 파일만 덮어쓰기** 됩니다. `cp -r build/web/scss/. ./checkout/path/scss/` 같은 형태로 두면, 우리가 만든 파일과 충돌하지 않는 기존 파일들은 그대로 유지됩니다.

- **빌드 실패 시 알림**: 빌드가 실패하면 에러 로그를 포함한 Slack 알림을 즉시 전송하고, Actions 로그 링크를 함께 제공합니다.

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
├── tokens.json                      # Figma → Tokens Studio 가 push 하는 소스 파일
├── tokens.processed.json            # 전처리된 중간 파일
├── package.json                     # Style Dictionary 의존성 및 빌드 스크립트
│
├── scripts/
│   └── preprocess-tokens.js         # set 래퍼 평탄화 + 메타데이터 제거
│
├── config/
│   └── web/                         # 도메인별 모듈 분리 (Step 3 참고)
│       ├── index.js                 # SD config + 포맷 등록 호출만
│       ├── colors.js                # color 도메인
│       ├── typography.js            # typography 도메인
│       └── shared.js                # 공통 유틸 (헤더 코멘트 등)
│
├── build/web/
│   ├── styles/
│   │   ├── variables.css            # CSS Custom Properties
│   │   └── tokens.ts                # ES6 상수 export
│   ├── tailwind/
│   │   ├── colors.ts                # Tailwind 용 컬러 객체
│   │   ├── fonts.ts                 # Typography 클래스 묶음
│   │   ├── fontSize.ts              # atomic typography 토큰
│   │   ├── lineHeight.ts
│   │   ├── fontWeight.ts
│   │   └── letterSpacing.ts
│   └── scss/
│       ├── color.scss               # SCSS color 맵 + 유틸 클래스
│       └── typography.scss          # SCSS typography 클래스
│
└── .github/workflows/
    ├── figma-slack.yml              # 토큰 push 즉시 Slack 알림
    └── build-web.yml                # Web: 빌드 → PR 생성 → Slack
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

여기서는 **Color 와 Typography 토큰**을 중심으로 다뤘는데, Spacing, Border Radius, Shadow, Motion 등 다른 토큰 도메인도 같은 패턴으로 확장할 수 있습니다. 도메인별로 모듈을 추가하고 진입점에 등록 한 줄을 더하면 됩니다.

디자인 토큰 자동화의 핵심은 **"디자이너와 개발자 사이의 수작업을 제거"** 하는 것입니다. 처음부터 모든 도메인을 자동화할 필요는 없고, **가장 손이 많이 가는 영역부터 시작해서 점진적으로 확장**하면 됩니다.
