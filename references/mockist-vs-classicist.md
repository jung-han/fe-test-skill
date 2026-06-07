# 모킹 학파: London vs Detroit

테스트에서 협력자(collaborator)를 얼마나 모킹할지를 두고 두 학파가 갈린다. JS/React 컴포넌트 테스트 기준으로 정리한다.

핵심 차이는 **"무엇이 올바름의 증거인가"**다. London = 협력자에게 한 호출(behavior), Detroit = 나온 결과(state).

---

## 두 학파

| | London (Mockist) | Detroit (Classicist) |
|--|------------------|----------------------|
| 별칭 | Mockist, outside-in | Chicago, Classical, inside-out |
| 모킹 철학 | 협력자 전부 모킹. 대상만 격리 | 최소화. 진짜 협력자 사용 |
| 검증 대상 | 상호작용(behavior) — "무엇을 호출했나" | 상태(state) — "결과가 뭔가" |
| 단위 정의 | 클래스/모듈 1개 | 동작 단위 (여러 객체 묶음 OK) |
| 모킹 경계선 | 모든 의존성 | 진짜 외부(network, DB, time)만 |
| 원조 | Steve Freeman, Nat Pryce (GOOS) | Kent Beck |

---

## 컴포넌트 예시

대상: `UserProfile` — `useUser` 훅으로 데이터를 가져와 렌더, 저장 버튼을 누르면 `saveUser` API 호출.

```tsx
function UserProfile({ userId }) {
  const { user } = useUser(userId);
  const [name, setName] = useState(user?.name ?? '');
  return (
    <form onSubmit={() => saveUser(userId, { name })}>
      <input value={name} onChange={e => setName(e.target.value)} />
      <button>저장</button>
    </form>
  );
}
```

### London (Mockist)

```tsx
// useUser, saveUser 둘 다 모킹. 컴포넌트만 격리
vi.mock('./useUser');
vi.mock('./api');

it('저장 시 saveUser가 올바른 인자로 호출된다', async () => {
  useUser.mockReturnValue({ user: { name: '철수' } });

  render(<UserProfile userId="1" />);
  await userEvent.clear(screen.getByRole('textbox'));
  await userEvent.type(screen.getByRole('textbox'), '영희');
  await userEvent.click(screen.getByRole('button', { name: '저장' }));

  // 상호작용 검증 — 무엇을 호출했나
  expect(saveUser).toHaveBeenCalledWith('1', { name: '영희' });
});
```

검증 = **호출 발생 여부**. 실제 저장 결과는 보지 않는다.

### Detroit (Classicist)

```tsx
// 진짜 외부(network)만 MSW로 가로챔. 훅·로직은 진짜 사용
const server = setupServer(
  http.get('/api/users/1', () => HttpResponse.json({ name: '철수' })),
  http.put('/api/users/1', () => HttpResponse.json({ ok: true }))
);

it('이름 수정 후 저장하면 변경된 값이 반영된다', async () => {
  render(<UserProfile userId="1" />);

  await screen.findByDisplayValue('철수');
  await userEvent.clear(screen.getByRole('textbox'));
  await userEvent.type(screen.getByRole('textbox'), '영희');
  await userEvent.click(screen.getByRole('button', { name: '저장' }));

  // 결과(상태) 검증 — 사용자가 보는 것
  expect(await screen.findByText('저장 완료')).toBeInTheDocument();
});
```

검증 = **사용자가 보는 결과**. 내부 호출은 알지 못한다.

---

## 장단점

| 관점 | London (Mockist) | Detroit (Classicist) |
|------|------------------|----------------------|
| **검증 대상** | 상호작용 — "무엇을 호출했나" | 상태 — "결과가 뭔가" |
| **속도** | ⚡ 빠름 (전부 가짜) | 🐢 느림 (진짜 실행) |
| **실패 지점** | 명확 (대상 1개만 격리) | 넓음 (협력자 포함) |
| **리팩터 내성** | ❌ 약함 — 내부 구조 바뀌면 깨짐 | ✅ 강함 — 동작 같으면 통과 |
| **통합 버그 탐지** | ❌ 못 잡음 (가짜끼리만 검증) | ✅ 잡음 (진짜 연동) |
| **신뢰도** | 중 — 가짜가 진짜와 다르면 헛통과 | 높음 — 실제 경로 검증 |
| **설계 주도** | ✅ outside-in, 미완성 의존성도 작성 가능 | 협력자 먼저 필요 (inside-out) |
| **셋업 비용** | 낮음 (mock 선언만) | 높음 (MSW, fake timer 등) |
| **실제 사용 유사도** | ❌ 낮음 — 사용자는 "호출"을 안 봄 | ✅ 높음 — 사용자가 보는 결과 검증 |

---

## 권장 (React/RTL)

RTL의 핵심 원칙은 **"테스트가 실제 사용과 닮을수록 신뢰도가 높아진다"**다. 협력자를 모킹할수록 실제 사용과 멀어지므로, RTL은 **Detroit 노선**을 기본으로 한다.

**기본 = Detroit. network·time만 London식 경계 모킹.**

| 대상 | 처리 | 학파 |
|------|------|------|
| 자식 컴포넌트 | 모킹 안 함, 진짜 렌더 | Detroit |
| 커스텀 훅 | 모킹 안 함, 진짜 실행 | Detroit |
| network (fetch/axios) | **MSW로 모킹** | 경계선 — 진짜 외부 |
| 시간/타이머 | **fake timers** | 경계선 — 진짜 외부 |
| 랜덤/UUID | **모킹** | 경계선 — 비결정성 |
| 콜백 prop (`onSubmit` 등) | `vi.fn()` 스파이 | London — 단위 테스트에선 출력=콜백 |

### 판단 기준 한 줄

> **"사용자가 그 차이를 볼 수 있나?"**
> - 볼 수 있음 (화면 결과) → 진짜 사용, 결과 검증 (Detroit)
> - 못 봄 (외부 네트워크/시간/랜덤) → 모킹 (경계 격리)

### 근거

- 협력자를 모킹할수록 실제 사용과 멀어진다 → 모킹 최소화가 신뢰도를 높인다.
- 내부 `toHaveBeenCalled` 남발 = 구현 결합 = 리팩터 시 깨짐 → RTL이 경계하는 안티패턴.
- 단, network/time까지 진짜 쓰면 flaky·느림 → 여기만 London식 격리. 이는 **결정성 확보**가 목적이지 격리 자체가 목적이 아니다.

---

## 안티패턴: 자식 컴포넌트 stubbing / shallow rendering

과거 흔했던 패턴 — 자식 컴포넌트를 가짜 div로 치환하고 `data-testid`로 렌더 여부·prop만 확인.

```tsx
// ❌ 자식을 가짜 div로 대체 — "렌더됐나 + 어떤 prop 받았나"만 검증
vi.mock('./HeavyChart', () => ({
  default: ({ data }) => <div data-testid="heavy-chart" data-len={data.length} />
}));

it('차트 영역이 렌더된다', () => {
  render(<Dashboard />);
  expect(screen.getByTestId('heavy-chart')).toBeInTheDocument();
});
```

원조는 Enzyme의 **shallow rendering** — 한 단계만 렌더하고 자식은 placeholder 태그로 남김.

```jsx
// ❌ Enzyme — 자식을 렌더 안 하고 <Child /> 태그 그대로 둠
const wrapper = shallow(<Parent />);
expect(wrapper.find('Child').prop('userId')).toBe('1');
```

### 왜 사라졌나

| 문제 | 설명 |
|------|------|
| 구현 결합 | 자식 이름·prop 구조 = 내부 세부사항. 리팩터하면 깨짐 |
| 실제와 멀어짐 | 사용자는 자식의 진짜 출력을 봄. 가짜 div는 아무것도 보장 안 함 |
| 통합 갭 | 부모-자식 연동 버그(잘못된 prop 전달, 렌더 조건) 못 잡음 |

Kent C. Dodds가 직격: **"Stop using shallow rendering"**, **"Avoid mocking components"**. Enzyme이 RTL에 밀려난 핵심 이유 중 하나다.

### 지금 권장 — 자식까지 진짜 렌더 (deep render)

```tsx
// ✅ 자식까지 진짜 렌더 — 사용자가 보는 실제 결과 검증
it('대시보드에 사용자 이름이 표시된다', () => {
  render(<Dashboard userId="1" />);
  expect(screen.getByText('철수님 환영합니다')).toBeInTheDocument();
});
```

**예외 — 모킹이 정당한 자식:** 외부 SDK 래퍼(지도·결제 위젯·차트 라이브러리), 무거운 비결정 컴포넌트(애니메이션·canvas). 즉 **inter-system 경계에 걸친 자식**만. 내 도메인 자식은 진짜로 렌더한다.

---

## 🚧 논의 중 — 과한 모킹 방지 기준 (미확정)

> 아래는 **팀 논의 후 결정할 초안**이다. 아직 확정 규칙 아님. AI 자동 판정에 쓸 것을 전제로 정리.

자식 stubbing 같은 안티패턴은 "과하게 모킹하다 보면 미끄러져 들어가는" 결과다. 그 미끄러짐을 막을 판정 기준을 어떻게 세울지가 열린 질문.

### 팀 결정 안건

**안건 1 — count 임계값 채택?**
- A: 숫자 가드레일 (`vi.mock` 3개+ → 경고). 명확·AI친화적, 단 자의적.
- B: 숫자 없이 원인 진단만. 정확하나 AI 판정 어려움.
- C: 숫자는 "경고"로만, "차단" 아님. (AI가 flag, 사람이 결정)

**안건 2 — 앱 유형 분기를 기준에 넣을까?**
- 격리 정당성이 전략에 따라 갈림: 서비스앱(트로피)은 통합으로 올리는 게 맞고, UI Kit(피라미드)은 단위 격리가 정상.
- A: `project-config.md`에 전략 필드 추가 → 판정이 거기 따라감.
- B: 유형 무관 단일 기준 (단순하나 UI Kit서 오탐).
- 연결: `@.claude/rules/test-selection.md`의 트로피/피라미드.

**안건 3 — 자식 컴포넌트 모킹 = 항상 경고?**
- A: 자식 `vi.mock` 무조건 flag, 예외는 주석으로 면제.
- B: 네이밍 allowlist (`*Map`, `*Chart`, `*Payment` 등 외부 래퍼) 자동 면제.

### AI 판정 규칙 초안 (정적 감지)

모호한 "이 모킹이 진실한가"를 **코드에서 검출 가능한 신호**로 환원. 핵심 프록시 = **모킹 대상의 import 경로**(상대경로=내부=의심, 패키지명=외부=허용).

| # | 감지 신호 | 판정 | 신뢰도 |
|---|-----------|------|--------|
| 1 | 모킹 대상 import이 상대경로(`./`,`../`) | 내부 모듈 모킹 → 🟡 경고 | 높음 |
| 2 | 모킹 대상이 `node_modules` 패키지 | 외부 → ✅ 허용 | 높음 |
| 3 | `vi.mock` 팩토리가 JSX 반환(`() => <div/>`) | 자식 stub → 🔴 안티패턴 | 높음 |
| 4 | mock 팩토리에 `data-testid` 포함 | shallow식 stub → 🔴 | 높음 |
| 5 | 한 파일 `vi.mock` 개수 ≥ N | 레이어 오선택 의심 → 🟡 | 중 |
| 6 | `toHaveBeenCalled*` / 전체 단언 ≥ 50% | behavior 과다 → 🟡 | 중 |
| 7 | network/time인데 진짜 호출(비모킹) | 경계 누락 → 🟡 flaky 위험 | 중 |

**판정 흐름**
```
각 vi.mock 발견 시:
1. 대상 경로 분석
   ├─ 패키지명 → 허용 (axios류 network는 "MSW 권장" 제안)
   └─ 상대경로 → 2로
2. 팩토리 반환이 JSX/data-testid?
   ├─ Yes → 🔴 자식 stub 안티패턴
   └─ No  → 🟡 내부 모듈 모킹, 원인 주석 요구
3. 파일 집계(vi.mock 수, behavior 비율) → 임계 초과 시 레이어 재검토 제안
```

### 미해결 쟁점
- count 임계값 N은 자의적 — 규칙 아닌 휴리스틱으로 격하할지.
- "격리가 유일 이유면 통합으로 올려라"는 UI Kit엔 안 맞음 (안건 2와 연결).
- flaky로 깨짐 vs 진짜 통합 버그로 깨짐을 작성자가 구분 못 하는 문제 — 하위 판단 질문 필요.

---

## 참고 문서

- [Mocks Aren't Stubs — Martin Fowler](https://martinfowler.com/articles/mocksArentStubs.html) — 두 학파를 정의한 원전
- [The London vs. Classical schools of TDD](https://www.thoughtworks.com/insights/blog/test-driven-development) — 학파 비교
- [Growing Object-Oriented Software, Guided by Tests (GOOS)](http://www.growing-object-oriented-software.com/) — London 학파 원조
- [Common mistakes with React Testing Library — Kent C. Dodds](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library) — 모킹 안티패턴
- [The Merits of Mocking — Kent C. Dodds](https://kentcdodds.com/blog/the-merits-of-mocking) — 언제 모킹할지
- `@references/mocking-patterns.md` — 이 프로젝트의 Vitest 모킹 실전 패턴
- `@references/rtl-patterns.md` — RTL "사용자처럼 테스트" 원칙
