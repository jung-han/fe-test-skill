---
name: write-test
description: >
  React 프로젝트의 테스트를 작성한다. 컨텍스트를 분석해 적합한 테스트 유형을 선택하고,
  모범사례에 맞춰 테스트 코드를 작성한다. "테스트 작성", "테스트 써줘", "/write-test" 등으로 호출된다.
---

## 대상 프로젝트 확인

`package.json`에 `react` 의존성이 없으면 이 스킬을 실행하지 않는다.

---

## 실행 절차

### Step 1. 프로젝트 설정 파악

프로젝트 루트에서 `project-config.md`를 찾는다.

있으면: 파일을 읽어 테스트 환경 설정을 파악한다.

없으면: 아래 파일들을 직접 읽어 설정을 파악하고, `@.claude/templates/project-config.md`를 기준으로 프로젝트 루트에 `project-config.md`를 생성해 채운다.

#### setupFiles 파일 자체를 읽는다

설정 파일에서 `setupFiles` 경로를 찾았다면, **그 파일의 내용도 반드시 읽는다.** 전역 hook(`beforeAll` / `beforeEach` / `afterEach` / `afterAll`)에서 무엇이 이미 처리되는지 파악해야 테스트 본문에서 중복 setup·teardown을 작성하지 않을 수 있다. 적용 규칙은 `@.claude/rules/writing-rules.md`의 "setupTests를 먼저 읽고 작성한다" 섹션 참조.

#### setupTests가 없거나 미흡한 경우

`setupFiles` 자체가 등록되어 있지 않거나, 파일이 비어 있거나, 단위/통합 테스트에 필요한 최소 구성(jest-dom import, MSW server 등록, mock 정리, fake timers, 시스템 시간 고정)이 빠져 있으면 **테스트 작성을 멈추고** 사용자에게 권장 구조를 안내한다.

권장 구조: `@.claude/templates/setup-tests.ts` (MSW v2 + jest-dom + 모든 테스트 fake timers + `vi.setSystemTime`로 시간 고정 + clear/reset 패턴 포함).

사용자가 채택하면 해당 템플릿을 setupFiles 경로(보통 `src/setupTests.ts`)에 그대로 복사하고, `vitest.config.*`에 등록되어 있는지 확인한 뒤 Step 2로 진행한다. 프로젝트 사정에 맞게 일부 hook을 빼거나 시각을 바꿀 수 있지만, **변경 사항은 사용자와 합의 후** 적용한다.

확인된 라이브러리 버전이 references/의 예시와 다를 수 있다. 아래 버전 분기를 확인해 코드 예시를 조정한다:

| 라이브러리                    | 확인 포인트                                                                       |
| ----------------------------- | --------------------------------------------------------------------------------- |
| `msw`                         | v1: `rest.get`, `ctx.json()` / v2: `http.get`, `HttpResponse.json()`              |
| `@testing-library/user-event` | v13: `userEvent.click()` 동기 / v14: `await user.click()` 비동기 + `setup()` 필수 |
| `@testing-library/react`      | v13 이하: `render` import 방식 다를 수 있음                                       |

context7 MCP가 사용 가능하면 확인된 버전의 공식 문서를 조회해 API를 정확히 참조한다.

---

### Step 2. 테스트 대상 파악

파일이 명시된 경우: 해당 파일을 읽는다.

파일이 명시되지 않은 경우: 최근 수정된 파일을 탐색하거나, 대화 맥락에서 대상을 파악한다. 판단이 어려우면 사용자에게 확인한다.

대상 파일을 읽은 후, import하는 훅·유틸·하위 컴포넌트도 필요한 만큼 읽어 전체 맥락을 파악한다.

파악할 것:

- 무엇을 하는 코드인가 (단일 컴포넌트, 훅, 유틸, 페이지, 여러 모듈 조합)
- 외부 의존성이 있는가 (API 호출, Context, 전역 상태)
- 핵심 비즈니스 로직이 어디에 있는가
- 이미 존재하는 테스트 파일이 있는가 (있으면 읽어 중복 방지)

---

### Step 3. 테스트 유형 선택

`@.claude/rules/test-selection.md`의 결정 트리를 기준으로 유형을 결정한다.

결정 후 사용자에게 아래를 간략히 제시한다:

- 선택한 유형과 이유
- 해당 유형의 한계 (선택에 영향을 줄 정도인 경우만)
- 대안이 있다면 한 줄로 언급

사용자가 다른 유형을 원하면 그에 따른다.

---

### Step 4. 환경 구성 확인

선택한 유형에 필요한 도구가 프로젝트에 설정되어 있는지 확인한다.

| 유형        | 확인 항목                                   |
| ----------- | ------------------------------------------- |
| 단위/통합   | `vitest.config.*` 또는 `jest.config.*`      |
| E2E         | `playwright.config.*`                       |
| 시각적 회귀 | `vitest.config.*`에 `browser.enabled: true` |

**미구성이면 코드를 작성하지 않는다.** 설정 방법을 안내하고 완료 후 진행한다.

---

### Step 5. 테스트 작성

유형에 따라 아래 문서를 참조해 작성한다:

| 유형        | 참조 문서                                                                                                                                   |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 단위/통합   | `@.claude/rules/writing-rules.md`, `@.claude/rules/test-data-strategy.md`, `@references/rtl-patterns.md`, `@references/mocking-patterns.md` |
| E2E         | `@.claude/rules/writing-rules.md`, `@.claude/rules/test-data-strategy.md`, `@references/e2e-best-practices.md`                              |
| 시각적 회귀 | `@references/visual-regression.md`                                                                                                          |

작성 시 반드시 지키는 것:

- `project-config.md`에 커스텀 setup 함수가 있으면 반드시 사용한다
- `project-config.md`의 globals 설정에 따라 import 여부를 결정한다
- 테스트 파일 위치와 네이밍은 `project-config.md`의 관례를 따른다
- AAA 패턴으로 구조화한다
- 디스크립션은 "무엇을 했을 때 어떻게 된다" 형태로 작성한다
- 시간/API 응답/도메인 데이터는 `@.claude/rules/test-data-strategy.md` 기준으로 작성한다
