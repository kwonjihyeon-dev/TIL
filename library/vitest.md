# Vitest

Vite 기반의 빠르고 현대적인 단위 테스트 프레임워크입니다.<br/>Vite의 설정과 플러그인을 그대로 활용할 수 있어 별도의 복잡한 설정 없이 테스트 환경을 구축할 수 있습니다.

### 주요 특징

-   **빠른 실행 속도<br />**
    Vite의 ESM 기반 변환과 HMR(Hot Module Replacement)을 활용하여 테스트를 빠르게 실행하고, 파일 변경 시 관련된 테스트만 다시 실행합니다.
-   **Jest 호환 API**<br />
-   **TypeScript 및 JSX 지원**<br />
-   **Watch 모드**<br />
    파일 변경을 감지하여 자동으로 테스트를 재실행하며, 스마트한 필터링으로 변경된 부분과 관련된 테스트만 실행
-   **UI 모드**<br />
    브라우저에서 테스트 결과를 시각적으로 확인할 수 있는 UI 제공

### 기본 설정

```js
// package.json
{
  "scripts": {
    "test": "vitest",
    "test:ui": "vitest --ui",
    "test:run": "vitest run"
  }
}
```

```js
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
    test: {
        globals: true,
        environment: 'jsdom', // 'node', 'jsdom', 'happy-dom' 등
        setupFiles: './src/test/setup.ts', // 전역 테스트 환경 설정
    },
});
```

### Setup과 Teardown

테스트 실행 전후에 특정 작업을 자동으로 수행하는 기능으로 <u>테스트 환경을 준비하고 정리하는 용도</u>로 사용합니다.<br />
기본적으로는 <u>파일 단위로 격리</u>되어 동작하고, 전역 혹은 여러 파일에 공통 적용하려면 `vitest.config.ts`에 `setupFiles` 에 `setup.ts` 파일 경로를 지정해주면 됩니다.

-   전역 + 로컬 파일 모두 있을 경우,<br/>
    전역 beforeAll → 로컬 beforeAll → 테스트 실행 → 로컬 afterAll -> 전역 afterAll 순서로 실행

#### 주요 역할

-   테스트 독립성 보장: 각 테스트가 서로 영향을 주지 않도록 매번 깨끗한 상태에서 시작할 수 있습니다.
-   중복 코드 제거: 여러 테스트에서 반복되는 준비 작업을 한 곳에서 관리할 수 있습니다.
-   리소스 관리: 데이터베이스 연결, 파일 핸들 등 리소스를 적절히 생성하고 정리할 수 있습니다.
-   테스트 신뢰성 향상: 일관된 초기 상태를 보장하여 테스트 결과의 신뢰성을 높입니다.

#### 주요 함수

-   beforeAll: 모든 테스트가 시작되기 전에 딱 1번만 실행
-   afterAll: 모든 테스트가 끝난 후에 딱 1번만 실행
-   beforeEach: 각각의 테스트가 실행되기 전마다 매번 실행
-   afterEach: 각각의 테스트가 끝날 때마다 매번 실행

### Fixture

테스트를 실행하기 위해 필요한 고정된 상태나 데이터 환경을 의미합니다.<br/>테스트가 일관되고 예측 가능한 조건에서 실행될 수 있도록 사전에 준비된 데이터나 객체를 제공합니다.

#### 주요 역할

-   테스트 데이터 준비: 테스트에 필요한 초기 데이터나 객체를 미리 설정
-   일관성 보장: 모든 테스트가 동일한 초기 상태에서 시작하도록 보장
-   재사용성: 여러 테스트에서 동일한 설정을 반복 사용
-   격리성: 각 테스트가 독립적으로 실행되도록 환경 구성

#### 사용 예시

```js
import { beforeEach, describe, it, expect } from 'vitest';

describe('User Service', () => {
    // Fixture: 테스트용 사용자 데이터
    let testUser: User;

    beforeEach(() => {
        // 각 테스트 전에 fixture 초기화
        testUser = {
            id: 1,
            name: 'John Doe',
            email: 'john@example.com',
        };
    });

    it('should update user name', () => {
        // testUser fixture를 사용
        const result = updateUserName(testUser, 'Jane Doe');
        expect(result.name).toBe('Jane Doe');
    });
});
```
