# 테스트 데이터 전략

테스트가 사용하는 모든 종류의 "입력값"(시간, API 응답, 도메인 데이터)은 **계층화된 기본값 + 분기점에서만 오버라이드** 원칙을 따른다.

베이스가 분명할수록 오버라이드의 의미가 또렷해진다. "이 테스트는 평소와 다른 X 때문에 존재한다"가 코드만으로 드러난다.

---

## 핵심 원칙

| 영역 | 글로벌 기본 | 오버라이드 위치 |
|------|-------------|------------------|
| **시간** | `setupTests`의 `vi.setSystemTime(...)` | 그 시각으로는 검증 못 하는 테스트만 `vi.setSystemTime` 재호출 |
| **API 응답** | `handlers.ts`의 기본 핸들러 (정상 응답) | `server.use()`로 그 테스트만 분기 (404/500/지연 등) |
| **도메인 데이터** | 통과 시나리오를 드러내는 최소 1개 베이스 | `it` 블록 안에서 인라인 정의 — 데이터만 보고 검증 의도가 읽혀야 함 |

부가 원칙:
- **최소화** — 검증 대상이 아닌 필드는 빈 문자열/0 등 최솟값으로 압축. 핵심 필드가 눈에 띄게.
- **자기 설명** — 헬퍼 함수(`makeEvent`, `createMockUser`)로 추출하지 않는다. 데이터 자체가 명세.
- **부작용 격리** — 오버라이드는 그 테스트의 `it` 블록 안에서 시작하고 끝난다. 다음 테스트는 setupTests/handlers의 기본 상태에서 다시 시작.

---

## 시간

### 글로벌 기준

`setupTests`의 `beforeEach`에서 한 시각으로 고정한다. 모든 테스트가 동일한 "현재"를 공유한다.

```ts
// setupTests.ts
beforeAll(() => {
  vi.useFakeTimers({ shouldAdvanceTime: true });
});

beforeEach(() => {
  vi.setSystemTime(new Date('2025-10-01'));
});
```

### 오버라이드 시점

기본 시각으로는 검증할 수 없는 테스트만 `it` 블록 안에서 `vi.setSystemTime`을 재호출한다.

```ts
// ❌ 모든 테스트에서 매번 시간을 직접 세팅 — 베이스가 무의미해진다
it('알림 생성', () => {
  vi.useFakeTimers();
  vi.setSystemTime(new Date('2025-10-01T09:00:00'));
  // ...
});

// ✅ 베이스 그대로 사용. 다른 시각이 필요한 케이스만 명시적 오버라이드
it('알림 생성', () => {
  vi.setSystemTime(new Date(2025, 9, 1, 9, 0, 0)); // 글로벌 UTC midnight과 다른 09:00 local 필요
  // ...
});
```

오버라이드의 코멘트엔 **왜 다른 시각인지**를 적는다 (예: timezone 의존, 특정 경계값).

---

## API 응답 (MSW)

### 글로벌 기본 핸들러

`handlers.ts`는 모든 엔드포인트의 **정상 응답**(200, 기본 데이터)을 정의한다. 대부분 테스트는 이걸로 충분해야 한다.

```ts
// handlers.ts
export const handlers = [
  http.get('/api/events', () => HttpResponse.json({ events })),
  http.post('/api/events', async ({ request }) => { /* 정상 생성 */ }),
  http.put('/api/events/:id', async ({ request }) => { /* 정상 수정 */ }),
  http.delete('/api/events/:id', () => new HttpResponse(null, { status: 204 })),
];
```

### 오버라이드: server.use()

특정 테스트에서 다른 응답이 필요할 때만 `server.use()`로 분기 핸들러를 추가한다. `setupTests`의 `afterEach`가 `server.resetHandlers()`를 호출하므로 다음 테스트로 새지 않는다.

```ts
// ❌ 각 테스트마다 모든 핸들러를 다시 작성 — 글로벌 핸들러가 무의미
it('이벤트를 가져온다', () => {
  server.use(http.get('/api/events', () => HttpResponse.json({ events: [...] })));
  // ...
});

// ✅ 정상 응답은 글로벌 사용. 오류 케이스만 오버라이드
it("로딩 실패 시 '이벤트 로딩 실패' 토스트가 표시된다", () => {
  server.use(http.get('/api/events', () => new HttpResponse(null, { status: 500 })));
  // ...
});
```

### 상태가 필요한 시나리오

CRUD가 한 테스트 안에서 in-memory 상태를 공유해야 하는 경우(예: POST 후 GET) handlersUtils 같은 유틸로 클로저 기반 핸들러를 묶는다. **각 테스트가 자기 인스턴스를 갖고**, `server.resetHandlers()`로 격리된다.

```ts
// handlersUtils.ts
export const setupMockHandlerCreation = (initEvents: Event[] = []) => {
  const events: Event[] = [...initEvents]; // 테스트 고유 클로저
  server.use(
    http.get('/api/events', () => HttpResponse.json({ events })),
    http.post('/api/events', async ({ request }) => {
      const body = await request.json();
      const newEvent = { ...body, id: String(Date.now()) };
      events.push(newEvent);
      return HttpResponse.json({ event: newEvent }, { status: 201 });
    })
  );
};
```

---

## 도메인 데이터

### it 블록 안에 인라인

테스트 데이터는 그 테스트가 드러내려는 의미만큼만, `it` 블록 안에서 직접 선언한다. 헬퍼 함수로 추출하지 않는다.

```ts
// ❌ 헬퍼로 추출 — 내부 default를 추적해야 의도가 보임
const event = makeEvent({ title: '회의' });

// ✅ 인라인 — 그 자리에서 검증 의도가 즉시 읽힘
const event: Event = {
  id: '1',
  title: '회의',
  date: '2025-10-15',
  startTime: '09:00',
  endTime: '10:00',
  description: '',
  location: '',
  category: '업무',
  repeat: { type: 'none', interval: 0 },
  notificationTime: 10,
};
```

### 검증 대상이 눈에 띄게

테스트와 무관한 필드는 빈 문자열/0 등으로 압축. 검증 대상 필드가 두드러지게 한다.

```ts
// ❌ 모든 필드에 의미 있는 값을 채워서 어디가 핵심인지 모름
const event: Event = {
  id: 'evt-2025-10-15-001',
  title: '월간 팀 미팅',
  date: '2025-10-15',
  startTime: '14:30',
  endTime: '15:30',
  description: '10월 OKR 리뷰 및 11월 계획 수립',
  location: '본사 3층 회의실 A',
  category: '업무',
  repeat: { type: 'weekly', interval: 1 },
  notificationTime: 30,
};

// ✅ 이 테스트가 검증하는 건 'startTime'뿐 — 나머지는 최소화
const event: Event = {
  id: '1',
  title: '회의',
  date: '2025-10-15',
  startTime: '09:00',  // ← 검증 대상
  endTime: '10:00',
  description: '',
  location: '',
  category: '업무',
  repeat: { type: 'none', interval: 0 },
  notificationTime: 10,
};
```

### 공유 베이스를 쓰는 경우

같은 파일에서 여러 테스트가 거의 동일한 데이터를 약간만 바꿔 사용하는 경우, 파일 상단에 `baseEvent` 같은 const를 두고 spread로 분기점만 교체한다. 단, 베이스를 만든다고 `it` 블록 안의 인라인 원칙을 깨면 안 된다 — 베이스는 "기본 통과 시나리오 1개"이고, 오버라이드는 명시적이어야 한다.

```ts
const baseEvent: Event = { id: '1', title: '회의', date: '2025-07-01', /* ...최소 필드 */ };

it('두 이벤트가 겹치는 경우 true를 반환한다', () => {
  const event1 = { ...baseEvent, startTime: '09:00', endTime: '11:00' };
  const event2 = { ...baseEvent, startTime: '10:00', endTime: '12:00' };
  expect(isOverlapping(event1, event2)).toBe(true);
});
```

이 패턴이 늘어나면 헬퍼 함수의 유혹이 생기지만, 베이스 const + spread는 **읽는 쪽에서 분기점이 즉시 보인다**는 점에서 함수 호출보다 우월하다.

---

## 교차 참조

- `@.claude/rules/writing-rules.md`: "테스트 데이터 작성 원칙" / "비교값은 항상 리터럴 직접 작성" / "setupTests를 먼저 읽고 작성한다"
- `@references/mocking-patterns.md`: MSW 핸들러, `vi.setSystemTime`, mock 초기화
- `@.claude/templates/setup-tests.ts`: 글로벌 시간 고정 + MSW 등록 권장 구조
