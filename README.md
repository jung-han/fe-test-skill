# fe-test-skill

React 프로젝트용 프론트엔드 테스트 Claude Code 플러그인.

테스트 작성과 검수, 두 가지 스킬을 제공한다.

---

## 스킬

| 스킬 | 호출 | 설명 |
|------|------|------|
| `write-test` | `테스트 써줘`, `/write-test` | 코드 분석 → 유형 선택 → 테스트 작성 |
| `review-test` | `테스트 검토해줘`, `/review-test` | 기존 테스트 → 모범사례 기준 개선점 제시 |

---

## 설치

```bash
# npx skills (준비 중)
npx skills install fe-test-skill

# 또는 로컬 설치
# ~/.claude/settings.json의 plugins에 경로 추가
```

---

## 초기 설정

스킬 최초 실행 시 프로젝트 루트에 `project-config.md`가 자동 생성된다.
`package.json`, `vitest.config.*` 등을 읽어 값을 채운다.

직접 채우고 싶다면 `docs/project-config.template.md`를 참고해 프로젝트 루트에 `project-config.md`를 만든다.

---

## 사용 방법

### 피처 개발 후 테스트 작성

```
# 방법 1: 파일 지정
Cart.tsx 테스트 써줘

# 방법 2: 파일 미지정 (최근 수정 파일 자동 탐지)
테스트 써줘

# 방법 3: 범위 지정
useCart 훅 단위 테스트 써줘
```

스킬이 하는 것:
1. 대상 코드와 관련 파일(훅, 유틸 등) 읽기
2. 테스트 유형 결정 + 이유 제시 (단위 / 통합 / E2E / 시각적 회귀)
3. 프로젝트 설정(globals, setup 함수 등)에 맞게 테스트 작성

예시 흐름:
```
> Cart.tsx 테스트 써줘

[스킬]: Cart.tsx + useCartStore.ts 읽음.
여러 컴포넌트 조합 + API 연동 있음 → 통합 테스트 선택.
단위 테스트도 가능하지만 비즈니스 로직 검증에는 통합이 더 적합.

[테스트 코드 작성...]
```

---

### 기존 테스트 검수

```
# 파일 지정
Cart.test.tsx 검토해줘

# 디렉토리 전체
src/components/ 테스트 전체 검토해줘
```

출력 예시:
```
Cart.test.tsx:L12: 🔴 fireEvent.change 사용. userEvent.type으로 교체한다.
Cart.test.tsx:L34: 🟡 waitFor 밖에서 assertion. waitFor 안으로 이동한다.
Cart.test.tsx:L67: 🔵 getByTestId 사용. getByRole('button', { name: /담기/i })로 교체한다.
```

---

## 지원 테스트 유형

| 유형 | 도구 | 선택 기준 |
|------|------|----------|
| 단위 | Vitest + RTL | 독립적 컴포넌트, 커스텀 훅, 유틸 |
| 통합 | Vitest + RTL + MSW | 비즈니스 로직, 컴포넌트 조합, API 연동 |
| E2E | Playwright | 전체 워크플로우, 인증/리다이렉트 |
| 시각적 회귀 | Vitest Browser Mode | CSS·레이아웃 변화 감지 |

유형 선택은 **유지보수 비용 최소화** 기준. 선택 이유는 항상 함께 제시.

---

## 요구사항

- React 프로젝트 (`package.json`에 `react` 의존성 필요)
- Vitest 또는 Jest 설치
- (E2E) Playwright 설치
- (시각적 회귀) Vitest Browser Mode 설정

환경 미구성 시 테스트 코드를 작성하지 않고 설정 방법을 안내한다.
