# 02 — Roadmap

> Incremental strangler migration; no big-bang. Sizes are relative (S/M/L), never hours. Exit criteria are binary — a phase is done when every box checks, not when it "feels done".

## Coexistence mechanics (applies to every phase — user-confirmed strategy)

- **Same domain, per-route strangler.** `console.azion.com` is served by both apps: the edge application's request rules route path prefixes to the new app's bucket/function; everything else falls through to the old SPA. Session is shared natively (HttpOnly cookies on `.azion.com` — neither app reads tokens).
- **Cutover unit = route prefix** (e.g. `^/edge-dns`). Rollback = flip one edge rule back (minutes, no deploy).
- **Fork-sunset rule:** a path may cut over only if (a) it has no account-flag fork at that path, or (b) the fork's losing side has been sunset (flag at 100%). Path-disjoint forks (`/domains` v3 vs `/workloads`) need nothing — `/domains` simply never cuts over and retires with the old console.
- **Cross-app navigation is free**: plain `<a>`/location navigations between prefixes hit the edge, which picks the right app. New-app middleware sends unauthenticated users to `/login` (old app) until wave F migrates auth UI.
- **Dual-run policy:** when a wave starts, its domains are feature-frozen in the old console (fixes only). Prevents chasing a moving parity target.
- **Parity checklist per route** (the validation instrument, kept in `apps/console/e2e/parity/`): screen inventory diff, URL/query contract, every user action, empty/error/loading states, tracker events, a11y pass. Signed off by the domain owner before the edge rule flips for 100%.
- **Safeguard:** per-account kill-switch flag (`console_next_optout`) evaluated in an edge rule, forcing the old app for support escalations during a cutover's first weeks.

---

## Phase 0 — Monorepo foundation & tooling

- **Objective:** a repo where `pnpm verify` gives a green/red answer in minutes, and every convention in `06-conventions.md` is enforced by machinery, before any product code exists.
- **Scope:** pnpm 11 workspace + catalogs; turbo 2.10 pipeline (`lint`, `typecheck`, `test:unit`, `build`, `verify`); `@console/config` (eslint 10 flat presets incl. boundaries/SSR-safety/security rules, tsconfigs, vitest presets, stylelint); prettier/husky/lint-staged/commitlint; empty-but-wired `@console/components`, `@console/services`, `@console/testing`; `apps/console` hello-Nuxt (`ssr:false`) with theme+webkit CSS contract rendering one webkit button; `apps/storybook` shell with one story running via addon-vitest; plop generators (`gen:service`, `gen:crud`, `gen:component`, `gen:adr`); `gen:api-types` producing `contracts/generated/api-v4.d.ts` from the processor spec; CI: pre-merge gate + security suite (gitleaks/semgrep/zizmor/osv) + stage deploy of the hello app via `azion` CLI; CLAUDE.md/AGENTS.md + ADR seed (ADRs 001–00N recording the decisions in `01-architecture.md`); CODEOWNERS + branch protection.
- **Prerequisites:** repo access to the OpenAPI spec bucket; stage bucket + edge application for the new app; Sentry project + DSN.
- **Exit criteria:**
  - [ ] `pnpm verify` passes locally and in CI from a clean clone (documented < 10 min cold, < 2 min warm cache)
  - [ ] A PR violating a boundary rule (page importing `services/impl`) fails CI with an actionable message
  - [ ] `pnpm gen:crud --domain demo` output compiles, lints, tests green with zero manual edits
  - [ ] Hello app deployed to stage bucket and served by a test edge rule; webkit button renders styled (proves the Tailwind-4/theme CSS contract)
  - [ ] `gen:api-types` runs and the drift CI job is green
- **Risks / mitigation:** ESLint-10 compat gaps in ported custom rules → budget a spike, rules are small ASTs; Tailwind-4 content detection misses (`@source`) → the exit criterion above catches it on day one.
- **Size:** M

## Phase 1 — App shell: session, chrome, middleware, observability

- **Objective:** an authenticated user with existing cookies opens a new-app route and sees the full console chrome with their account loaded — without the new app owning any login UI.
- **Scope:** layouts (`default`, `auth`, `fullscreen`); global middleware chain (`01.logout` → `02.auth` → `03.billing` → `04.flags` → `05.redirect`); `services` W1 domains (`aux-auth` verify/refresh/logout, `account` identity) + session/preferences/chrome stores; `useFlag`/`usePermission`; chrome on webkit `global-header`/`sidebar`/`menu-*`/`platform-shell` (product menu incl. flag gating, profile menu, theme switcher; account switcher renders read-only + links to old-app `/switch-account`); error pages (403/404/500 via `error.vue`); toast bridge (`useApiToast`); Sentry module live (release=SHA, sourcemaps, masking policy); Segment analytics plugin (tracker facade port); Nuxt loading indicator; e2e smoke: cookie-authenticated navigation across chrome.
- **Prerequisites:** Phase 0.
- **Exit criteria:**
  - [ ] With valid stage cookies, `/` on the new app (behind a stage-only edge rule) renders chrome + account identity; without cookies it redirects to old-app `/login` and back after login
  - [ ] Token refresh path proven: expired access cookie → one transparent retry → session continues (e2e green)
  - [ ] All 8 old guard behaviors covered by middleware unit tests (incl. billing lock redirect and flag 404)
  - [ ] A thrown render error produces `error.vue` AND a Sentry event with release + readable stack (sourcemapped) + `domain` tag
  - [ ] Page-view and identify events arrive in Segment stage with the same payload shape as the old console
- **Risks / mitigation:** subtle guard-order semantics → port the old guard unit tests (10 exist) as middleware tests first; webkit chrome gaps vs current navbar widgets → gap list resolved in this phase, widgets are small.
- **Size:** L

## Phase 2 — Vertical slice: Edge DNS end-to-end ⭐ (the pattern-setting phase)

- **Objective:** one real domain (`/edge-dns`) runs 100% on the new stack in production for all accounts, proving every pattern the remaining waves will copy.
- **Why Edge DNS:** mid-size (8 views), fully on v4 (`dns-api.yaml`, 15 ops), already v2-served (clean reference), exercises the entire platform kit — list with server pagination/search, create/edit forms, records tab, drawers, delete-confirm, toasts, tracker — with no versioning complexity and no account-flag fork at its path (cutover rule §above satisfied trivially).
- **Scope (all named):**
  - Services: `contracts/edge-dns.ts` (+ generated types), `impl/http/edge-dns/{api,adapter}`, `composables/use-edge-dns.ts`, MSW handlers + fixtures, unit + contract tests — built **via `gen:service`**, then refined.
  - Components (first real consumers — APIs stabilize here): `PageShellBlock`, `ListTableBlock`, `FormBlock` + `FormActionBar`, `EntityDrawerBlock`, `DeleteDialogBlock`, `FormSkeleton`, `UnsavedChangesGuard` — each with story + play-test + a11y green.
  - Pages: `pages/edge-dns/index.vue` (zones list), `create.vue`, `edit/[id]/[tab].vue` (settings + records), records drawer CRUD — **generated by `gen:crud`**, then customized; zod schemas in `features/edge-dns/schemas`.
  - Tests: unit (adapter/schema/composable), component (blocks via stories), e2e journey (create zone → add record → edit → delete) tagged `@smoke`, parity checklist executed against old `/edge-dns`.
  - Rollout: stage cutover → prod edge rule for `^/edge-dns` at 100% + kill-switch flag armed; old-console Edge DNS feature-frozen at phase start.
  - Pattern documentation: `docs/adr/` records for "anatomy of a domain migration"; generator templates updated with every deviation discovered (**the generator is the pattern's executable spec**).
- **Prerequisites:** Phases 0–1; Edge DNS freeze in old console.
- **Exit criteria:**
  - [ ] `^/edge-dns` served by the new app in **production** for all accounts; kill-switch tested (flip → old app serves; flip back)
  - [ ] Parity checklist signed by domain owner; zero P1/P2 defects open after 2 weeks at 100%
  - [ ] E2E journey green in CI; `@smoke` suite < 5 min
  - [ ] `gen:crud` regenerated against the final patterns reproduces a compiling, green Edge-DNS-shaped skeleton (proves the next 40 domains are cheap)
  - [ ] Coverage ratchet ≥ 90% on `features/edge-dns` + `services` edge-dns files; bundle budget baseline recorded and enforced (entry + route chunk)
  - [ ] Sentry: zero unhandled-error events with `domain=edge-dns` in the last 7 days of the bake period
- **Risks / mitigation:** platform blocks over-fitted to DNS → wave A immediately stress-tests them on 8 more domains before wave B builds on them; hidden behavior parity gaps → the checklist includes tracker events and URL/query contracts, both diffable; production incident → kill-switch + 2-week bake at stage first.
- **Size:** L

## Phase 3 — Wave A: simple CRUD domains

- **Objective:** the platform kit survives contact with 8 domains built mostly by generator, at a pace that validates the "agent executes, human reviews" model.
- **Scope:** `/personal-tokens`, `/credentials`, `/users`, `/teams-permission`, `/settings`, `/real-time-purge`, `/data-stream` (+ domain-limit rule), `/edge-node` + `/edge-services` (orchestrator services move v3→v4 per spec), `/edge-pulse`. New blocks: `InfoDrawerBlock`, `CopyKeyDialog`, `DangerZoneBlock`.
- **Prerequisites:** Phase 2 exit (patterns frozen).
- **Exit criteria:** every listed prefix at 100% prod cutover with signed parity checklist; ≥ 80% of new page/service files generator-produced (measured by generator manifest); smoke suite still < 8 min; zero regressions on ratchet/budgets.
- **Risks / mitigation:** orchestrator v3→v4 endpoint behavior drift → contract tests against both during transition; generator drift → Phase 2's regeneration criterion repeated per domain.
- **Size:** L

## Phase 4 — Wave B: core configuration + versioning framework

- **Objective:** the product's heart — workloads, applications, security config and the draft→deploy versioning system — runs on the new stack.
- **Scope:** `/workloads`, `/deployments` (+ release composer), `/applications`, `/firewalls`, `/waf`, `/functions` (Monaco via `CodeEditorBlock`, JSONForms args via `JsonFormsRenderers`), `/digital-certificates`, `/network-lists`, `/custom-pages`, `/connectors`, `/variables`, `/environments`; `features/versioning` rebuild (version shell, capability machine, command bus, release composition on webkit `flow-*`); services per `03-services.md` order 4.
- **Prerequisites:** Wave A done; **v6 flag sunset confirmed per domain before its path cuts over** (fork-sunset rule); versioning UX owner allocated.
- **Exit criteria:** all prefixes at 100% with signed checklists; versioning e2e (draft → build → deploy → rollback) green; release-composer parity approved by its PM; rules-engine forms (the 1,000-line monsters) rebuilt within `max-lines` budgets; Monaco bundled locally (no CDN request in prod — verified by CSP report/e2e).
- **Risks / mitigation:** this is the riskiest wave — versioning semantics are subtle → port its mutation-tested composables' test suites first (they are the executable spec); v6 sunset slips → wave C/D proceed in parallel, B waits per-domain, the strangler makes partial progress safe.
- **Size:** L (largest)

## Phase 5 — Wave C: observability

- **Objective:** events and metrics dashboards on the new stack with one filter implementation instead of three.
- **Scope:** `/real-time-events` (V2 experience only), `/real-time-metrics`, `/activity-history`; `AdvancedFilterBlock` (single AQL parser), `DateTimeRangeBlock`, `ChartBlock`/`MapChartBlock` (c3 + OpenLayers as lazy islands); services: `aux-events`, `aux-metrics` (GraphQL), `metrics-api` dashboards CRUD.
- **Prerequisites:** Wave A (kit hardened); RTE v1 retirement decision (open question Q-7) — else v1 users stay on old console.
- **Exit criteria:** prefixes at 100%; AQL filter passes the ported parser test corpus (both old parsers' cases); dashboard TTI within budget on a mid-tier laptop profile in LHCI; auto-refresh + SSE stable for 1h soak test.
- **Risks / mitigation:** c3 is unmaintained → wrapped behind `ChartBlock` so a future lib swap is one package change (recorded as debt, not done now).
- **Size:** L

## Phase 6 — Wave D: data plane

- **Objective:** the heavy interactive tools (SQL workbench, object storage browser) migrate.
- **Scope:** `/sql-database` (Monaco SQL editor, tables browser), `/object-storage` (upload/download, jszip flows, kv).
- **Prerequisites:** `CodeEditorBlock` from wave B.
- **Exit criteria:** prefixes at 100%; large-object upload/download e2e green (multi-MB fixtures); SQL editor keyboard/undo parity checklist signed.
- **Risks / mitigation:** file-handling browser APIs are SPA-only patterns → concentrated in client-only islands, already the architecture.
- **Size:** M

## Phase 7 — Wave E: account, billing, IAM management

- **Objective:** money- and identity-touching screens migrate with extra bake time.
- **Scope:** `/account`, `/billing` (+ Stripe elements, invoices, GraphQL accounting aux), `/identity-providers`, `/mfa-management`, `/client|group|reseller/management`.
- **Prerequisites:** wave A IAM services; Stripe stage keys; billing PM sign-off process agreed.
- **Exit criteria:** prefixes at 100% after a 4-week staged rollout (stage → internal accounts → 100%); payment-method add/change e2e green against Stripe test mode; billing-lock middleware behavior byte-identical to old guard (unit-diff test).
- **Risks / mitigation:** payment regressions are reputationally expensive → longest bake, kill-switch default-armed, Sentry alert threshold at 1 event.
- **Size:** M

## Phase 8 — Wave F: public surface & long tail

- **Objective:** the new app owns its front door; nothing user-facing remains on the old console.
- **Scope:** `/login` (+ social IdPs, reCAPTCHA), `/signup` + additional-data, `/password`, `/mfa`, `/switch-account` (full SSO callback flow), `/cli-callback-*`, `/gh-connect`, `/marketplace`, `/create` wizard, `/github` import, `/copilot` (SSE chat), **`/` home dashboard** (migrates here because its widgets aggregate every prior wave).
- **Prerequisites:** all prior waves at 100%; OAuth security headers replicated on new-app responses (edge response rules).
- **Exit criteria:** login/signup/MFA/switch-account e2e green incl. social IdP happy path on stage; COOP/security headers verified by e2e response assertions; home renders with all widgets from new services; **zero traffic to old-app HTML for 14 consecutive days (edge analytics)**.
- **Risks / mitigation:** auth flows are the highest-blast-radius change → they go last deliberately, when the platform is boring and battle-tested; social IdP callback quirks → parity e2e recorded against old flow first.
- **Size:** L

## Phase 9 — Decommission

- **Objective:** one console.
- **Scope:** delete old-app edge rules + bucket; archive `azion-console-kit` (read-only, final tag); remove kill-switch flag + dual-run tooling; sweep redirects for any retired URL; close the plan with a final ADR.
- **Exit criteria:** old bucket serves nothing for 30 days (edge logs) then deleted; repo archived; `07-open-questions.md` fully resolved or converted to tracked issues; team retro documented.
- **Size:** S
