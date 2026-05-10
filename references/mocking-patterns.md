# 모킹 패턴 (Vitest)

## globals 설정

`vitest.config`에 `globals: true`가 설정된 프로젝트에서는 `vi`, `describe`, `it`, `test`, `expect`, `beforeEach` 등을 **import 없이** 사용.
설정 여부는 `project-config.md` 또는 `vitest.config.*` 확인.

```ts
// globals: true — import 불필요
vi.fn();
vi.spyOn(obj, 'method');

// globals: false — import 필요
import { vi } from 'vitest';
```

---

## 모킹 원칙

**가능한 한 모킹하지 않는다.** 모킹이 많아질수록 테스트가 구현에 결합되어 신뢰도가 낮아지고 유지보수 비용이 증가한다. RTL의 핵심 원칙(`@references/rtl-patterns.md`의 "사용자처럼 테스트")과 직결된다.

### 결정 트리 — inter-system vs intra-system

```
무엇을 모킹할지 고민 중이면 먼저 분류:

├─ 외부 시스템 (HTTP, 시간, FS, 브라우저 API, 외부 SDK)
│   = inter-system communication
│   → 모킹 정당. 어떤 도구를 쓸지는 "모킹 대상별 도구 매핑" 표 참조.
│
└─ 내부 모듈 (도메인 hook, 유틸, 같은 시스템 내 컴포넌트)
    = intra-system communication
    → 가능하면 실제 사용. 모킹은 마지막 수단.
        ├─ 격리해야 진짜 단위 테스트가 되는 작은 함수 → 예외적으로 vi.fn / vi.spyOn 허용
        └─ 외에는 통합 테스트로 묶어 실제 협력 검증 (Kent C. Dodds)
```

근거: 도메인 내부 협력은 "구현 세부사항"이라 모킹하면 리팩터링에 깨지고 신뢰도 잃음. 외부 경계는 빠져나가는 지점이라 모킹이 격리에 정당.

### 통합 테스트의 API 모킹

`vi.fn()`으로 fetch/axios를 직접 모킹하지 않는다. 네트워크 레이어와 동작이 달라져 신뢰도가 떨어진다. **MSW**로 네트워크 경계에서 인터셉트한다 (아래 "MSW" 섹션).

---

## Test Double — 무엇을 쓰고 있는지 인식

Vitest API를 쓸 때 "이게 5종 중 어느 것인지" 의식하면 의도가 명확해지고 안티패턴을 식별하기 쉽다. 분류는 Gerard Meszaros / Martin Fowler의 정의를 따른다.

| 종류 | 정의 | Vitest 매핑 |
|------|------|-------------|
| **Dummy** | 자리 채우기용. 실제 호출 안 됨. | `vi.fn()` (단언 없이 prop으로만 전달) |
| **Stub** | 미리 정해진 응답 반환. 호출 검증 X. | `vi.fn().mockReturnValue(...)`, `mockResolvedValue(...)` |
| **Spy** | stub + 호출 정보 기록. 원본 동작 유지 가능. | `vi.spyOn(obj, 'method')` |
| **Mock** | 호출 기대치를 사전 프로그래밍 → behavior verification. | `vi.fn()` + `expect(fn).toHaveBeenCalledWith(...)` |
| **Fake** | 동작하는 구현이지만 프로덕션 부적합 (in-memory DB 등). | MSW의 in-memory CRUD 핸들러, 자체 작성 fake module |

핵심 차이: **Mock만 행동(behavior) 검증**. 나머지는 상태(state) 검증.

자주 헷갈리는 지점:
- `vi.fn()`은 **Stub과 Mock의 어느 쪽이든** 될 수 있다. `expect(fn).toHaveBeenCalledWith(...)`로 호출을 검증하면 Mock이고, 단지 응답만 제공하면 Stub이다.
- `vi.spyOn()`은 기본 Spy(원본 유지). `.mockReturnValue()`까지 붙이면 Stub처럼 동작.

> 안티패턴: Mock이 필요한데 Stub로 멈춤 → 호출 검증 누락. 반대로 Stub이면 충분한데 Mock으로 작성 → 구현 결합.

---

## 모킹 대상별 도구 매핑

본 프로젝트의 표준 매핑. **setupTests에 이미 등록된 패턴이면 거기에 맞춰 사용**하고, 새로 정의하지 않는다.

| 모킹 대상 | 권장 도구 | 비고 |
|-----------|-----------|------|
| 시간 | `vi.setSystemTime` / `vi.useFakeTimers` | setupTests에서 글로벌 고정 권장 (`@.claude/rules/test-data-strategy.md`) |
| HTTP | MSW (`setupServer` + `handlers.ts`) | setupTests에서 server.listen, 테스트별 `server.use`로 분기 |
| 모듈 (router 등) | `vi.mock` | 호이스팅됨, 파일 상단 위치 무관 |
| 환경변수 | `vi.stubEnv` | afterEach에서 unstub 또는 `vi.unstubAllEnvs()` |
| 글로벌 (alert, location 등) | `vi.stubGlobal` + `vi.unstubAllGlobals` 짝 | `clearAllMocks`/`resetAllMocks`는 stubGlobal 복원 안 함 — 명시적 unstub 필요 |
| 콜백 props | `vi.fn` | 호출 인자/횟수 검증 |
| 객체 메서드 | `vi.spyOn` | 원본 유지하며 추적 |

setupTests에 이미 정의된 항목이면 테스트 본문에서 다시 호출하지 않는다 (중복 setup 방지 — `@.claude/rules/writing-rules.md` "setupTests를 먼저 읽고 작성한다").

---

## 함수 모킹

### vi.fn() — 빈 mock 함수 생성

```tsx
const mockFn = vi.fn();

mockFn('arg1');

expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledWith('arg1');
expect(mockFn).toHaveBeenCalledTimes(1);
```

### 구현 주입

```tsx
// 반환값 고정
const mockFn = vi.fn().mockReturnValue('result');

// 비동기 반환값
const mockFn = vi.fn().mockResolvedValue({ data: [] });

// 에러 반환
const mockFn = vi.fn().mockRejectedValue(new Error('실패'));

// 호출마다 다른 값 (순서대로 소진)
const mockFn = vi.fn()
  .mockReturnValueOnce('first')
  .mockReturnValueOnce('second')
  .mockReturnValue('default'); // 이후 모든 호출

// 커스텀 구현
const mockFn = vi.fn().mockImplementation((x) => x * 2);

// 첫 번째 호출만 다른 구현
const mockFn = vi.fn()
  .mockImplementationOnce(() => 'first call')
  .mockImplementation(() => 'subsequent calls');
```

---

## 스파이 (vi.spyOn)

기존 객체의 메서드를 감시하거나 교체. 원본 구현 유지하면서 호출 추적 가능.

```tsx
const calculator = {
  add: (a: number, b: number) => a + b,
};

// 원본 구현 유지하면서 호출만 추적
const spy = vi.spyOn(calculator, 'add');
calculator.add(1, 2);
expect(spy).toHaveBeenCalledWith(1, 2);

// 구현 교체
vi.spyOn(calculator, 'add').mockReturnValue(10);
expect(calculator.add(1, 2)).toBe(10);

// 테스트 후 원본 복원
spy.mockRestore();
```

### 컴포넌트에서 자주 쓰는 패턴

```tsx
// props로 받은 콜백 함수 추적
it('onChange가 올바른 값으로 호출된다.', async () => {
  const handleChange = vi.fn();
  await render(<TextField onChange={handleChange} />);

  await user.type(screen.getByRole('textbox'), 'hello');

  expect(handleChange).toHaveBeenLastCalledWith('hello');
});
```

---

## 모킹 후 실행 내역 확인

```tsx
const mockFn = vi.fn();
mockFn('a', 'b');
mockFn('c');

mockFn.mock.calls         // [['a', 'b'], ['c']]
mockFn.mock.calls[0]      // ['a', 'b']
mockFn.mock.lastCall      // ['c']
mockFn.mock.results       // [{ type: 'return', value: undefined }, ...]
```

---

## 모듈 모킹

### vi.mock() — 모듈 전체 모킹

```tsx
// 자동 모킹 (모든 export를 vi.fn()으로 교체)
vi.mock('@/utils/api');

// 커스텀 구현
vi.mock('@/utils/api', () => ({
  fetchUser: vi.fn().mockResolvedValue({ id: 1, name: '홍길동' }),
}));
```

> `vi.mock()`은 호이스팅됨 — 파일 상단에 위치하지 않아도 실제로는 import보다 먼저 실행.

### 자주 모킹하는 패턴

```tsx
// Next.js router
vi.mock('next/navigation', () => ({
  useRouter: () => ({ push: vi.fn(), replace: vi.fn(), back: vi.fn() }),
  usePathname: () => '/',
  useSearchParams: () => new URLSearchParams(),
}));

// 환경변수
vi.stubEnv('VITE_API_URL', 'http://localhost:3000');

// 날짜 (타이머 모킹 대신 특정 날짜 고정)
vi.setSystemTime(new Date('2024-01-01'));
```

---

## 타이머 모킹

실제 타이머(`setTimeout`, `setInterval`)를 사용하면 테스트가 실제 시간만큼 대기해야 함.
`vi.useFakeTimers()`로 타이머를 제어 가능하게 교체.

```tsx
it('300ms debounce 후 함수가 1회 호출된다.', () => {
  vi.useFakeTimers();

  const spy = vi.fn();
  const debouncedFn = debounce(spy, 300);

  debouncedFn();
  vi.advanceTimersByTime(200); // 200ms 경과
  debouncedFn();               // 타이머 리셋

  vi.advanceTimersByTime(300); // 300ms 경과

  expect(spy).toHaveBeenCalledTimes(1);

  vi.useRealTimers(); // 반드시 복원
});
```

```tsx
// afterEach에서 복원하는 것이 안전
afterEach(() => {
  vi.useRealTimers();
});
```

---

## 모킹 초기화 3종

| 메서드 | 초기화 대상 | 사용 시점 |
|--------|------------|----------|
| `vi.clearAllMocks()` | 호출 내역(calls, results)만 | `afterEach` |
| `vi.resetAllMocks()` | 호출 내역 + 구현(mockReturnValue 등) | `afterAll` |
| `vi.restoreAllMocks()` | 호출 내역 + 구현 + spyOn 원본 복원 | `afterAll` (spyOn 사용 시) |

권장 패턴 (`setupTests.ts`에 전역 설정):
```ts
afterEach(() => {
  vi.clearAllMocks(); // 테스트 간 호출 내역 격리
});

afterAll(() => {
  vi.resetAllMocks(); // suite 간 구현 격리
  vi.useRealTimers(); // 타이머 복원
});
```

참고: https://medium.com/@yujso66/번역-당신의-jest-테스트는-잘못되어-있을-수도-있습니다-866f5f982ff9

---

## 부록 — 학파 컨텍스트

테스트 학파는 두 갈래로 나뉜다 (Martin Fowler, "Mocks Aren't Stubs"):

- **Classical (Detroit)** — 가능하면 실제 객체 사용. 외부 시스템(DB, HTTP, FS)만 모킹. 상태 검증.
- **Mockist (London)** — 모든 협력자(neighboring class)를 모킹. 행동 검증, outside-in.

**본 프로젝트는 Classical 입장.** 위 "모킹 원칙"의 결정 트리(inter/intra-system)가 Vladimir Khorikov의 정리("intra-system communications are implementation details") 그대로다. 결과적으로 Kent C. Dodds의 "Write tests. Not too many. Mostly integration."과도 부합한다 — 통합 테스트가 신뢰도 ROI 최고.

---

## MSW — API 모킹 (통합 테스트)

함수 모킹으로 fetch/axios를 직접 모킹하지 말 것. MSW로 네트워크 레이어에서 인터셉트.

```ts
// mock/handlers.ts
import { http, HttpResponse } from 'msw'; // msw v2

export const handlers = [
  http.get('/api/user', () =>
    HttpResponse.json({ id: 1, name: '홍길동' })
  ),
  http.post('/api/login', () =>
    HttpResponse.json({ token: 'abc123' })
  ),
];

// 실패 케이스
http.get('/api/user', () =>
  HttpResponse.json({ message: '권한 없음' }, { status: 401 })
),
```

```ts
// vitest.setup.ts
import { setupServer } from 'msw/node';
import { handlers } from './mock/handlers';

const server = setupServer(...handlers);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers()); // 핸들러 오염 방지
afterAll(() => server.close());
```

특정 테스트에서 핸들러 교체:
```tsx
it('API 실패 시 에러 메시지를 표시한다.', async () => {
  server.use(
    http.get('/api/user', () =>
      HttpResponse.json({ message: '서버 오류' }, { status: 500 })
    )
  );

  await render(<UserProfile />);
  expect(await screen.findByText('오류가 발생했습니다.')).toBeInTheDocument();
});
```
