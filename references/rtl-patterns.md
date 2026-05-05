# RTL 패턴 (React Testing Library)

## 핵심 원칙

> "The more your tests resemble the way your software is used, the more confidence they can give you."
> — Kent C. Dodds

1. **DOM 기반** — 컴포넌트 인스턴스가 아닌 DOM 노드와 상호작용한다.
2. **사용자 중심** — 실제 사용자가 앱을 다루는 방식과 유사하게 테스트한다.
3. **단순/유연** — API와 구현은 단순해야 한다.

이후 모든 쿼리·이벤트 권장사항은 이 원칙에서 파생된다.

출처: https://testing-library.com/docs/guiding-principles

---

## 쿼리 선택

요소를 가져올 때는 **우선순위**가 있다. 접근성 트리 기반일수록 실제 사용자의 사용 방식에 가깝기 때문이다. **위 단계부터 시도하고, 불가능할 때만 아래로 내려간다.**

### 1단계 — 누구나 접근 가능 (Accessible to Everyone)
스크린리더 사용자를 포함한 모든 사용자가 인지하는 방식.

| 우선 | 쿼리 | 용도 |
|------|------|------|
| 1 | `getByRole` | 거의 모든 인터랙티브 요소. `name` 옵션으로 필터. |
| 2 | `getByLabelText` | form 필드. 사용자가 라벨로 입력칸을 찾는 방식. |
| 3 | `getByPlaceholderText` | 라벨이 없는 경우만 (라벨이 권장). |
| 4 | `getByText` | 비-인터랙티브 요소(div, span 등)의 텍스트. |
| 5 | `getByDisplayValue` | 값이 채워진 폼 요소 (편집 화면 초기 상태 검증). |

### 2단계 — 시맨틱 (Semantic)
브라우저/스크린리더 지원이 일관되지 않을 수 있음.

| 우선 | 쿼리 | 용도 |
|------|------|------|
| 6 | `getByAltText` | `<img>`, `<area>`, `<input type="image">`. |
| 7 | `getByTitle` | `title` 속성. 지원 제한적이라 비추. |

### 3단계 — Test ID
사용자에게 보이지 않음. 1·2단계로 도저히 안 될 때만.

| 우선 | 쿼리 | 용도 |
|------|------|------|
| 8 | `getByTestId` | role/text로 구분 불가능한 dynamic content. |

사용 금지: `container.querySelector` — 클래스/ID는 사용자에게 의미 없다.

```tsx
// ✅ 1단계
screen.getByRole('button', { name: /제출/i });

// ❌ 1단계로 가능한데 testId 사용
screen.getByTestId('submit-button');
```

네이티브 HTML 요소(`<button>`, `<input>` 등)에 `role` 속성을 중복으로 추가하지 않는다.

---

## 쿼리 변형 선택

목적에 맞게 변형을 고른다. 0/1/>1 매치 시 동작이 다르므로 **검증하려는 상황과 반환값을 일치시켜야** 한다.

| 쿼리 | 0 매치 | 1 매치 | >1 매치 | 비동기 |
|------|--------|--------|---------|--------|
| `getBy...`      | throw  | element | throw | ✗ |
| `queryBy...`    | null   | element | throw | ✗ |
| `findBy...`     | throw  | element | throw | ✓ |
| `getAllBy...`   | throw  | array   | array | ✗ |
| `queryAllBy...` | []     | array   | array | ✗ |
| `findAllBy...`  | throw  | array   | array | ✓ |

선택 기준:
- 요소가 **존재함**을 검증 → `getBy*` (없으면 즉시 throw, 명확한 실패 지점).
- 요소가 **없음**을 검증 → `queryBy*` (`getBy*`는 throw하므로 부재 검증 불가).
- 요소가 **비동기로 등장** → `findBy*` (`waitFor(() => getBy*())`와 동일하지만 더 명확).
- 여러 개 → `*All*` 변형.

기본 타임아웃: `findBy*` / `waitFor` = 1000ms, 폴링 50ms.

```tsx
// ✅ 존재
expect(screen.getByRole('button')).toBeInTheDocument();

// ✅ 부재
expect(screen.queryByText('에러 메시지')).not.toBeInTheDocument();

// ✅ 비동기 대기
const el = await screen.findByText('로딩 완료');

// ❌ — waitFor 안에서 getBy 사용 금지, findBy로 대체
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
