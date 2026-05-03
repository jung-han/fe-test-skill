# 테스트 작성 원칙

## 핵심 4원칙

**1. 인터페이스(명세) 기준으로 작성**
내부 구현(state 직접 접근, private 메서드) 테스트 금지. 외부 동작 기준으로 작성.
컴포넌트는 사용자가 사용하는 방식으로 — 테스트와 실제 사용 방식이 유사할수록 신뢰도 향상.

```tsx
// ❌ 구현 기반 — 내부 state 직접 접근
expect(component.state.isOpen).toBe(true);

// ✅ 명세 기반 — 사용자가 보는 결과로 검증
expect(screen.getByRole("dialog")).toBeInTheDocument();
```

**2. 커버리지보다 의미있는 테스트**
간단한 유틸 함수는 생략 가능. 다른 모듈에 포함 시 통합 테스트에서 함께 검증이 더 효율적.
100% 커버리지 강제 금지 — 수확 체감 법칙 적용됨.

**3. 테스트가 문서다**
명확한 디스크립션 → 파일만 봐도 앱 동작 파악 가능.
`describe`로 맥락, `it`으로 동작 명세.

```tsx
// ❌
it('test1', () => { ... });

// ✅
describe('TextField', () => {
  describe('placeholder', () => {
    it('기본 placeholder "텍스트를 입력해 주세요."가 노출된다.', async () => { ... });
    it('placeholder prop 전달 시 해당 텍스트로 노출된다.', async () => { ... });
  });
});
```

**4. 하나의 테스트 = 하나의 동작**
여러 시나리오는 `it` 분리. 실패 시 정확한 포인트 파악 가능.

---

## 테스트 패턴: AAA 패턴을 사용한다.

**AAA (Arrange-Act-Assert)**

```tsx
it("className prop으로 설정한 css class가 적용된다.", async () => {
  // Arrange — 환경 준비
  await render(<TextField className="my-class" />);

  // Act — 동작 재현 (렌더링 검증만이면 생략 가능)
  // 클릭, 입력, prop 변경 등이 여기 해당

  // Assert — 결과 검증
  expect(screen.getByPlaceholderText("텍스트를 입력해 주세요.")).toHaveClass(
    "my-class",
  );
});
```

---

## 테스트 독립성

모든 테스트는 순서에 관계없이 독립 실행 가능해야 함.
순서가 바뀌었을 때 실패한다면 독립적으로 작성되지 않은 것.

```tsx
beforeEach(async () => {
  await prepareModules(); // 각 테스트 전 초기화
});

afterEach(async () => {
  await destroyModules(); // 각 테스트 후 정리
});

beforeAll(async () => {
  await startMockingServer(); // 전체 suite 시작 전 1회
});

afterAll(async () => {
  await stopMockingServer(); // 전체 suite 종료 후 1회
});
```

전역 설정은 `setupFiles`:

```js
// vitest.config.js
test: {
  setupFiles: ["vitest.setup.js"];
}
```

---

## Vitest API 선택 기준

### 테스트 작성

```tsx
// 기본
it('동작 설명', () => { ... });
test('동작 설명', () => { ... }); // it의 alias

// 일시적으로 하나만 실행 — 로컬 디버깅용, 커밋 금지
test.only('이것만 실행', () => { ... });
describe.only('이 그룹만 실행', () => { ... });

// 일시적으로 스킵 — 로컬 디버깅용, 커밋 금지
test.skip('스킵할 테스트', () => { ... });
describe.skip('스킵할 그룹', () => { ... });

// 구현 예정 표시 — 보고서에 TODO로 표시됨
test.todo('나중에 작성할 테스트');

// 특정 환경에서만 실행
const isDev = process.env.NODE_ENV === 'development';
test.runIf(isDev)('개발 환경에서만 실행', () => { ... });
```

> `test.only` / `describe.only`는 커밋하지 않도록 ESLint 규칙(`no-only-tests`) 설정 권장.

### 병렬/순차 실행

```tsx
// 기본은 순차 실행
describe('suite', () => {
  test('test 1', async () => { ... });
  test('test 2', async () => { ... });
});

// 병렬 실행 — 각 테스트가 서로 독립적이고 순서 무관할 때만
describe('suite', () => {
  test.concurrent('parallel 1', async () => { ... });
  test.concurrent('parallel 2', async () => { ... });
});

// describe 전체를 병렬로
describe.concurrent('suite', () => {
  test('test 1', async () => { ... }); // 전부 병렬
  test('test 2', async () => { ... });
  test.sequential('sequential', async () => { ... }); // 이것만 순차
});
```

### 그룹화

```tsx
describe('컴포넌트명 또는 기능명', () => {
  describe('세부 기능 또는 시나리오', () => {
    it('구체적인 동작', () => { ... });
  });
});
```

테스트 보고서가 계층 구조로 출력되어 가독성 향상.
