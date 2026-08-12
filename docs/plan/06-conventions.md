# 06 — Conventions

> One way to do each thing, checkable by a machine. Where a convention cannot be linted, it must be produced by a generator (`tools/generators`) so it is followed by construction. Manual code review is the last line, not the enforcement mechanism.

## 1. Language & typing

| Rule | Detail |
|---|---|
| TypeScript everywhere | `strict: true`, `noUncheckedIndexedAccess: true`, `noImplicitOverride: true`. Base tsconfigs in `@console/config/tsconfig/{base,vue-app,vue-lib,node}.json`. |
| No `any` | `@typescript-eslint/no-explicit-any: error`; escape hatch is a typed `unknown` + narrowing. |
| Typecheck command | `pnpm typecheck` → `vue-tsc --noEmit` per workspace via turbo (cached). |
| zod | pinned to the **3.25.x** line (see `01-architecture.md` §4). Schemas are the single source for both validation and inferred types: `type X = z.infer<typeof xSchema>`. |

## 2. Files, naming, structure

| Rule | Detail |
|---|---|
| File/dir names | `kebab-case` for everything (matches webkit). Component files `kebab-case.vue`; their exported name is PascalCase. |
| Vue SFC shape | `<script setup lang="ts">` only. No Options API (`vue/component-api-style: ['error', ['script-setup']]`). Type-based `defineProps`/`defineEmits`. |
| Styling | Tailwind utility classes; `<style>` blocks are exceptional and must be justified in the PR (webkit ships zero style blocks — same bar). |
| Icons | font classes from `@aziontech/icons` (`ai-*`, `pi-*`) via component `icon` props. |
| Tests | **Colocated**: `foo.ts` + `foo.spec.ts` (or `__tests__/` for multi-file suites). The old console's parallel `src/tests/**` mirror tree is explicitly not ported — it drifted from the source it mirrors. |
| Copy | English, sentence case, actionable ("Certificate name is required", not "invalid input"). No i18n indirection (decision `01-architecture.md` §3.5). |

### App layout (`apps/console/app/`)

```
pages/            # THIN route shells: definePageMeta + mount one feature component (≤ ~15 lines)
features/<domain>/  # the real code: components/, composables/, schemas/, constants/, *.spec.ts
components/       # app-wide chrome only (navbar widgets, footer)
layouts/          # default, auth (public/minimal), fullscreen
middleware/       # NN.name.global.ts — numbered, ordered
plugins/          # *.client.ts when DOM-touching (toast, analytics, monaco)
stores/           # ONLY: session, preferences, chrome (see lint rule below)
utils/client/     # the only place browser globals are legal
config/env.ts     # the only reader of runtimeConfig
```

Pages exist for routing; features own logic. A page that grows a `<script>` beyond meta + composition wiring is a lint-flagged smell (`max-lines` on `app/pages/**`: 40).

### Package layout

- `packages/components/src/<block-name>/{<block-name>.vue, index.ts, <block-name>.spec.ts, <block-name>.stories.ts}` — component + export + test + story travel together; the generator creates all four.
- `services/` layout is specified in `03-services.md` (it is the disposal boundary, stricter rules apply).

## 3. Lint rule catalog (the conventions that are laws)

All in `@console/config/eslint`, flat config, ESLint 10. Every rule `error` — `warn` is where conventions go to die (discovery: the old repo's architecture ruleset is `warn` for all legacy paths and the debt calcified).

| Rule (or preset) | Enforces |
|---|---|
| `boundaries/element-types` + `boundaries/no-private` | the dependency matrix in `01-architecture.md` §2; no deep imports into `@console/services` internals |
| `no-restricted-globals` (window, document, localStorage, navigator…) outside `app/utils/client/**`, `*.client.ts` | SSR-readiness (§3.1) |
| `no-restricted-imports`: `axios` (use services), `primevue/*` (gone), `@aziontech/webkit` root (use subpath exports), `yup` (zod only) | stack discipline |
| ported `azion-architecture/pure-adapters` | adapters: no IO, no flags, no toast/UI, no service-to-service calls |
| ported `azion-architecture/no-io-in-components` | HTTP only inside `services/` |
| new `no-flags-outside-app` | `useFlag` banned in `services/` and `packages/components` |
| new `no-store-outside-stores-dir` + review allowlist of 3 store ids | Pinia = true globals only |
| `@aziontech/webkit/eslint-plugin` `strict` | webkit usage rules incl. `prefer-webkit-component` |
| `eslint-plugin-security` + `eslint-plugin-no-unsanitized` + `vue/no-v-html` | injection/XSS surface (replaces dead `eslint-plugin-xss`) |
| `@nuxt/eslint` project preset + `eslint-plugin-vue` flat/recommended | Nuxt/Vue correctness |
| `import-x/order` (builtin → external → workspace → relative, alphabetized) | diff stability |
| `vue/component-api-style`, `vue/block-lang`, `vue/define-props-declaration` | SFC shape above |
| `max-lines` 400 (source), 40 (`app/pages/**`) | the 1,500-line `.vue` files end here |

Stylelint 17 with `@aziontech/webkit/stylelint-config` for the few CSS files.

## 4. Commits, branches, PRs

| Rule | Detail |
|---|---|
| Commits | Conventional Commits enforced by commitlint 21. Types: `feat fix refactor test docs chore ci perf build`. **Scopes = workspace or domain**: `console`, `components`, `services`, `config`, `testing`, `e2e`, or a business domain (`edge-dns`, `workloads`, `billing`…). |
| Branches | `<type>/<scope>-<slug>` (e.g. `feat/edge-dns-records-drawer`). |
| Changelog | generated by git-cliff at deploy time; never hand-edited. |
| PRs | One domain per PR. Template requires: what/why, verification evidence (`pnpm verify` output or CI link), screenshots for UI, checklist for agent-authored PRs (see `05-ai-first.md` §7.6). |
| Hooks | pre-commit: lint-staged (eslint --fix + prettier, staged files only). commit-msg: commitlint. Nothing slower than ~5 s in hooks — the full gate is CI's job. |

## 5. Testing conventions

| Layer | Tool | What belongs here | Convention |
|---|---|---|---|
| Unit (node) | Vitest 4.1, node env | adapters, schemas, utils, composable logic, query-key factory | pure, no DOM; MSW for HTTP via `@console/testing` handlers |
| Component | Vitest browser mode (Playwright provider) + `vitest-browser-vue` | `packages/components` blocks; feature components with interaction | Storybook stories double as tests via `@storybook/addon-vitest` (portable stories) |
| Contract | Vitest (node) in `services/` | generated-type drift vs. fixtures; see `03-services.md` §5 | regenerate types in CI, `git diff --exit-code` |
| E2E | Playwright 1.62, `apps/console/e2e` | smoke journeys per migrated wave + parity checks vs. old console | tagged `@smoke` subset runs in the PR gate; full suite nightly |
| Coverage | v8 provider | **ratchet** (floors only rise), new-code target 90% | `scripts/check-coverage-ratchet` port; baseline JSON in repo |

Test naming: `describe` = unit under test, `it` = behavior in plain English ("shows one toast per API error message"). Flaky tests are deleted or fixed within a day — a flaky gate trains agents (and humans) to ignore red.

## 6. Command surface (identical local and CI — see `05-ai-first.md` §7.3)

| Command | Definition |
|---|---|
| `pnpm dev` | console dev server |
| `pnpm lint` / `pnpm lint:fix` | turbo eslint + stylelint + prettier check |
| `pnpm typecheck` | turbo vue-tsc |
| `pnpm test:unit` | turbo vitest (node + browser projects) |
| `pnpm test:e2e:smoke` | Playwright `@smoke` grep |
| `pnpm gen:<thing>` | plop generators (`service`, `crud`, `component`, `adr`) |
| `pnpm gen:api-types` | refresh generated OpenAPI types (`03-services.md` §4) |
| `pnpm verify` | the whole gate: lint + typecheck + test:unit + build (+ smoke when app affected), turbo-cached |
