# 01 — Target Architecture

> Decisions here follow the rule "decide, don't list": one choice per topic, with the trade-off that justified it. Anything genuinely undecidable from code/docs is in `07-open-questions.md`. Versions were verified against the npm registry on **2026-08-11** (not assumed).

## 1. Monorepo topology

```
console-kit/
├── apps/
│   ├── console/                # Nuxt 4 app (the product)
│   │   ├── app/                #   Nuxt srcDir: pages/, layouts/, middleware/, plugins/, features/
│   │   ├── e2e/                #   Playwright specs + fixtures (colocated: they test THIS app)
│   │   ├── nuxt.config.ts
│   │   └── azion.config.mjs    #   edge deploy (origins/rules for the strangler live here)
│   └── storybook/              # Storybook 10 for packages/components (standalone Vite)
├── packages/
│   ├── components/             # @console/components — platform blocks on top of webkit
│   ├── config/                 # @console/config — shared eslint/ts/vitest/stylelint presets
│   └── testing/                # @console/testing — MSW handlers, fixtures, factories, test utils
├── services/                   # @console/services — disposable API layer (see 03-services.md)
├── tools/
│   └── generators/             # plop generators (service, CRUD page set, component, ADR)
├── docs/
│   ├── adr/                    # architecture decision records (MADR-short)
│   └── plan/                   # this plan
├── CLAUDE.md · AGENTS.md       # agent context (see 05-ai-first.md)
├── turbo.json
├── pnpm-workspace.yaml         # packages: apps/*, packages/*, services, tools/*
└── package.json
```

Adjustments vs. the starting sketch, with reasons:

| Change | Why |
|---|---|
| **Added `packages/config`** | ESLint 10 flat-config presets, tsconfig bases, vitest presets and stylelint config must be importable by every workspace or conventions drift. One source, versioned together. |
| **Added `packages/testing`** | MSW server/handlers and API fixtures are consumed by `services` tests, `components` stories/tests and `apps/console` e2e. Without a home they get duplicated (the old console has 3 copies of filter fixtures). |
| **No `packages/types`** | Types live with their owners: API DTOs are generated into `services` (they die with it — that's the point), component prop types are exported by `components`. A shared types package becomes a dependency magnet that defeats the disposal boundary. |
| **No `apps/e2e`** | E2E tests exercise `apps/console` only; colocating them (`apps/console/e2e`) keeps one `--filter console` scope for agents and avoids a workspace that imports app internals. |
| **`services/` stays at root** (as in the sketch and the repo skeleton) | The location advertises its special status: not a lasting package, a scheduled-for-deletion seam. |
| **Storybook is standalone Vite, not `@nuxtjs/storybook`** | It documents `packages/components` (plain Vue), not the app; the Nuxt module (9.0.1) lags Storybook 10.5.7 and adds nothing here. |

### Internal packages ship raw source (no build step)

Same strategy webkit itself uses: `exports` maps point at `.vue`/`.ts` files; the consuming app's bundler compiles them. Justification: nothing is published to npm (non-goal), a build step would add watch orchestration and kill cross-package HMR, and Turborepo still caches the *checks* (lint/typecheck/test) per package. Consequence: `apps/console/nuxt.config.ts` sets `build.transpile: ['@aziontech/webkit', '@console/components']`.

## 2. Dependency rules (enforced, not reviewed)

Direction of allowed imports (→ = "may import"):

```
apps/console  →  @console/components, @console/services, @console/testing(dev), @aziontech/*
apps/storybook → @console/components, @console/testing(dev), @aziontech/*
@console/components → @aziontech/webkit|theme|icons, vue, @vueuse/core   (NEVER services, stores, app code)
@console/services   → zod, @tanstack/vue-query, ofetch, vue (composables) (NEVER components, webkit, app code)
@console/testing    → @console/services (contracts only), msw
@console/config     → (leaf; imports nothing internal)
```

Enforcement (all fail CI, none rely on review):

1. **`eslint-plugin-boundaries` 7.2** in the shared flat config — element types `app`, `components`, `services`, `testing`, `config` with the matrix above; also bans **deep imports** into `@console/services/*` internals (only the package root export is public — this is the disposal boundary, see `03-services.md`).
2. **pnpm workspace isolation** — a package can only import what is in its own `package.json`; `@console/components` simply does not declare `@console/services`.
3. **Ported architecture rules** from the old console's custom plugin (they exist and work — keep the good ones as `error` from day 1): `pure-adapters` (no IO/flags/UI in adapters), `no-io-in-components` (no HTTP outside services), plus new `no-browser-globals-outside-client-utils` (SSR-readiness, §3.1).
4. **`@aziontech/webkit/eslint-plugin`** `strict` preset (12 rules, including `prefer-webkit-component`, which bans PrimeVue and raw-HTML equivalents).

## 3. Platform decisions (§4.5 of the brief)

### 3.1 Rendering: SPA-first (`ssr: false`), SSR-ready by construction — **user-confirmed**

- **What**: Nuxt 4.5 with `ssr: false`. Build output = static assets in Azion object storage behind the existing edge-rules BFF — the deploy topology that already runs the product. `azion` CLI ≥3.1 has `nuxt`/`nitro` presets when we need them.
- **Why not SSR now**: an authenticated dashboard gets no SEO value; webkit 4.3.0 crashes under SSR (`dialog-content.vue:37` and `drawer-content.vue:43` dereference `document` at setup top level); 8 auth/billing/flag guards depend on browser storage; cookie-forwarding through Nitro adds a failure mode the edge BFF already solves.
- **Why Nuxt at all then**: file routing + layouts + middleware + typed config + module ecosystem + the option to flip specific routes to SSR/prerender later via `routeRules` without a second migration.
- **SSR-readiness is enforced, not aspirational**: browser globals (`window`, `document`, `localStorage`, `navigator`) are banned by lint outside `app/utils/client/**` and `*.client.ts` files; all storage goes through one `useClientStorage()` util; plugins that touch the DOM are `.client.ts`. When Azion's Nuxt SSR adapter matures and webkit fixes land, flipping is a config change plus a bounded audit, not an archaeology project.

### 3.2 Auth/session: keep the HttpOnly-cookie + edge model exactly

- Tokens (`azsid`, `_azrt`, `_azat`) remain HttpOnly cookies scoped to `.azion.com`, set/rewritten by edge response rules. **JS never reads tokens** — this survived discovery as the best part of the current design and is what makes the per-route strangler trivially safe (both apps share the session with zero work).
- App-side session state = `useSessionStore` (Pinia): `hasSession` marker + account identity loaded from `accounts/identity` — a port of `accountGuard`+`account-data.js`, as a composable-driven store instead of module singletons.
- Middleware chain (ordered global middleware replaces the 8 guards; numbering makes order explicit):
  `01.logout.global.ts` → `02.auth.global.ts` (public-route bypass via `definePageMeta({ public: true })`; bootstrap identity; on failure store intended route and `navigateTo('/login')` — which the edge serves from the old console until wave F) → `03.billing.global.ts` → `04.flags.global.ts` → `05.redirect.global.ts`. Theme is a client plugin, not a guard. Tracking moves to `router.afterEach` inside the analytics plugin.
- Token refresh: keep the current lazy semantics (verify → refresh on failure) inside the session composable; no timer. A global 401 interceptor in the HTTP client (missing today) retries once after refresh, then logs out — fixes the known race the old console dodges with guard-level try/catch.
- Dev ergonomics preserved: `NUXT_PUBLIC_PERSONAL_TOKEN` (dev-only header injection) and debug-login bypass, both no-ops in production builds.

### 3.3 Data fetching: services layer + TanStack Query v5 — not `useFetch`/`useAsyncData`

- `useAsyncData` solves SSR payload transfer, which `ssr:false` makes moot, and has no story for mutations, invalidation graphs, optimistic updates, IndexedDB persistence or cross-tab sync — all of which the current v2 layer already implements on TanStack Query and the product visibly depends on (22 post-login prefetches, cache-sync invalidation map).
- Port from v2 base (as TS): hierarchical **query-key factory**, cache-type presets, IndexedDB persister (+ encryption), abort manager, BroadcastChannel sync, SSE client. Pages consume only typed composables exported by `@console/services` (`useWorkloads().list(params)` etc.). Contract details and the disposal boundary: `03-services.md`.
- HTTP client: **ofetch 1.5** (not axios). Native to the Nuxt ecosystem, interceptor + retry hooks cover the 401 flow, typed responses, and it removes the last axios-global-mutation pattern. `withCredentials` semantics via `credentials: 'include'`.

### 3.4 State: Pinia 4 for true globals only

- Stores: `session` (account/identity/flags/permissions), `preferences` (theme, table page size — persisted via `pinia-plugin-persistedstate` 4, `pick` API), `chrome` (banner/sidebar/create-menu UI state). That is the whole list.
- Server data lives in the query cache, never in Pinia. The six page-scoped stores found in discovery (`release` 705 lines, `deploy`, `solution-create`, `edge-dns`, `purge`, `graphql-query`) are re-homed to feature-local composables/provide-inject; a lint rule caps store creation to `app/stores/**` so "quick global state" stops being the path of least resistance.

### 3.5 i18n: English-only, no i18n layer (matches product reality)

The current console has zero i18n and 100% hardcoded English; adding `@nuxtjs/i18n` would tax every page and generator with key indirection for a language count of one. Decision: no vue-i18n. Copy lives inline; validation messages come from zod schema messages. Retrofit path documented in `07-open-questions.md` (Q-3): if product commits to localization, extraction is mechanical (agents sweep `<template>` strings + zod messages into `@nuxtjs/i18n` catalogs); nothing in this architecture blocks it.

### 3.6 Feature flags: identity `client_flags` via one composable + page meta

- `useFlag('flag_name')` reads the session store (reactive, SSR-safe). Route gating: `definePageMeta({ flags: ['use_v6_configurations'] })` checked by `04.flags.global.ts` → `/not-found` on mismatch (same UX as today).
- Flags are **banned in `services/` and `packages/components`** (lint) — discovery found 10 service files reading flags, which is why adapters stopped being pure. Pages/features resolve flags and choose what to call/render.
- Winner-only scope kills the v6 fork machinery: the new app assumes v6-true / v4-account behavior; the cutover rule (`02-roadmap.md`) keeps legacy-flagged accounts on the old console per route.

### 3.7 RBAC/permissions: backend-enforced, thin client façade

Discovery found exactly two granular client checks; authorization truth is the API's 403. Decision: `usePermission()` façade over identity payload (covers today's two checks + `is_account_owner`), no client-side authz matrix, no route `meta.permission` until the backend RBAC model (teams/permissions v4 `identity-api`) stabilizes — tracked in `07-open-questions.md` (Q-4).

### 3.8 Errors & loading

- **`ApiError`** (TS port of v2 `ErrorHandler`): JSON:API-aware message extraction, `status/messages[]/code/meta`, thrown by every service impl. UI renders it through one `useApiToast()` bridge to webkit's toast (severity mapping, sticky errors, action buttons — the current `toast-block` policy).
- Route-level failures: Nuxt `error.vue` (403/404/500 pages reusing webkit `error-403`/`error-404` SVGs) + `NuxtErrorBoundary` on risky islands (editors, charts). This adds the global error boundary the old app lacks; Sentry captures via `@sentry/nuxt` hooks.
- Loading: Nuxt's route loading indicator replaces the `loadingGuard`/store; data loading states come from query flags rendered by skeleton blocks (`FormSkeleton`, table skeleton) — no bespoke loading store.

### 3.9 Routing & code splitting

- File-based pages **mirroring current URLs 1:1** (`/workloads`, `/applications`, `/edge-dns/...`). URL parity is a strangler invariant: the edge cuts over per path prefix, and deep links/bookmarks/docs keep working. Any URL change is a deliberate product decision recorded as an ADR, never a migration side effect.
- Per-page chunks are Nuxt's default; heavy libraries (Monaco ~2 MB, c3+d3, OpenLayers, exceljs, jszip) load via dynamic import inside client-only blocks so no list/form route pays for them. Budgets in CI enforce this (§5, `05-ai-first.md` §7.5).
- Route metadata contract: `definePageMeta({ title, breadcrumbs, flags, public, layout })`; `useHead` sets `Azion Console — <title>` centrally in the default layout.

### 3.10 Tailwind 4 + theme/webkit CSS — the coexistence answer

Verified against webkit 4.3.0 source: **webkit requires Tailwind 4** (1,643 uses of v4-only `utility-(--var)` syntax across 169 files; `@aziontech/theme` ships a v4 `@theme` stylesheet with 160 `@utility` rules; there is no v3 preset). So there is no v3/v4 conflict to manage — Tailwind 3 is the configuration that breaks. The rules that do matter:

```css
/* apps/console/app/assets/css/main.css — order is a contract */
@import '@aziontech/theme';          /* tailwindcss + @theme tokens + layers   */
@import '@aziontech/webkit/styles';  /* @source shim — REQUIRED: TW4 skips     */
                                     /* node_modules content detection          */
/* app-level @source for packages/components + app/, then app utilities */
```

plus `import '@aziontech/icons'` (icon font) at app entry. Integration via `@tailwindcss/vite` (faster than the PostCSS path webkit's CLI writes). No `tailwind.config.js` anywhere. Dark mode = `data-theme="dark"` on `<html>` (theme CSS swaps semantic vars; components carry no `dark:` classes). Known trap documented from source: never rely on the theme's own `@source` (it hardcodes a monorepo-relative path that dangles under pnpm); the webkit `styles` import is the fix. Z-index: use theme tokens (`--z-input-*`); webkit hardcodes 1100/50 for its overlays — app chrome must stay below 1100.

### 3.11 webkit integration specifics (from source, not README — the README documents two export paths that don't exist)

| Requirement | Where |
|---|---|
| `build.transpile: ['@aziontech/webkit', '@console/components']` | `nuxt.config.ts` (webkit ships raw SFCs) |
| `vite.optimizeDeps.include: ['vee-validate']` | `nuxt.config.ts` (port of `@aziontech/webkit/vite`) |
| `ToastPlugin` registered in `plugins/toast.client.ts` | it is SSR-guarded upstream, `.client` keeps intent explicit |
| CSS entry exactly as §3.10; icons imported once | `app/assets/css` + entry plugin |
| Icon props take `ai-*`/`pi-*` font classes (402 icons) | convention doc |
| `field-*` components need an enclosing vee-validate form context | `FormBlock` owns it (see `04-components.md`) |

### 3.12 Observability: `@sentry/nuxt` 10.70

- Module in `nuxt.config.ts`; client config only while SPA (server config joins if SSR flips). Source maps uploaded on deploy via the module's Vite plugin; **release = git SHA** (no semver theater for a continuously deployed app); `hidden` sourcemaps — never `inline` in prod (current bug).
- Port the replay masking policy (passwords, HMAC keys, `[data-sentry-mask]`). Tags on every event: `domain` (route prefix), `feature`, `owner` (generated from `CODEOWNERS` at build). Full error→agent-fix pipeline: `05-ai-first.md` §7.4.
- Web vitals via Sentry browser tracing; budgets and synthetic checks: `05-ai-first.md` §7.5.

### 3.13 Environment & runtime config

`runtimeConfig.public.*` (NUXT_PUBLIC_* envs) replaces the 20+ `VITE_*` keys; one typed `app/config/env.ts` module is the only reader (the old repo branches env logic in 3 places). Secrets never ship to the client and never live in the repo — discovery flagged a **committed `.env` with a live-looking Sentry auth token** in the old repo; rotate it and add gitleaks from Phase 0.

## 4. Root tooling decisions (§6 of the brief)

| Topic | Decision | Justification |
|---|---|---|
| Package manager | **pnpm 11.21** + `pnpm-workspace.yaml`; versions pinned via **catalogs** | single lockfile (old repo has two), strict isolation powers boundary rule #2, catalog gives agents one place to bump a version |
| Node | **24.x** (`.nvmrc` exists; engines `^24.11`) | Nuxt 4.5 engines: `^22.19 ∥ ^24.11 ∥ >=26` |
| Task runner | **Turborepo 2.10** | cached, affected-only `lint/typecheck/test/build`; the fast feedback loop is an AI-first requirement (`verify` < minutes, not tens) |
| ESLint | **ESLint 10 flat config** in `@console/config` (there is no eslintrc option anymore); composed of `@nuxt/eslint` 1.17 (app), `eslint-plugin-vue` 10.10 + `vue-eslint-parser` 10.4, `eslint-plugin-security` 4, `eslint-plugin-no-unsanitized` 4, `eslint-plugin-boundaries` 7.2, webkit plugin `strict`, ported architecture rules | one preset repo-wide; **drop `eslint-plugin-xss`** — 0.1.12, unmaintained for years, incompatible with flat config; its guarantees are covered by `no-unsanitized` + `vue/no-v-html` + `security` |
| Pre-commit speed | husky 9 + **lint-staged 17** (eslint --fix + prettier on staged files only); commit-msg = commitlint 21 conventional; typecheck/tests stay in CI (turbo-cached), not in hooks | staged-only keeps the hook <5s on any machine; the full gate is `pnpm verify` in CI |
| Formatting | prettier 3.9 (root `.prettierrc.json` already present) + `@prettier/plugin-oxc`? — no: default parser is fine; stylelint 17 with webkit's shipped `stylelint-config` for the few real CSS files |
| Versioning/changelog | **No semantic-release** — it exists to publish packages, and nothing is published (non-goal). Deploys are tagged `console/vYYYY.MM.DD-<sha>` by CI; changelog generated per deploy with **git-cliff 2.13** from conventional commits; Sentry release = SHA | keeps commit discipline useful (changelog, revert hygiene, Sentry correlation) without release-machinery overhead |
| Validation lib | **zod, pinned to the 3.25.x line** (not 4.x) | `@vee-validate/zod` peers `zod ^3.24`; webkit bundles zod 3; one zod version repo-wide beats a forms/services split. Upgrade trigger documented in `07-open-questions.md` (Q-6) |

## 5. Verified stack (npm registry, 2026-08-11)

| Package | Version | | Package | Version |
|---|---|---|---|---|
| nuxt | 4.5.2 | | @tanstack/vue-query | 5.101.4 |
| vue | 3.5.41 | | pinia / @pinia/nuxt | 4.0.2 / 1.0.1 |
| @aziontech/webkit | 4.3.0 | | pinia-plugin-persistedstate | 4.7.1 |
| @aziontech/theme | 4.3.0 | | ofetch | 1.5.1 |
| @aziontech/icons | 4.1.0 | | zod | **3.25.x line** (latest is 4.4.3 — see §4) |
| tailwindcss / @tailwindcss/vite | 4.3.3 | | vee-validate / @vee-validate/zod | 4.15.1 |
| @sentry/nuxt | 10.70.0 | | @jsonforms/core·vue·vue-vanilla | 3.8.0 |
| storybook / @storybook/vue3-vite | 10.5.7 | | motion-v | 2.3.0 (name confirmed) |
| @storybook/addon-vitest / addon-a11y | 10.5.7 | | vitest / @vitest/coverage-v8 | 4.1.10 |
| eslint | 10.8.1 | | @nuxt/test-utils | 4.1.0 |
| eslint-plugin-vue / parser | 10.10.0 / 10.4.1 | | @playwright/test | 1.62.1 |
| @nuxt/eslint | 1.17.0 | | msw | 2.15.0 |
| eslint-plugin-security | 4.0.1 | | openapi-typescript | 7.13.0 |
| eslint-plugin-no-unsanitized | 4.1.5 | | plop | 4.0.5 |
| eslint-plugin-boundaries | 7.2.0 | | turbo | 2.10.9 |
| prettier | 3.9.6 | | pnpm | 11.21.0 |
| husky / lint-staged | 9.1.7 / 17.3.0 | | typescript / vue-tsc | 7.0.2 / 3.3.9 |
| @commitlint/cli | 21.2.1 | | git-cliff | 2.13.1 |
| stylelint | 17.14.1 | | @lhci/cli / size-limit | 0.15.1 / 13.0.3 |

Compatibility notes verified: `@tailwindcss/vite` accepts Vite 5–8 (Nuxt 4.5 uses 7); `@sentry/nuxt` peers `nuxt >=3.7 ∥ 4.x ∥ 5.x`; `@storybook/vue3-vite@10` peers Vite 5–7; motion-v peers `@vueuse/core >=10`.

## 6. CI shape (details per area in 05/06)

Single required PR gate (`pre-merge`) built on turbo affected-graph: `lint` → `typecheck` → `test:unit` → `build` → `test:e2e:smoke` (app-affected only) + budget checks. Security suite ported from the old repo (it is good): gitleaks, semgrep, zizmor, osv-scanner, OSSF scorecard, `pnpm audit`. Deploys: push to `dev` → stage, push to `main` → production, both via `azion` CLI with per-env config dirs (pattern already proven in the old repo). Coverage uses the **ratchet** model from the old repo (floors only move up; new-code target 90%) — the mechanism survived contact with a 2,400-file codebase and is agent-friendly (binary, local, fast).
