# fe-test-skill

React 프로젝트 전용 프론트엔드 테스트 Claude Code 플러그인.

## 스킬

- `write-test`: 코드/컨텍스트 분석 → 테스트 유형 선택 → 모범사례 기반 작성
- `review-test`: 기존 테스트 검토 → 모범사례 기반 개선점 제시

## 대상 프로젝트

React 기반만. `package.json`에 `react` 의존성 없으면 스킬 발동 안 함.

## 기술 스택

| 역할 | 도구 |
|------|------|
| 테스트 러너 | Vitest, Jest |
| 컴포넌트/훅 테스트 | React Testing Library (RTL) |
| E2E | Playwright |
| API 모킹 | MSW |
| 시각적 회귀 | Vitest Browser Mode (`toMatchScreenshot`) |

> **Vitest Browser Mode**: jsdom 대신 실제 브라우저(Playwright provider)로 실행. `toMatchScreenshot()`으로 실제 픽셀 캡처. Storybook 8+와 통합 가능.

## 환경 미구성 시 동작

코드 작성 전 `package.json`, `vitest.config.*`, `playwright.config.*` 확인.
미구성 감지 시 → 코드 작성 안 함. 설정 방법만 안내.

## context7 MCP

사용 가능한 경우: 프로젝트의 라이브러리 버전 확인 → context7로 해당 버전 공식 문서 조회 → 버전별 API 반영.
없으면 생략하고 docs/ 파일 기준으로 작성.

---

## 스킬 실행 흐름

### write-test
1. 사용자 프로젝트 루트에서 `project-config.md` 탐지
2. 없으면 `package.json`, `vitest.config.*` 등 읽어서 생성
3. `test-selection.md` 기준으로 테스트 유형 결정
4. 유형에 맞는 docs 참조해서 작성

### review-test
1. `project-config.md` 탐지 (위와 동일)
2. `review-checklist.md`의 검수 순서대로 평가
3. 각 관점에서 참조 문서의 ❌ 패턴 기준으로 문제 수집
4. 출력 포맷에 맞춰 결과 제시

---

## 참고 문서 (docs/)

| 파일 | 역할 | 사용 시점 |
|------|------|----------|
| `project-config.template.md` | 사용자 프로젝트에 생성할 config 템플릿 | 스킬 최초 실행 시 |
| `test-selection.md` | 테스트 유형 선택 기준 (결정 트리, 장단점, 전략) | 유형 판단 시 |
| `review-checklist.md` | 검수 관점 및 출력 포맷 | review-test 실행 시 |
| `writing-rules.md` | 핵심 4원칙, AAA 패턴, Vitest API | 작성/검수 공통 |
| `rtl-patterns.md` | RTL 쿼리, userEvent, waitFor, 안티패턴 | 단위/통합 테스트 |
| `mocking-patterns.md` | vi.fn/spyOn, 모듈 모킹, 타이머, MSW | 모킹 필요 시 |
| `e2e-best-practices.md` | Playwright 패턴, API 모킹 원칙 | E2E 테스트 |
| `visual-regression.md` | Vitest Browser Mode, 스냅샷 관리 | 시각적 회귀 테스트 |

---

## 파일 구조

```
fe-test-skill/
├── CLAUDE.md
├── skills/
│   ├── write-test/SKILL.md        ← (Phase 2)
│   └── review-test/SKILL.md       ← (Phase 2)
└── docs/
    ├── project-config.template.md
    ├── test-selection.md
    ├── review-checklist.md
    ├── writing-rules.md
    ├── rtl-patterns.md
    ├── mocking-patterns.md
    ├── e2e-best-practices.md
    └── visual-regression.md
```

---

## 작업 체크리스트

### Phase 1: 자료 준비 ✅
- [x] 강의 자료 기반 규칙 추출
- [x] Kent C. Dodds RTL 모범사례 보강
- [x] 시각적 회귀 테스트 최신 정보 조사 (Vitest Browser Mode + 스냅샷 관리)
- [x] 실제 프로젝트 설정 반영 (vitest globals, setupTests, setup 함수 패턴)
- [x] docs/ 구조 정리 (SSOT 원칙, 선택 기준 단일화, 검수 관점 분리)

### Phase 2: 스킬 작성 ✅
- [x] `skills/write-test/SKILL.md` 작성
- [x] `skills/review-test/SKILL.md` 작성

### Phase 3: 검증
- [ ] `.claude/settings.json`에 로컬 플러그인 등록
- [ ] 평가 방식 확정 (아래 참고)
- [ ] 샘플 React 프로젝트로 write-test 실행 — 단위/통합/E2E 각 1케이스
- [ ] 샘플 테스트 파일로 review-test 실행 — 의도적으로 심은 안티패턴 감지 확인
- [ ] project-config.md 자동 생성 흐름 확인
- [ ] 결과물 검토 및 docs/ 규칙 보완

#### 평가 방식 (검토 필요)
스킬 품질을 어떻게 측정할 것인가:
- write-test: 작성된 테스트가 실제로 통과하는가 / 모범사례를 지키는가 / 유형 선택이 적절한가
- review-test: 의도적으로 심은 안티패턴을 빠짐없이 잡는가 / 오탐(false positive)이 없는가
- 평가용 샘플 코드 + 기대 출력을 미리 정의해두는 방식 고려

### Phase 4: 배포 (옵션)
- [ ] npx skills 배포 준비

---

## 결정 사항

| 날짜 | 결정 | 이유 |
|------|------|------|
| 2026-04-26 | 스킬 2개 구조 (Agent 아님) | 인터랙티브 작업, 파이프라인 불필요 |
| 2026-04-26 | caveman 플러그인 구조 참조 | 검증된 SKILL.md 패턴 |
| 2026-04-26 | Kent C. Dodds 블로그 보강 | 강의 자료의 안티패턴 공백 보완 |
| 2026-04-26 | 시각적 회귀: Vitest Browser Mode 중심 | 무료, 기존 스택 확장, 실제 픽셀 캡처 |
| 2026-04-26 | project-config를 사용자 프로젝트에 생성 | 스킬 패키지 안에 고정되면 안 됨 |
| 2026-04-26 | review-checklist는 관점만 정의 | SSOT — 규칙 내용은 각 docs가 담당 |

## 참고 링크

- caveman 플러그인: `~/.claude/plugins/cache/caveman/caveman/63e797cd753b/`
- Kent C. Dodds RTL: https://kentcdodds.com/blog/common-mistakes-with-react-testing-library
- Kent C. Dodds 테스트 전략: https://kentcdodds.com/blog/write-tests
- mock 초기화 차이: https://medium.com/@yujso66/번역-당신의-jest-테스트는-잘못되어-있을-수도-있습니다-866f5f982ff9
