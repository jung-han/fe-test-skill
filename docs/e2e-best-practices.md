# E2E 테스트 모범사례 (Playwright)

## API 모킹 원칙

기본: **API 모킹 안 함.** 실제 서버와 연동.
- 테스트 전용 DB 또는 검증용 계정 별도 구성
- 서버 부하 방지를 위해 유관부서(백엔드, QA) 협의 필수

예외적으로 모킹 허용:
- 실패 케이스 (결제 실패, 서버 에러 등) — 실제 재현 어려움
- 백엔드 협의가 불가능한 외부 서비스

```ts
// 특정 테스트에서만 실패 케이스 모킹
test('결제 실패 시 에러 메시지 표시', async ({ page }) => {
  await page.route('/api/payment', route =>
    route.fulfill({ status: 500, body: JSON.stringify({ message: '결제 실패' }) })
  );
  // ...
});
```

---

## Playwright 패턴

### 기본 구조

```ts
import { test, expect } from '@playwright/test';

test.describe('기능 또는 페이지 단위 그룹', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/');
  });

  test('시나리오 설명', async ({ page }) => {
    // 액션
    await page.getByRole('button', { name: '로그인' }).click();
    await page.getByLabel('이메일').fill('user@example.com');
    await page.getByLabel('비밀번호').fill('password');
    await page.getByRole('button', { name: '확인' }).click();

    // 검증
    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByText('환영합니다')).toBeVisible();
  });
});
```

### 인증 상태 재사용 (storageState)

로그인이 필요한 테스트마다 로그인 반복 금지.
`storageState`로 인증 세션 저장 후 재사용:

```ts
// playwright.config.ts
export default defineConfig({
  projects: [
    {
      name: 'authenticated',
      use: { storageState: 'playwright/.auth/user.json' },
      dependencies: ['setup'],
    },
  ],
});

// auth.setup.ts
import { test as setup } from '@playwright/test';
setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('이메일').fill('user@example.com');
  await page.getByLabel('비밀번호').fill('password');
  await page.getByRole('button', { name: '확인' }).click();
  await page.context().storageState({ path: 'playwright/.auth/user.json' });
});
```

### 로케이터 선택 우선순위

RTL 쿼리 원칙과 동일 — 접근성 기반 우선:

```ts
page.getByRole('button', { name: /제출/i })  // ✅ 최우선
page.getByLabel('이메일')                     // ✅
page.getByPlaceholder('이메일을 입력하세요')   // ✅
page.getByText('확인')                        // ✅
page.getByTestId('submit-btn')               // ⚠️ 마지막 수단
page.locator('.submit-button')               // ❌ CSS 셀렉터 지양
```

### 비동기 대기

Playwright는 기본적으로 auto-waiting 내장 — 명시적 `sleep` 금지.

```ts
// ❌
await page.waitForTimeout(1000);

// ✅ 조건 충족까지 대기
await expect(page.getByText('로딩 완료')).toBeVisible();
await page.waitForURL('/dashboard');
await page.waitForResponse('/api/user');
```

---

## 환경 확인 (실행 전)

`playwright.config.ts` 또는 `playwright.config.js` 존재 여부 확인.
미구성 시 코드 작성 안 함 — 설치/설정 방법만 안내.

```bash
# 미구성 프로젝트 초기 설정 안내용
npm init playwright@latest
```
