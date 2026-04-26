# RTL 패턴 (React Testing Library)

## 쿼리 선택

요소를 가져올 때는 아래 우선순위를 따른다. 접근성 트리 기반 쿼리일수록 실제 사용자가 앱을 사용하는 방식에 가깝기 때문이다.

1. `getByRole` — 최우선으로 사용한다
2. `getByLabelText` — form 요소에 사용한다
3. `getByPlaceholderText`
4. `getByText`
5. `getByTestId` — 위 쿼리로 대체 불가한 경우에만 사용한다
6. `container.querySelector` — 사용하지 않는다

```tsx
// ✅
screen.getByRole('button', { name: /제출/i });

// ❌ — 접근성 쿼리로 대체 가능한데 testId 사용
screen.getByTestId('submit-button');
```

네이티브 HTML 요소(`<button>`, `<input>` 등)에 `role` 속성을 중복으로 추가하지 않는다.

---

## 쿼리 변형 선택

- 요소가 **존재함**을 검증할 때는 `getBy*`를 사용한다. 없으면 즉시 에러를 던져 명확한 실패 지점을 알려주기 때문이다.
- 요소가 **없음**을 검증할 때는 `queryBy*`를 사용한다. `getBy*`는 없으면 에러를 던지기 때문에 부재 검증에 쓸 수 없다.
- **비동기로 나타나는** 요소를 기다릴 때는 `findBy*`를 사용한다. `waitFor(() => getBy*())`와 동일하지만 더 명확하다.

```tsx
// ✅ 존재 확인
expect(screen.getByRole('button')).toBeInTheDocument();

// ✅ 부재 확인
expect(screen.queryByText('에러 메시지')).not.toBeInTheDocument();

// ✅ 비동기 대기
const el = await screen.findByText('로딩 완료');

// ❌ — waitFor 안에서 getBy 사용하지 않는다, findBy로 대체한다
await waitFor(() => screen.getByText('로딩 완료'));
```

---

## userEvent

사용자 이벤트를 발생시킬 때는 `userEvent`를 가급적 사용한다. RTL의 핵심 철학이 사용자 시나리오와 유사하게 테스트하는 것이기 때문이다. `fireEvent`는 단일 이벤트만 dispatch하지만, `userEvent`는 실제 브라우저처럼 이벤트 시퀀스를 시뮬레이션한다 (click = mouseOver → mouseDown → focus → mouseUp → click).

프로젝트에 커스텀 `setup` 함수가 있으면 반드시 그것을 사용한다. 없으면 `userEvent.setup()`으로 인스턴스를 생성해서 사용한다. 각 테스트마다 독립적인 인스턴스를 만들어야 테스트 간 입력 장치 상태가 오염되지 않기 때문이다.

```tsx
// ✅ 프로젝트 setup 함수가 있는 경우
const { user } = setup(<MyComponent />);
await user.click(screen.getByRole('button', { name: '저장' }));

// ✅ setup 함수 없는 경우
const user = userEvent.setup();
render(<MyComponent />);
await user.type(screen.getByRole('textbox'), '입력값');
await user.click(screen.getByRole('button'));

// ❌ — fireEvent.change는 사용하지 않는다
fireEvent.change(input, { target: { value: '입력값' } });
```

`fireEvent`는 `userEvent`로 재현이 불가능한 경우(scroll 등)에만 사용한다.

---

## waitFor

비동기 완료를 기다릴 때는 assertion을 `waitFor` 안에 넣는다. 밖에서 검증하면 타이밍이 보장되지 않기 때문이다.

`waitFor` 안에 assertion을 하나만 넣는다. 첫 번째 assertion이 실패하면 이후 assertion이 실행되지 않아 원인 파악이 어렵기 때문이다.

`waitFor` 안에서 이벤트를 발생시키지 않는다. 이벤트는 `waitFor` 밖에서 먼저 실행한다.

```tsx
// ✅
await waitFor(() => expect(mockFn).toHaveBeenCalledTimes(1));
await waitFor(() => expect(a).toBe(1));
expect(b).toBe(2); // 동기 assertion은 waitFor 밖에서

// ❌ — waitFor 밖에서 assertion
await waitFor(() => {});
expect(mockFn).toHaveBeenCalledTimes(1);

// ❌ — waitFor 안에서 이벤트 발생
await waitFor(() => {
  fireEvent.click(button);
  expect(result).toBeVisible();
});
```

---

## act()

`render()`, `userEvent`, `fireEvent`를 `act()`로 감싸지 않는다. 이미 내부적으로 `act()` 처리가 되어 있기 때문이다. 감싸면 오히려 경고가 발생한다.

```tsx
// ❌
act(() => { render(<Component />); });

// ✅
render(<Component />);
```

---

## jest-dom 매처

DOM 상태를 검증할 때는 jest-dom 매처를 사용한다. 원시 DOM 프로퍼티 비교보다 가독성이 높고 실패 메시지가 명확하기 때문이다.

```tsx
// ❌
expect(button.disabled).toBe(true);

// ✅
expect(button).toBeDisabled();
expect(element).toBeInTheDocument();
expect(input).toHaveValue('텍스트');
expect(el).toHaveClass('active');
```

---

## MSW (통합 테스트 API 모킹)

통합 테스트에서 API 응답을 모킹할 때는 MSW를 사용한다. `vi.fn()`으로 fetch를 직접 모킹하면 실제 네트워크 레이어와 동작이 달라져 신뢰도가 낮아지기 때문이다.

```js
// vitest.setup.js — setupTests에 전역 설정
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers()); // 테스트별 핸들러 오염 방지
afterAll(() => server.close());
```

특정 테스트에서 응답을 바꿔야 할 때는 `server.use()`로 핸들러를 교체한다:
```tsx
server.use(
  http.get('/api/user', () =>
    HttpResponse.json({ message: '권한 없음' }, { status: 401 })
  )
);
```

---

## 스냅샷 테스트

스냅샷 테스트는 단순한 구조 변경 추적 목적으로만 제한적으로 사용한다. DOM이 복잡한 컴포넌트에 적용하면 스냅샷이 수십 줄을 넘어가 가독성이 나빠지고, `u` 키 한 번으로 업데이트되어 의도하지 않은 변경을 놓치기 쉽기 때문이다. CSS·레이아웃 변화를 감지해야 한다면 시각적 회귀 테스트를 사용한다.

---

## ESLint 플러그인

아래 플러그인을 설정하면 잘못된 쿼리 사용, act 중복 등 자주 하는 실수를 자동으로 감지한다:

```
eslint-plugin-testing-library
eslint-plugin-jest-dom
```
