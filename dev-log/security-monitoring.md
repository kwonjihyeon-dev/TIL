---
title: GitHub Dependabot 도입기 - 라이브러리 보안 취약점, 자동으로 잡아내기
date: 2026-03-18
tags: [GitHub, Dependabot, Security, npm audit, GitHub Actions, CI/CD]
---

# GitHub Dependabot 도입기: 라이브러리 보안 취약점, 자동으로 잡아내기

## 왜 보안 취약점 모니터링을 고민하게 되었을까

우리 서비스는 수많은 오픈소스 라이브러리 위에서 돌아가고 있다. React, Next.js, TanStack Query 등 프론트엔드만 해도 의존성이 수백 개에 달한다. 물론 발견되는 취약점 모두가 당장 대응해야 하는 건 아니다. Critical이 아닌 이상 즉각 조치가 필수는 아니지만, **Critical이 터졌을 때 즉각 조치할 수 있느냐**가 문제다.

어제(3/17) [Next.js Security 페이지](https://github.com/vercel/next.js/security)에 보안 취약점 관련 글이 올라왔는데, 이걸 알게 된 건 **직접 해당 사이트를 구독해두었기 때문**이었다. 구독하지 않았다면 한참 뒤에야 알았을 것이다. 그런데 우리가 사용하는 라이브러리가 Next.js 하나뿐인가? React, TanStack Query, Jotai, 수많은 유틸리티 라이브러리까지 — **모든 라이브러리의 보안 페이지를 일일이 구독**해둘 수는 없다.

그동안 우리 팀은 이런 상황이었다.

- 취약점이 발견되어도 **수동으로 확인**하고 대응하다 보니, 발견 자체가 늦어지는 경우가 잦았다.
- 누군가가 직접 구독하거나 뉴스에서 보지 않는 이상, **취약점 존재 자체를 모르고 지나가는 경우**가 많았다.
- 팀 간 **공유가 되더라도** "이거 우리 서비스도 해당되나요?" 혹은 **"대수롭지 않게"** 넘어가는 경우도 있었다.
- 위험도가 높은 취약점이 나왔을 때, **어떤 서비스의 어떤 버전이 영향을 받는지** 빠르게 파악하기 어려웠다.

사람이 직접 챙기는 방식으로는 즉각 대응에 한계가 분명했다. 현재는 Datadog POC를 진행 중이라 Code Security 기능으로 취약점 확인이 가능하긴 하지만, **비용이 들어서 유지가 불투명하다는 문제**가 있다. 이후에도 계속 적용할 수 있을지 미지수인 상황이라, **별도의 대응 체계**가 필요했다.

결국 우리에게 필요한 건 명확했다. 비용 부담 없이, **취약점을 자동으로 탐지하고, 우선순위를 매기고, 알림을 보내고, 조치를 추적할 수 있는 체계**를 만드는 것.

---

## 도입 후 우리가 기대한 것

Datadog에 의존하지 않고도 동작하는 보안 모니터링 체계를 만들면서, 세 가지를 기대했다.

### 1. 탐지 자동화 — 구독 없이도 알 수 있게

지금처럼 라이브러리 보안 페이지를 직접 구독하거나, 뉴스를 챙겨보지 않아도 **취약점이 발견되면 자동으로 알림이 오는 구조**. 사람이 챙기는 게 아니라 시스템이 챙겨주는 것이 핵심이다.

### 2. 우선순위 기반 대응 — Critical부터 먼저

모든 취약점에 같은 무게를 둘 수 없다. 심각도에 따라 **즉시 대응해야 할 것과 나중에 봐도 되는 것을 구분**하고, Critical/High는 즉시 알림, 나머지는 요약으로 받아보는 체계가 필요했다.

### 3. 개발 워크플로우 내재화 — 별도 도구 없이 PR에서 해결

별도의 보안 대시보드에 접속하는 게 아니라, **GitHub PR과 이메일 알림**으로 자연스럽게 연동한다. 개발자가 이미 일하고 있는 곳에서 바로 확인하고 대응할 수 있으니, 추가 비용 없이 기존 워크플로우에 녹아든다.

---

## GitHub Dependabot 적용기 — 개인 레포에서 먼저 검증하기

팀 레포에 바로 적용하기 전에, **개인 레포에서 먼저 동작을 확인**해보기로 했다. 추가 비용 없이 GitHub에 내장된 기능이라 설정 자체는 간단하지만, 실제로 알림이 어떻게 오는지, 어떤 형태로 PR이 생성되는지 직접 경험해보는 게 중요했다.

### Dependabot이 해주는 일

Dependabot은 세 가지 기능을 제공한다.

| 기능                            | 설명                                                                                                              |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| **Dependabot Alerts**           | `package.json` / `package-lock.json`의 의존성을 GitHub Advisory Database와 대조하여 알려진 CVE가 있으면 알림 발송 |
| **Dependabot Security Updates** | 취약한 의존성을 패치된 버전으로 올리는 PR을 **자동 생성**                                                         |
| **Dependabot Version Updates**  | 보안과 무관하게 최신 버전으로 업데이트하는 PR 생성                                                                |

핵심은 **Alerts**와 **Security Updates**의 조합이다. 취약점이 발견되면 메일 알림이 오고, 동시에 패치 버전으로 올리는 **PR까지 Dependabot이 자동 생성**해준다. 개발자는 리뷰하고 머지만 하면 된다.

여기에 추가로, **GitHub Actions로 dev 푸시 시점에 `npm audit`을 돌려서 Issue를 자동 생성**하는 스크립트를 직접 구성했다. Dependabot의 PR 생성과는 별개로, 푸시 시점에 즉각 확인하는 용도다.

### PR과 Issue, 뭐가 다른가?

Dependabot이 만들어주는 **PR**과 GitHub Actions로 생성하는 **Issue**는 역할이 다르다.

**Dependabot Security Updates → PR 생성**

- 취약점이 발견되면 **패치된 버전으로 `package.json`을 수정한 PR**을 자동으로 만들어준다.
- 예를 들어 `next@14.1.0`에 CVE가 발견되면 → `next@14.1.1`로 올리는 코드 변경 PR이 생성된다.
- PR에는 **CVE 번호, 체인지로그, 릴리즈 노트**까지 첨부되어 있어서, 이걸 참고해 대응 방식을 정할 수 있다.
  - 단순 patch 업데이트 → 그대로 머지
  - breaking change가 있는 major 업데이트 → PR 내용 참고해서 영향 범위 판단 후 직접 수정
  - 당장 적용이 어려운 경우 → Issue로 트래킹하면서 일정 잡기

**GitHub Actions 스크립트 → Issue 생성**

- dev에 푸시될 때 `npm audit`을 돌려서 취약점이 있으면 **"취약점이 있다"는 알림용 Issue**를 생성한다.
- 코드를 수정해주지는 않는다 — **"확인해라"라는 알림 역할**이다.
- Dependabot이 커버하지 못하는 타이밍(dev 푸시 시점)에 즉각 인지하기 위한 보조 장치다.

정리하면 **PR은 "해결책까지 제시"**, **Issue는 "문제 발생 알림"**이다.

| 역할                    | 누가                                          | 결과물                 | 성격        |
| ----------------------- | --------------------------------------------- | ---------------------- | ----------- |
| 취약점 탐지 + 메일 알림 | **Dependabot Alerts** (GitHub 내장)           | 메일, 깃헙 등으로 알림 | 알림        |
| 패치 버전 PR 자동 생성  | **Dependabot Security Updates** (GitHub 내장) | PR (코드 수정 포함)    | 해결책 제시 |
| dev 푸시 시 즉시 감사   | **GitHub Actions 스크립트** (직접 구성)       | Issue (알림용)         | 문제 인지   |

### 설정 방법

#### Step 1: Dependabot Alerts + Security Updates 활성화

1. GitHub 리포지토리의 **Settings > Advanced Security**에서 Dependabot alerts와 Dependabot security updates를 Enable 설정
2. 알림을 받고싶은 GitHub 리포지토리를 watch한다. (**Watch > Custom > Security alerts**를 체크)<br/>
   개인 레포라면 admin이라 기본 활성화되어 있을 수 있지만, 명시적으로 확인해두는 게 확실하다.

이것만으로 취약점 탐지, 메일 알림, 그리고 패치 PR 자동 생성이 시작된다.

#### Step 2: dev 푸시 시 자동 보안 체크 (GitHub Actions)

**dev 브랜치에 코드가 푸시될 때마다 즉시 보안 체크**를 실행한다.

```yaml
# 예시
# .github/workflows/{스크립트 파일명}.yml
name: Security Audit

on:
  push:
    branches: [dev]

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run security audit
        run: npm audit --audit-level=high
        continue-on-error: true

      - name: Create issue on vulnerability found
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: '🚨 보안 취약점 발견 - npm audit',
              body: '`dev` 브랜치에서 High 이상의 보안 취약점이 발견되었습니다.\n\n커밋: ' + context.sha + '\n\n`npm audit`을 실행하여 상세 내용을 확인해주세요.',
              labels: ['security']
            })
```

dev에 코드가 푸시되는 순간 `npm audit`이 돌고, High 이상의 취약점이 발견되면 **GitHub Issue가 자동 생성**된다. Dependabot PR이 "해결책"이라면, 이 Issue는 "지금 바로 확인해라"라는 **즉시 알림** 역할이다.

#### Step 3: 메일 알림 연결

Dependabot Alerts가 활성화되면, GitHub는 기본적으로 **리포지토리 관리자에게 이메일 알림**을 발송한다.

개인 레포에서 확인할 설정:

1. GitHub > **Settings > Notifications**에서 "Dependabot alerts" 항목의 이메일 알림을 활성화한다.
2. 알림 빈도를 설정한다 — 매번 즉시 알림, 일간 요약, 주간 요약 중 선택 가능하다.

### 참고: dependabot.yml로 더 세밀하게 제어할 수도 있다

위 Step 1~3만으로 기본적인 보안 모니터링은 동작하지만, Version Updates, PR 생성 규칙 등 여러 항목을 커스텀하게 제어하고 싶다면 리포지토리 루트에 `.github/dependabot.yml` 파일을 추가하면 된다.

```yaml
version: 2
updates:
  - package-ecosystem: 'npm'
    directory: '/'
    schedule:
      interval: 'weekly' # Version Updates 체크 주기 (Security Updates는 즉시 동작)
    open-pull-requests-limit: 10 # 동시 PR 최대 개수 (취약점 1개당 1PR → PR 폭탄 방지)
    target-branch: 'dev' # PR 대상 브랜치 지정
    allow:
      - dependency-type: 'production' # devDependencies 제외, 프로덕션만
    ignore:
      - dependency-name: 'next' # 특정 패키지 제외
        versions: ['>=15.0.0'] # 특정 버전 범위 제외
    labels:
      - 'security' # PR에 자동 라벨 부여
    reviewers:
      - 'jihyeon' # GitHub 유저네임으로 리뷰어 자동 지정
    assignees:
      - 'jihyeon' # PR 담당자 자동 지정
```

| 설정                       | 설명                                                 |
| -------------------------- | ---------------------------------------------------- |
| `schedule.interval`        | Version Updates 체크 주기 (daily / weekly / monthly) |
| `allow`                    | production 의존성만, 또는 특정 패키지만 PR 생성      |
| `ignore`                   | 특정 패키지나 버전 범위는 PR 생성 제외               |
| `open-pull-requests-limit` | PR이 한꺼번에 쏟아지는 것 방지                       |
| `target-branch`            | PR 대상 브랜치 지정 (기본은 default branch)          |
| `reviewers` / `assignees`  | PR 생성 시 리뷰어/담당자 자동 지정                   |

### 확인한 것

개인 레포에 위 설정을 적용한 뒤, 실제로 확인하고 싶었던 것들:

- Dependabot Alerts가 **메일로 제대로 오는지**
- 취약점 발견 시 **Security Updates PR이 자동 생성되는지**
- GitHub Actions의 `npm audit`이 **dev 푸시 시 정상 동작하는지**
- Issue 자동 생성이 **취약점 내용을 잘 담고 있는지**

---

## 알아두어야 할 한계

Dependabot의 기능에 대해 명확히 인지하고 있어야 할 점들이 있다.

- **소스 코드 보안 취약점(XSS, SQL injection 등)은 분석하지 못한다.** 이 영역은 CodeQL(GitHub Advanced Security)이 담당한다.
- **npm 생태계만 커버한다.** Docker 이미지 내부의 라이브러리 등은 별도 설정이 필요하다.
- **알려지지 않은 취약점(0-day)은 탐지할 수 없다.** Advisory Database에 등록된 것만 감지한다.

---

## 앞으로,

이번 글에서는 개인 레포에서 Dependabot + GitHub Actions 보안 체크를 설정하고, 메일 알림이 정상적으로 동작하는지까지 확인했다.

다음 글에서는 이걸 **실제 팀 레포에 적용한 과정**들을 정리할 예정이다:

- **Slack 알림 연동**: 취약점 발견 시 팀 Slack 채널로 즉시 알림 발송
- **알림 채널 및 승인 규칙 합의**: 어떤 심각도부터 즉시 알림을 보낼지, PR 자동 머지 범위를 어디까지 허용할지
- **브랜치 보호 규칙과의 연계**: Dependabot PR에 CI 필수 통과, 코드 리뷰 정책 적용
- **Datadog POC 결과에 따른 역할 분담**: 비용 판단 후 Dependabot 단독 운영 vs Datadog 병행 구조 결정
