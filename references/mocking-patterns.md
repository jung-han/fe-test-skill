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

실제 구현을 사용할 수 있으면 모킹하지 않는다. 모킹이 많아질수록 테스트가 구현에 결합되어 유지보수 비용이 증가하기 때문이다.

모킹이 필요한 경우는 단위 테스트에서 외부 의존성을 격리할 때로 제한한다.

통합 테스트에서 API를 모킹할 때는 MSW를 사용한다. `vi.fn()`으로 fetch를 직접 모킹하면 실제 네트워크 레이어와 동작이 달라져 테스트 신뢰도가 낮아지기 때문이다.

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
