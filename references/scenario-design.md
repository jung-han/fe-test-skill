# 시나리오 도출 기법

테스트를 작성하기 전에 **무엇을 검증할 것인지를 빠뜨리지 않고 도출**한다.
직관과 경험에만 의존하면 핵심 케이스를 놓치고, 의미 없는 케이스를 만든다.
본 문서는 7가지 black-box 도출 기법을 프런트엔드 컨텍스트로 매핑해 정리한다.

> 출처: ISTQB foundation / Boris Beizer / 사용자 번역물 4편 (`@references/_research/07-user-translations.md`)

---

## 도출 워크플로우

테스트 종류(`@.claude/rules/test-selection.md`)를 결정한 뒤, 아래 순서로 시나리오를 도출한다.

```
1. Use Case 식별 — 사용자 트랜잭션의 시작과 끝을 정의
   ↓
2. 트랜잭션 입력 분석
   ├─ 단일 입력 → ECP + BVA
   ├─ 다중 조건 조합 → Decision Table
   │     └─ 케이스 폭발 시 → Pairwise
   └─ 상태가 있는 흐름 → State Transition
   ↓
3. 함수형 유틸이면 → Property-Based 추가 고려
   ↓
4. 위 기법으로 잡히지 않는 엣지 → Error Guessing으로 보완
```

---

## 1. Use Case — 사용자 트랜잭션 식별

### 정의
시작부터 끝까지 한 사용자의 트랜잭션 단위로 시나리오를 정의하는 기법.
"무엇을 / 누가 / 어떤 조건에서 / 어떻게 / 결과는" 을 구조화한다.

### 구성 요소
| 요소 | 의미 |
|------|------|
| Actor | 시스템과 상호작용하는 주체 (사용자, 외부 시스템) |
| Precondition | 시작 전 충족되어야 할 조건 (로그인 상태, 데이터 존재 등) |
| Main Flow | 정상 경로의 단계 |
| Alternative Flow | 정상 외 경로 (다른 입력 / 분기 선택) |
| Exception Flow | 실패·에러 경로 |
| Postcondition | 종료 후 시스템 상태 |

### 절차
1. 도메인을 사용자가 수행하는 트랜잭션으로 분해.
2. 각 트랜잭션에 actor·precondition·main·alternative·exception·postcondition을 채운다.
3. 각 흐름이 1개 이상의 테스트 시나리오가 된다.

### 프런트엔드 예시 — 장바구니 결제

```
Actor: 로그인된 구매자
Precondition:
  - 사용자 로그인 상태
  - 장바구니에 1개 이상 상품
  - 결제 수단 등록됨

Main Flow:
  1. 장바구니 → "구매하기" 클릭
  2. 배송 정보 폼 작성
  3. 쿠폰 선택 (optional)
  4. "결제하기" 클릭
  5. 결제 성공 화면 → 메인으로

Alternative Flow:
  A1. 쿠폰 선택 안 함 → 정가 결제
  A2. 적립금 사용 → 차감 후 결제

Exception Flow:
  E1. 배송 정보 누락 → 에러 메시지
  E2. 결제 API 실패 (5xx) → 재시도 안내
  E3. 카드 한도 초과 → 결제 거절 메시지

Postcondition:
  - 주문 생성, 장바구니 비워짐, 메인 페이지로 이동
```

위 구조로 통합 + E2E 시나리오가 도출된다 (Main = 골든패스, Alternative + Exception = 보호장치).

### 한계
- "어떤 입력값으로 검증할지"는 안 잡힘 → ECP/BVA로 보완.
- 흐름이 단순한 트랜잭션엔 과한 형식.

### 결합
- Use Case → ECP/BVA: 각 step의 입력 검증.
- Use Case → State Transition: 다단계 폼·세션은 상태로 모델링.

---

## 2. Equivalence Partitioning (ECP) — 입력 동치 분할

### 정의
입력 도메인을 **동일하게 동작하는 클래스**로 나누고, 각 클래스에서 대표 1개씩 뽑는다.
"같은 클래스 안의 어떤 값으로 통과하면, 클래스 안 모든 값이 통과한다"는 가정.

### 클래스 종류
| 종류 | 예시 |
|------|------|
| 유효 (valid) | 0~999개 수량 입력에서 5 |
| 무효 (invalid) | -1, 1000 |
| 범위 (range) | 18~64세 → 한 클래스 |
| 특정값 (specific) | "관리자" 권한 |
| 집합 (set) | ["VISA", "MasterCard", "JCB"] |
| 불리언 (boolean) | 동의/거부 |

### 절차
1. 입력 변수 식별.
2. 각 변수의 도메인을 유효·무효 클래스로 분할.
3. 클래스당 대표값 1개 선택 → 테스트 케이스.

### 프런트엔드 예시 — 회원가입 연령 입력 (만 14세 이상)

| 클래스 | 대표값 | 종류 |
|--------|--------|------|
| 14~120세 | 30 | 유효 |
| 0~13세 | 10 | 무효 (미성년) |
| 음수 | -1 | 무효 |
| 비숫자 | "abc" | 무효 |
| 공백 | "" | 무효 |

→ 5개 케이스로 모든 분기를 커버.

### 한계
- 클래스 내부 동등성은 가정. 실제로는 경계 근처에서 결함이 자주 발생 → BVA로 보완.

### 결합
- ECP + BVA → 표준 결합. 클래스 분할 후 각 경계까지 함께 검증.

---

## 3. Boundary Value Analysis (BVA) — 경곗값 분석

### 정의
유효/무효 영역 사이의 경계에서 결함 빈도가 가장 높음. 경계 양쪽을 집중 검증.

### 경계 값 후보 (1 변수당)
- min - 1 (just outside)
- min (boundary)
- min + 1 (just inside)
- nominal (대표값, ECP에서 가져옴)
- max - 1
- max
- max + 1

n개 변수 → 최대 4n + 1 케이스 (Standard) 또는 6n + 1 (Robust).

### 프런트엔드 예시 — 비밀번호 길이 8~20자

| 케이스 | 입력 길이 | 기대 동작 |
|--------|-----------|-----------|
| 7 | "1234567" | 무효 (just outside lower) |
| 8 | "12345678" | 유효 (boundary) |
| 9 | "123456789" | 유효 (just inside lower) |
| 14 | "..." | 유효 (nominal) |
| 19 | "..." | 유효 (just inside upper) |
| 20 | "..." | 유효 (boundary) |
| 21 | "..." | 무효 (just outside upper) |

### 프런트엔드 예시 — 페이지네이션 (총 100개, 페이지당 20개 → 5페이지)

경계: 1, 5 (페이지 번호) / 1, 20 (페이지 내 항목 인덱스)
- 1 페이지의 첫 항목·마지막 항목 표시
- 5 페이지의 첫 항목·마지막 항목 표시
- 0 페이지 / 6 페이지 요청 → 에러 또는 fallback

### 한계
- 다중 조건 조합은 BVA 단독으로 부족 → Decision Table.

---

## 4. Decision Table — 다중 조건 매트릭스

### 정의
여러 조건의 **모든 조합 → 결과**를 표로 매핑. 비즈니스 규칙·복합 검증에 강함.

### 절차
1. 조건(원인) 식별.
2. 액션(결과) 식별.
3. 조건의 참/거짓 조합을 표 열로 나열 (n개 조건 → 2ⁿ 열).
4. 각 조합에 대응하는 액션을 채움.
5. 각 열 = 1 테스트 케이스.

### 프런트엔드 예시 — 결제 버튼 활성화

조건: ① 배송 정보 입력됨 ② 결제 수단 선택됨 ③ 약관 동의됨

| 조건 \ 케이스 | C1 | C2 | C3 | C4 | C5 | C6 | C7 | C8 |
|---------------|----|----|----|----|----|----|----|----|
| 배송 정보 | T | T | T | T | F | F | F | F |
| 결제 수단 | T | T | F | F | T | T | F | F |
| 약관 동의 | T | F | T | F | T | F | T | F |
| **결과: 버튼 활성** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

→ 8 케이스 중 1개만 활성화. 모든 조건의 AND 동작 확인.

### 케이스 폭발 대응
- 조건 5개 → 32 케이스. 매니지먼트 어려움.
- 대응: ① 조건 합성 (의미가 같은 조건 묶기) ② Pairwise(다음 항목)로 전환.

### 한계
- 조건이 boolean이 아닌 enum이면 케이스 수가 nᵏ로 폭발.
- 조건 간 의존성(서로 모순) 표현 어려움 → 표에 "—" 또는 "N/A" 명시.

### 결합
- Decision Table → ECP/BVA: 각 조건의 참/거짓 자체가 ECP의 입력일 때 BVA로 깊이 추가.

---

## 5. State Transition — 상태 흐름

### 정의
시스템을 유한한 상태들과 이벤트에 의한 전이로 모델링.

### 4 구성 요소
| 요소 | 의미 |
|------|------|
| State | 시스템의 식별 가능한 조건 (LoggedOut, LoggedIn, Locked) |
| Event | 전이를 일으키는 트리거 (login click, timeout) |
| Transition | 한 상태 → 다른 상태로의 이동 |
| Action | 전이의 부수효과 (token 저장, redirect) |

### 표현
- **상태 다이어그램**: 박스(상태) + 화살표(전이). 시각적.
- **상태 전이 표**:

| 현재 상태 | 이벤트 | 다음 상태 | 액션 |
|-----------|--------|-----------|------|
| LoggedOut | login(valid) | LoggedIn | token 저장 |
| LoggedOut | login(invalid×3) | Locked | 잠금 메시지 |
| LoggedIn | logout | LoggedOut | token 삭제 |
| LoggedIn | timeout(30min) | LoggedOut | 세션 만료 |
| Locked | unlock | LoggedOut | 잠금 해제 |

### 절차
1. 상태 열거.
2. 각 상태에서 가능한 이벤트 열거.
3. 표 채움.
4. 각 행 = 1 테스트 케이스.
5. **무효 전이**도 검증 (예: Locked에서 login 클릭 무시).

### 프런트엔드 예시 — 다단계 폼 (배송 → 결제 → 확인)

상태: Step1, Step2, Step3, Submitted
이벤트: next, prev, submit, validate-fail

```
Step1 → next(valid) → Step2
Step1 → next(invalid) → Step1 (에러 표시)
Step2 → next(valid) → Step3
Step2 → prev → Step1
Step3 → submit(API success) → Submitted
Step3 → submit(API 5xx) → Step3 (에러 표시)
Step3 → prev → Step2
```

→ 각 화살표가 1 테스트.

### 적합·부적합
- 적합: 로그인/세션, 모달 open/close, 다단계 폼, 결제 진행, 드로어, 온/오프라인.
- 부적합: 상태가 무한·연속(게임 캐릭터 위치 등).

---

## 6. Pairwise (All-Pairs) — 조합 폭발 방지

### 정의
모든 변수쌍의 조합만 보장 → 케이스 수 급감. "결함의 60~95%는 두 인자 상호작용에서 발생" 관찰에 기반.

### 효과 예시
3개 변수 (각 옵션 2/3/4) → 전체 조합 = 24 → Pairwise = **12** (절반).
변수가 늘수록 절감 폭 커짐.

### 도구
- [PICT](https://github.com/microsoft/pict) (Microsoft, CLI)
- [ACTS](https://csrc.nist.gov/projects/automated-combinatorial-testing-for-software) (NIST)
- [pairwise.org](https://www.pairwise.org/) (온라인 도구 모음)

### 프런트엔드 예시 — 상품 필터 화면

변수:
- 카테고리: [전자, 의류, 식품]
- 가격대: [저, 중, 고]
- 정렬: [최신, 인기, 가격]
- 재고: [O, X]

전체 조합: 3 × 3 × 3 × 2 = 54 케이스. Pairwise: ~9~12 케이스.

도구 출력 예 (PICT 입력):
```
카테고리: 전자, 의류, 식품
가격대:   저, 중, 고
정렬:     최신, 인기, 가격
재고:     O, X
```
→ 9개의 조합으로 모든 쌍을 커버.

### 한계
- 3개 변수 동시 상호작용에서 발생하는 결함은 미보장 (드물다고 가정).
- 핵심 비즈니스 로직(결제 등)은 Decision Table 또는 전체 조합.

---

## 7. Property-Based — 불변식 기반

### 정의
"입출력 예시" 대신 "어떤 입력이든 만족해야 할 속성"을 명시. 라이브러리가 입력을 자동 생성·축소(shrinking).

### 라이브러리 — fast-check
```ts
import fc from 'fast-check';

it('정렬 함수의 길이는 보존된다', () => {
  fc.assert(
    fc.property(fc.array(fc.integer()), (arr) => {
      expect(sort(arr).length).toBe(arr.length);
    })
  );
});

it('정렬 후 인접 원소는 비내림차순', () => {
  fc.assert(
    fc.property(fc.array(fc.integer()), (arr) => {
      const sorted = sort(arr);
      for (let i = 1; i < sorted.length; i++) {
        expect(sorted[i]).toBeGreaterThanOrEqual(sorted[i - 1]);
      }
    })
  );
});
```

### 프런트엔드 적합 영역
- 정렬·필터·집계 유틸.
- 폼 검증 함수 (`isValidEmail` 등).
- URL/쿼리스트링 파싱·직렬화 (왕복 항등).
- 상태 머신의 invariant (어떤 이벤트 시퀀스를 줘도 invalid 상태로 빠지지 않음).

### 프런트엔드 한계
- UI 컴포넌트의 시각적·인터랙션 결과는 속성으로 표현 어려움.
- `fast-check-frontend`로 random user interaction 시퀀스 생성 가능하지만 신뢰도·디버깅 비용 ↑ → 도입 신중.

### Shrinking
실패 시 자동으로 최소 반례를 찾아준다 (e.g., 100원소 배열 실패 → 2원소까지 축소). 디버깅 비용 절감.

---

## 8. Error Guessing — 경험 기반 보완

### 정의
위 기법으로 잡히지 않는 엣지 케이스를 경험·직관·도메인 지식으로 추가.

### 프런트엔드 체크리스트
| 카테고리 | 케이스 |
|----------|--------|
| 빈 / 누락 | `""`, `null`, `undefined`, `[]`, `{}` |
| 0 / 음수 | `0`, `-0`, `-1`, `Number.NEGATIVE_INFINITY` |
| 매우 큰 수 | `Number.MAX_SAFE_INTEGER`, `Infinity`, `1e9` |
| 부동소수 | `0.1 + 0.2`, `NaN`, 매우 작은 값 |
| 문자열 길이 | 1자, 매우 긴 문자열(10MB), 멀티바이트 1자 |
| 인코딩 | 이모지(`👨‍👩‍👧`), RTL(`עברית`), 0폭 문자, 합자 |
| 시간 | 윤초, 윤년 2/29, DST 전후, timezone 경계, epoch 0 |
| 네트워크 | timeout, 5xx, 4xx, 부분 응답, slow 3G |
| 브라우저 | 뒤로가기, 새로고침, 다중 탭, 오프라인, 화면 잠금 |
| 동시성 | 빠른 연속 클릭, 폼 제출 중 unmount, race condition |

### 사용 시점
- 위 기법 6종을 적용한 후 마지막 검토 단계.
- "이 기능에서 사용자가 실수할 만한 게 뭘까?" 질문.

---

## 9. 종합 — 기능별 기법 매핑

| 기능 유형 | 1순위 | 보조 |
|-----------|-------|------|
| 폼 입력 검증 (단일 필드) | ECP + BVA | Error Guessing |
| 폼 다중 검증 (필드 간 의존) | Decision Table | ECP/BVA |
| 다단계 폼 / wizard | State Transition | Use Case (전체 흐름) |
| 로그인·회원가입·결제 | Use Case → State Transition + Decision Table | Error Guessing |
| 필터·정렬·옵션 화면 | Pairwise | ECP |
| 정렬·계산·파싱 유틸 | Property-Based | ECP/BVA |
| 모달·드로어·토스트 | State Transition | Error Guessing |
| 페이지네이션·무한스크롤 | BVA | State Transition |
| 권한·역할 분기 | Decision Table | — |

---

## 10. 안티패턴

### 10.1 "행복한 경로"만 테스트
Main Flow만 작성하고 Alternative·Exception 누락. 실제 결함의 다수가 비정상 흐름에서 발생.
→ Use Case 작성 시 alternative/exception 수를 main의 1.5~2배로 가져갈 것.

### 10.2 입력값을 "그럴듯한 1개"로만
`age = 30`으로 폼 검증 테스트 통과 → 14, 13, -1 등 boundary 미검증.
→ ECP로 클래스를 명시하고 BVA로 경계까지 가져갈 것.

### 10.3 Decision Table 없이 다중 조건 즉흥 작성
조건 3개 = 8 케이스인데 5~6개만 작성 후 "충분"으로 간주.
→ 표를 먼저 그리고, "—"(N/A) 명시한 후 케이스 추출.

### 10.4 상태 흐름을 "단방향"으로만
Step1 → Step2 → Step3만 검증. prev / cancel / 새로고침 누락.
→ 모든 화살표를 표로 나열 후 검증.

### 10.5 Property-Based의 만능 사용
UI 인터랙션까지 속성 기반으로 시도 → 디버깅 지옥.
→ 순수 함수에 한정.

### 10.6 Error Guessing부터 시작
체크리스트만 보고 케이스를 채워서 핵심 흐름·경계를 누락.
→ 1~6번을 먼저 적용한 후 마지막 보완으로만 사용.

---

## 11. 작성 절차 체크리스트

새 기능 테스트 작성 시:

- [ ] **Use Case 식별**: actor / precondition / main / alternative / exception / postcondition을 채웠는가?
- [ ] **입력 분석**: 각 입력에 대해 ECP 클래스를 정의했는가? BVA 경계까지 추가했는가?
- [ ] **다중 조건**: 조건이 2개 이상이면 Decision Table을 그렸는가?
- [ ] **상태**: 흐름에 상태가 있다면 전이표를 채웠는가? 무효 전이도 포함했는가?
- [ ] **조합 폭발**: 변수 4개 이상이면 Pairwise로 줄였는가?
- [ ] **불변식**: 순수 함수라면 Property-Based 1~2개를 추가했는가?
- [ ] **엣지**: Error Guessing 체크리스트로 마지막 검토했는가?
- [ ] **매핑**: 도출된 시나리오를 어떤 테스트 종류(단위/통합/E2E)로 작성할지 결정했는가? (`@.claude/rules/test-selection.md`)

---

## 교차 참조

- `@.claude/rules/test-selection.md` — 도출된 시나리오를 어떤 테스트 종류로 작성할지.
- `@.claude/rules/writing-rules.md` — 도출된 시나리오를 코드로 옮기는 작성 원칙.
- `@.claude/rules/test-data-strategy.md` — 시나리오의 입력 데이터를 어떻게 만들지.
- `@references/mocking-patterns.md` — 시나리오의 외부 의존을 어떻게 모킹할지.
