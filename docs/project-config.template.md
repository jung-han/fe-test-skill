# 프로젝트 테스트 설정

> **이 파일은 템플릿이다.** 스킬 최초 실행 시 사용자 프로젝트 루트에 `project-config.md`로 복사되어 생성된다.
> 생성 후 스킬이 `package.json`, `vitest.config.*` 등을 읽어 빈 값을 자동으로 채운다.
> 스킬은 항상 사용자 프로젝트의 `project-config.md`를 먼저 읽고, 없으면 이 템플릿 기준으로 탐지한다.

---

## 테스트 러너

- **러너**: (vitest / jest)
- **설정 파일 경로**: (예: `vite.config.ts`, `vitest.config.ts`)
- **globals**: (yes / no) — yes면 `vi`, `describe`, `it`, `expect` 등 import 불필요
- **environment**: (jsdom / happy-dom / node)
- **setupFiles 경로**: (예: `src/setupTests.ts`)

## jest-dom

- **사용 여부**: (yes / no)
- **등록 방식**: (setupFiles에서 import / 자동)

## MSW

- **사용 여부**: (yes / no)
- **버전**: (v1 / v2)
- **handlers 경로**: (예: `src/__mocks__/handlers.ts`)
- **server 경로**: (예: `src/__mocks__/server.ts`)

## 커스텀 setup 함수

- **사용 여부**: (yes / no)
- **경로**: (예: `src/utils/test/setup.tsx`)
- **함수명**: (예: `setup`)
- **포함된 Provider**: (예: ThemeProvider, SnackbarProvider)
- **userEvent 포함 여부**: (yes / no) — yes면 반환값에서 `user` 꺼내 쓴다

## 테스트 파일 구조

- **단위/통합 테스트 위치**: (예: 소스 파일 옆 `*.test.tsx` / `src/__tests__/`)
- **E2E 테스트 위치**: (예: `e2e/`)
- **파일 네이밍**: (예: `*.test.tsx` / `*.spec.tsx`)

## E2E (Playwright)

- **사용 여부**: (yes / no)
- **설정 파일 경로**: (예: `playwright.config.ts`)

## 시각적 회귀

- **사용 여부**: (yes / no)
- **방식**: (vitest-browser / playwright-screenshot / 없음)
