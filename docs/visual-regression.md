# 시각적 회귀 테스트

## 구조 이해

```
Vitest (테스트 러너)
  └─ Browser Mode
       └─ Playwright (브라우저 provider — 실제 Chromium 구동)
            └─ toMatchScreenshot() → 실제 픽셀 이미지 캡처 + 비교
```

Storybook이 있는 프로젝트라면 Story를 Vitest에서 직접 실행 가능:

```
Storybook (컴포넌트 격리 환경)
  └─ @storybook/experimental-addon-test (Storybook 8+ Vitest plugin)
       └─ Story → Vitest Browser Mode에서 실행
            └─ toMatchScreenshot() 동일하게 사용
```

---

## 스냅샷 유형

- **DOM 스냅샷** (`toMatchSnapshot`) — HTML 직렬화. CSS 변화 감지 불가.
- **이미지 스냅샷** (`toMatchScreenshot`) — 실제 픽셀 비교. CSS·레이아웃 변화 감지.

픽셀 비교는 브라우저/OS 따라 결과 다를 수 있음 → 허용 오차(threshold) 설정 필수.

## 도구 선택지

| 방식 | 비용 | 컴포넌트 단위 | 페이지 단위 | 비고 |
|------|------|--------------|------------|------|
| **Vitest Browser Mode** | 무료 | ✅ | ❌ | 기존 Vitest 스택 확장 |
| **Vitest + Storybook** | 무료 | ✅ | ❌ | Story 재사용, Storybook 8+ 필요 |
| **Playwright** (`toHaveScreenshot`) | 무료 | △ | ✅ | E2E와 시각 검증 통합 |
| **reg-suit** | 무료 (스토리지 필요) | ✅ | ✅ | PR diff 리포트, 팀 협업용 |
| **Storybook + Chromatic** | 유료 | ✅ | ❌ | 무료 tier 스냅샷 수 제한 |

---

## 권장 조합

### 기본: Vitest Browser Mode (컴포넌트 단위)

추가 도구 없이 기존 Vitest 스택에서 바로 사용.

```ts
// vitest.config.ts
export default defineConfig({
  test: {
    browser: {
      enabled: true,
      provider: 'playwright',
      instances: [{ browser: 'chromium' }],
    },
  },
});
```

```ts
import { page } from 'vitest/browser'

test('Button 시각 스냅샷', async () => {
  render(<Button variant="primary">제출</Button>)
  const button = page.getByRole('button')
  await expect(button).toMatchScreenshot('primary-button')
})
```

스냅샷 업데이트:
```bash
vitest --update  # 또는 watch 모드에서 u 키
```

**주의**: `browser.enabled: true` 없으면 동작 안 함. jsdom 테스트와 별도 project 구성 권장.

---

### Storybook 있는 프로젝트: Vitest + Storybook 통합

```ts
// vitest.config.ts — Storybook 8+ addon-test 사용 시
import { storybookTest } from '@storybook/experimental-addon-test/vitest-plugin'

export default defineConfig({
  plugins: [storybookTest()],
  test: {
    browser: {
      enabled: true,
      provider: 'playwright',
      instances: [{ browser: 'chromium' }],
    },
  },
})
```

Story 파일에서 바로 시각 테스트:
```ts
// Button.stories.ts
export const Primary: Story = {
  play: async ({ canvasElement }) => {
    await expect(canvasElement).toMatchScreenshot('button-primary')
  },
}
```

---

### 페이지/플로우 단위: Playwright `toHaveScreenshot`

E2E와 시각 검증을 한 번에.

```ts
test('홈 페이지 시각 스냅샷', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveScreenshot('home.png', {
    maxDiffPixelRatio: 0.02,
  });
});
```

스냅샷 업데이트:
```bash
npx playwright test --update-snapshots
```

---

### 팀 협업 + PR 리뷰: reg-suit

스냅샷을 S3/GCS에 저장, PR마다 HTML 비교 리포트 생성.

```bash
npm i -D reg-suit
npx reg-suit init
```

흐름: 스크린샷 생성 → reg-suit 비교 → HTML 리포트 → PR 코멘트

---

## DOM 스냅샷 vs 이미지 스냅샷

| | DOM 스냅샷 | 이미지 스냅샷 |
|--|-----------|--------------|
| 방법 | `toMatchSnapshot()` | `toMatchScreenshot()` / `toHaveScreenshot()` |
| 속도 | 빠름 | 느림 (브라우저 렌더링 필요) |
| CSS 변화 감지 | ❌ 안 됨 | ✅ 됨 |
| flaky 위험 | 낮음 | 있음 (threshold 설정으로 완화) |
| 실제 시각 변화 | 간접적 | 직접적 |

→ CSS·레이아웃 변경 감지가 목적이면 이미지 스냅샷.
→ 구조 변경 감지가 목적이면 DOM 스냅샷으로 충분.

---

## 스냅샷 관리 모범사례

### Git에 커밋하나?

**Yes — 기준 이미지는 반드시 커밋.** 변경 추적과 PR 리뷰를 위해 필요.
대규모 프로젝트는 저장소 비대화 방지를 위해 **Git LFS** 사용 권장.

### 핵심 문제: OS/브라우저 렌더링 차이

로컬(Mac) vs CI(Linux) 환경이 다르면 **같은 코드도 다른 픽셀** 나옴 → 테스트 실패.

> **팀 프로젝트 권장**: 기준 이미지는 Docker(Linux) 환경에서만 생성.
> 플랫폼별 이미지 분리(darwin/linux 2세트)는 관리 부담이 크고 팀 협업에 적합하지 않음.

```bash
# 로컬에서도 Docker로 기준 이미지 생성/업데이트
docker run --rm -v $(pwd):/app mcr.microsoft.com/playwright:latest \
  npx playwright test --update-snapshots
```

단일 Linux 기준 이미지만 유지 → CI와 동일 환경 보장.
단, 프로젝트 규모나 팀 상황에 따라 다를 수 있으므로 **강제 룰이 아닌 권장 방향으로 제시**.

### 디렉토리 구조

```
src/components/Button/
├── Button.tsx
├── Button.test.tsx
└── __screenshots__/
    ├── button-primary-chromium-linux.png    # 기준 이미지 (커밋)
    └── button-primary-chromium-darwin.png   # 기준 이미지 (커밋)

e2e/
├── home.spec.ts
└── home.spec.ts-snapshots/
    ├── home-chromium-linux.png              # 기준 이미지 (커밋)
    └── home-chromium-darwin.png             # 기준 이미지 (커밋)
```

### .gitignore 패턴

기준 이미지는 커밋, 테스트 실패 시 생성되는 diff/actual은 무시:

```gitignore
# 테스트 실패 임시 산출물 — 무시
**/test-results/
**/*-snapshots/*-diff.png
**/*-snapshots/*-actual.png
__screenshots__/*-diff.png
__screenshots__/*-actual.png
```

### 업데이트 워크플로우

**CI에서 자동 업데이트 절대 금지.** 기준 이미지 변경 = 코드 변경과 동일하게 취급.

```
1. 로컬에서 의도한 UI 변경 확인
2. --update-snapshots 로 기준 이미지 갱신
3. diff 직접 눈으로 확인
4. 변경된 이미지 파일과 함께 PR 제출
5. 팀 리뷰 후 머지
```

```bash
# 로컬에서만
vitest --update
npx playwright test --update-snapshots

# CI는 항상 비교만 (업데이트 없음)
vitest
npx playwright test
```

### CI 설정 원칙

- 시각적 회귀 테스트는 단위/통합 테스트와 **별도 job**으로 분리 (속도 차이 큼)
- 매 커밋이 아닌 PR 단위 또는 수동 트리거로 실행
- 동적 콘텐츠(날짜, 사용자 데이터) 마스킹 또는 비활성화
- 뷰포트 크기 고정 (예: 1280×720)

---

## flaky 방지 패턴

```ts
// 1. 허용 오차 설정
await expect(page).toHaveScreenshot({ maxDiffPixelRatio: 0.02 });

// 2. 애니메이션 비활성화
await page.addStyleTag({ content: '* { animation: none !important; transition: none !important; }' });

// 3. 폰트 로딩 완료 후 캡처
await page.waitForLoadState('networkidle');

// 4. 동적 콘텐츠 마스킹 (날짜, 광고 등)
await expect(page).toHaveScreenshot({ mask: [page.locator('.date')] });
```

---

## 환경 미구성 시

Vitest Browser Mode 또는 Playwright 시각적 스냅샷 미설정 시 코드 작성 안 함.
설정 방법만 안내.

**확인 기준**:
- Vitest: `vitest.config.*`에 `browser.enabled: true` 여부
- Playwright: `playwright.config.*` 존재 + `toHaveScreenshot` 사용 이력
