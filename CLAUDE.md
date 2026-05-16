# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 저장소 개요

Vue.js 3 core 모노레포입니다. pnpm 워크스페이스로 관리되며 `packages/`에 공개 패키지, `packages-private/`에 내부 도구(SFC playground, template explorer, dts 테스트 등)가 있습니다. 패키지 매니저는 `pnpm`으로 고정되어 있고(`preinstall`에 `only-allow pnpm`), Node 버전은 `.node-version`을 따릅니다.

## 자주 쓰는 명령어

빌드 / 개발:
- `pnpm build [패키지명...]` — Rollup 프로덕션 빌드. 패키지 이름은 fuzzy match (`pnpm build runtime --all` 등). **타입 체크는 하지 않음** — 별도로 `pnpm check` 실행 필요.
- `pnpm build [패키지명] -f <format>` — 포맷 지정 (`global`, `esm-bundler`, `esm-browser`, `cjs`, 그리고 `vue` 패키지 전용 `*-runtime`). 쉼표로 다중 포맷 지정 가능.
- `pnpm build [패키지명] -s` — 소스맵 포함 빌드 (느림).
- `pnpm dev [패키지명]` — 단일 패키지 watch 모드 (기본 `vue`, `global` 포맷). fuzzy match 미지원, 전체 이름 필요.
- `pnpm dev-sfc` — 로컬 SFC Playground. 재현 가능한 버그 디버깅에 가장 빠른 피드백 루프.
- `pnpm dev-compiler` — Template Explorer를 `http://localhost:3000`에 서빙. 컴파일러 출력 디버깅용.
- `pnpm build-dts` — 모든 `.d.ts` 파일 생성 후 rollup-plugin-dts로 패키지당 하나의 `.d.ts`로 롤업.

검증:
- `pnpm check` — 전체 코드베이스에 대해 `tsc --noEmit` (incremental). pre-commit 훅에서 자동 실행됨.
- `pnpm lint` — ESLint (cache 사용).
- `pnpm format` / `pnpm format-check` — Prettier.

테스트 (Vitest, projects로 분리):
- `pnpm test` — 모든 프로젝트 watch 모드.
- `pnpm test-unit` — `unit*` 프로젝트(소스 코드 대상 단위 테스트)만 실행.
- `pnpm test-e2e` — `vue` global 빌드를 만든 뒤 `e2e` 프로젝트(`packages/vue/__tests__/e2e/*.spec.ts`) 실행.
- `pnpm test run` — watch 없이 1회 실행 (`vitest run`과 동등).
- `pnpm test <패키지명>` 또는 `pnpm test <파일패턴>` — 패키지/파일 필터.
- `pnpm test <파일패턴> -t '<테스트명>'` — 특정 테스트만.
- `pnpm test-coverage` — `unit*` 프로젝트에 대해 v8 커버리지.
- `pnpm test-dts` — `build-dts` 후 `packages-private/dts-test`의 타입 테스트 실행. dts가 이미 빌드되어 있으면 `pnpm test-dts-only`로 재실행 가능.
- `pnpm bench` / `pnpm bench-compare` — Vitest 벤치마크 (결과는 `temp/bench.json`).

릴리스 / 기타:
- `pnpm release` — `scripts/release.js`로 릴리스 수행.
- `pnpm clean` — `packages/*/dist`, `temp`, `.eslintcache` 제거.

> Note: 기여 가이드는 `nr` (from `@antfu/ni`)를 권장하지만 `pnpm <script>`도 동일하게 동작합니다.

## 아키텍처

### 패키지 그래프 (반드시 지켜야 할 import 규칙)

```
runtime-dom → runtime-core → reactivity
compiler-sfc → { compiler-core, compiler-dom }
compiler-dom → compiler-core
vue → { compiler-dom, runtime-dom }
```

- **다른 패키지에서 항목을 가져올 땐 절대 상대 경로 금지.** 소스 패키지에서 export 한 뒤 패키지명(`@vue/...`)으로 import. 이 alias는 `tsconfig.json`의 `compilerOptions.paths`, `scripts/aliases.js`(Rollup/Vitest 공용), pnpm workspaces로 동시에 구성되어 있습니다.
- **compiler ↔ runtime 패키지는 서로 import 하지 않습니다.** 공통이 필요하면 `@vue/shared`로 추출.
- `@vue/shared` 내 일부 헬퍼(예: `isHTMLTag`, `isSVGTag`)는 **compiler 전용**이므로 runtime 코드에서 쓰지 말 것 — 트리 쉐이킹이 깨져 런타임 번들에 dev/compiler 전용 코드가 포함될 수 있습니다.
- 패키지 A가 패키지 B로부터 값을 import 하거나 타입을 re-export 하면, B는 A의 `package.json` `dependencies`에 명시되어야 합니다 (ESM-bundler/CJS 빌드에서 externalize 되기 때문).

### 빌드 시 정의되는 전역 컴파일 플래그

런타임 코드는 빌드 시점 상수로 분기됩니다. 모든 dev-only 코드는 반드시 `__DEV__` 분기 안에 둬서 프로덕션 빌드에서 제거되도록 해야 합니다. Vitest 환경 기본값(`vitest.config.ts`):

- `__DEV__`, `__TEST__`, `__ESM_BUNDLER__`, `__CJS__`, `__SSR__`, `__FEATURE_OPTIONS_API__`, `__FEATURE_SUSPENSE__`, `__COMPAT__` → `true`
- `__BROWSER__`, `__GLOBAL__`, `__ESM_BROWSER__`, `__FEATURE_PROD_DEVTOOLS__`, `__FEATURE_PROD_HYDRATION_MISMATCH_DETAILS__` → `false`

런타임 사이즈에 민감하므로(컴파일러 코드는 덜) 핫패스(컴포넌트 업데이트, vdom patch 등)에서 변경 시 번들 크기와 perf 영향을 의식해야 합니다.

### Vitest 프로젝트 구조

`vitest.config.ts`에 네 개의 프로젝트가 정의되어 있습니다:
- `unit` — Node 환경 단위 테스트. `vue`, `vue-compat`, `runtime-dom`, e2e, `ssrWatch` 는 **제외**됩니다.
- `unit-gc` — `--expose-gc`가 필요한 `ssrWatch.spec.ts` 전용 (`forks` pool).
- `unit-jsdom` — `vue`/`vue-compat`/`runtime-dom` 패키지의 테스트는 jsdom 환경에서 실행.
- `e2e` — `packages/vue/__tests__/e2e/*.spec.ts`, jsdom, 격리 실행. 실행 전 `vue` global 빌드 필요(`pnpm test-e2e`가 자동으로 수행).

플랫폼 비종속 / 저수준 vdom 동작을 테스트할 때는 `@vue/runtime-test`를 사용하세요 — 플랫폼별 런타임은 플랫폼 고유 동작을 검증할 때만 씁니다.

## 워크 브랜치 & 커밋

- **`main`** — 버그 픽스, chore 등.
- **`minor`** — 새 API surface를 추가하는 feature는 이 브랜치로.
- 커밋 메시지는 `.github/commit-convention.md`의 정규식 `/^(revert: )?(feat|fix|docs|dx|style|refactor|perf|test|workflow|build|ci|chore|types|wip)(\(.+\))?: .{1,50}/` 를 통과해야 합니다 (`scripts/verify-commit.js`가 commit-msg 훅에서 강제). 버그 픽스는 PR 제목에 `(fix #xxxx)`를 포함하면 changelog에 자동 링크됩니다.

## Git Hooks (simple-git-hooks)

- `pre-commit`: `pnpm lint-staged` (변경된 `.js/.json`은 Prettier, `.ts(x)`는 ESLint --fix + Prettier) → `pnpm check` (전체 타입 체크).
- `commit-msg`: `scripts/verify-commit.js`로 메시지 규칙 검증.
