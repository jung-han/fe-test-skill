# 테스트 검수 관점

테스트를 검수할 때 이 문서에 명시된 관점으로 각 참조 문서를 읽고 평가한다.
규칙 내용은 각 문서가 SSOT — 이 문서는 "무엇을 어떤 눈으로 볼 것인가"만 정의한다.

---

## 검수 순서

### 1. 프로젝트 설정 파악
`project-config.md` (사용자 프로젝트 루트)를 읽어 테스트 환경을 파악한다.
없으면 `package.json`, `vitest.config.*`를 직접 읽어 파악한다.

---

### 2. 테스트 유형이 적합한가
`@.claude/rules/test-selection.md`의 결정 트리와 장단점 기준으로 판단한다.

확인할 것:
- 지금 작성된 테스트가 올바른 유형(단위/통합/E2E/시각적 회귀)인가
- 통합 테스트인데 모킹이 지나치게 많아 E2E가 더 적합하진 않은가
- 단위 테스트로 작성되어 있지만 실제로는 여러 모듈 조합을 검증하고 있진 않은가

---

### 3. 시나리오 도출이 충분한가
`@references/scenario-design.md`의 7가지 black-box 도출 기법 기준으로 평가한다.

확인할 것:
- **Use Case**: main flow만 검증하고 alternative / exception flow가 누락되진 않았는가
- **ECP / BVA**: 폼/입력 검증에서 boundary(min, min±1, max, max±1)가 빠지진 않았는가
- **Decision Table**: 다중 조건(조건 2개 이상)이 즉흥 작성되어 일부 조합만 검증되고 있진 않은가
- **State Transition**: 상태 흐름의 prev / cancel / 새로고침 / 무효 전이가 누락되진 않았는가
- **Pairwise**: 변수 4개 이상의 옵션 조합을 모두 또는 일부만 자의적으로 골라 작성하진 않았는가
- **Error Guessing**: null / undefined / 빈 문자열 / 멀티바이트 / 매우 큰 수 등 흔한 엣지가 빠지진 않았는가

심각도 기준:
- 핵심 비즈니스 로직(결제, 인증)에서 exception flow 누락 → **Critical**
- 입력 검증의 boundary 누락 → **Major**
- Error Guessing 체크리스트 일부 미반영 → **Minor**

---

### 4. 작성 원칙을 지키고 있는가
`@.claude/rules/writing-rules.md`의 핵심 4원칙 기준으로 각 테스트를 평가한다.

확인할 것:
- 내부 구현(state, private 메서드)을 직접 테스트하고 있진 않은가 (원칙 1)
- 의미 없는 테스트(커버리지만을 위한 trivial 케이스)가 있진 않은가 (원칙 2)
- 테스트 디스크립션이 불명확해서 무엇을 검증하는지 알기 어렵진 않은가 (원칙 3)
- 하나의 테스트에서 여러 동작을 검증하고 있진 않은가 (원칙 4)

---

### 5. RTL 안티패턴이 있는가
`@references/rtl-patterns.md`에서 ❌로 표시된 패턴이 코드에 존재하는지 확인한다.

---

### 6. 모킹이 적절한가
`@references/mocking-patterns.md`의 모킹 원칙과 초기화 패턴 기준으로 평가한다.

---

### 7. 데이터 전략을 따르고 있는가
`@.claude/rules/test-data-strategy.md`의 원칙 기준으로 평가한다.

---

### 8. E2E 테스트인 경우
`@references/e2e-best-practices.md`의 패턴 기준으로 평가한다.

---

### 9. 프로젝트 관례를 따르고 있는가
`project-config.md`에 커스텀 setup 함수가 정의되어 있는데 사용하지 않고 있진 않은가.

---

## 출력 포맷

```
파일경로:L줄번호: 문제. 수정 방향.
```

심각도 prefix (혼재 시):
- `Critical` — 테스트가 아무것도 검증하지 않거나 잘못된 결과를 신뢰하게 만드는 패턴
- `Major` — 유지보수 비용을 크게 높이거나 리팩터링 시 깨질 가능성이 높은 패턴
- `Minor` — 가독성·일관성·관례를 해치는 패턴

한 줄로 작성한다. 문제와 수정 방향이 명확하면 이유는 생략한다.
