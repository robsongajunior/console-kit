# 00 — Discovery

> Phase 0 inventories for the migration of `azion-console-kit` (Vue 3 + Vite SPA) to the new Nuxt monorepo (`console-kit`).
> Every claim below was read from code on **2026-08-11**. Sources:
>
> | Source | Path | Version |
> |---|---|---|
> | Current console | `/Users/unknown1/www/azion/azion-console-kit` (branch `dev`) | v1.57.2 |
> | Design system | `/Users/unknown1/www/azion/webkit/packages/webkit` | `@aziontech/webkit` 4.3.0 |
> | Tokens/theme | `/Users/unknown1/www/azion/webkit/packages/theme` | `@aziontech/theme` 4.3.0 |
> | Icons | `/Users/unknown1/www/azion/webkit/packages/icons` | `@aziontech/icons` 4.1.0 |
> | v4 OpenAPI pipeline | `/Users/unknown1/www/azion/azionapi-v4-processor`, `/Users/unknown1/www/azion/azionapi-v4-go-sdk-dev` | see `03-services.md` |
>
> Also read: current console's internal docs (`docs/ARCHITECTURE.md`, `docs/auth-and-authorization.md`, `docs/V6-GUIDELINES.md`, `.eslintrc-architecture.cjs`), `azion.config.mjs`, all 49 route modules, `src/services/**`, `src/templates/**`, `src/components/**`.

## 1. Current system at a glance

| Fact | Value |
|---|---|
| Runtime | Vue 3.5.40 + Vite 6, pure JS (no TS build, `jsconfig.json` only), Yarn 1 classic (27 `resolutions`) |
| Scale | `src/` = 2,387 files — views 644 · services 587 · tests 502 · templates 188 · components 111 |
| Routing | 49 route-module files → 51 top-level records, ~160 paths, 156 named routes; 8 sequential global guards + 9 `beforeEnter` sites |
| UI stack | `@aziontech/webkit` **3.0.5** (PrimeVue-3-based) + primeflex + Tailwind **3** (`important: true`) — the new webkit 4.3 is PrimeVue-free and Tailwind-4-only |
| Data | Two service generations (v1 functional ×376 files, v2 class-based ×211); v2 = TanStack Query v5 + IndexedDB persistence + BroadcastChannel cross-tab sync |
| Auth | HttpOnly cookies (`azsid`, `_azrt`, `_azat`) set/rewritten **at the edge**; SPA never reads tokens; `hasSession` boolean in localStorage |
| Deploy | Static SPA in Azion object storage + **~27 edge rules acting as a same-origin BFF** for 17 origins (`azion.config.mjs`) |
| Flags | `client_flags[]` from identity payload → `useFlag()` composable (121 call sites) + route `meta.flag` guard; 12 view dirs forked v6-vs-legacy |
| i18n | None. All copy hardcoded English (100 route titles, 183 breadcrumb labels) |
| Analytics | Segment via `AnalyticsTrackerAdapter` facade + 6 domain trackers (101 view files) |
| Observability | `@sentry/vue` 10.42 (browser tracing + replay w/ masking); no global Vue `errorHandler` |
| Tests | 649 unit/spec files (~3,616 cases), coverage **ratchet** system, contract-drift via Playwright, Vitest-browser functional suite (versioning only), Stryker on 16 files. **No real E2E.** |

## 2. Route/screen inventory (§4.1)

Screen types: **L**ist / **D**etail / **C**reate / **E**dit / **W**izard / **Dash**board / **T**abs.
Size: S / M / L (relative migration effort, incl. views + services + tests).
Waves (defined in `02-roadmap.md`): **W1** shell · **W2** vertical slice · **A** simple CRUD · **B** v6 config + versioning · **C** observability · **D** data plane · **E** account/billing/IAM · **F** public auth, marketplace, home & long tail · **✂** not migrated (losing fork side, stays in old console until flag sunset).

| # | Route module (`src/router/routes/`) | Path(s) | Domain | Types | Size | Dependencies (services · flags · guards) | Wave |
|---|---|---|---|---|---|---|---|
| 1 | `home-routes` | `/` | Home dashboard | Dash | M | many domain widgets, onboarding | F |
| 2 | `login-routes` | `/login` | Auth | D | M | v1 `auth-services`, social IdPs, reCAPTCHA · `isPublic` | F |
| 3 | `signup-routes` | `/signup`, `/signup/additional-data` | Auth | W | M | v1 `signup-services` (15), HubSpot, reCAPTCHA · `isPublic` | F |
| 4 | `password-routes` | `/password/new/:uidb64/:token?` | Auth | D | S | v1 password reset · `isPublic` | F |
| 5 | `mfa-routes` | `/mfa/setup`, `/mfa/authentication` | Auth | D | S | v2 `mfa`, qrcode · `isPublic` | F |
| 6 | `switch-account-routes` | `/switch-account` | Auth (SSO callback) | D | S | v1 `auth-services`; whole flow in `beforeEnter` · `isPublic` | F |
| 7 | `cli-callback-routes` | `/cli-callback-{success,fail}` | Auth (CLI) | D | S | personal-token generation · `isPublic` | F |
| 8 | `github-routes` | `/gh-connect` | VCS OAuth popup | D | S | v2 `vcs` | F |
| 9 | `error-routes` | `/forbidden`, `/:catchAll(.*)` | Error pages | D | S | none | **W1** |
| 10 | `edge-dns-routes` | `/edge-dns` (+ zones, records tab) | Edge DNS | L C E T | M | v2 `edge-dns` | **W2 (slice)** |
| 11 | `personal-tokens-routes` | `/personal-tokens` | IAM | L C | S | v2 `personal-token` | A |
| 12 | `credentials-routes` | `/credentials/:tab?` | IAM | L C E | S | v1 credentials | A |
| 13 | `users-routes` | `/users` | IAM | L C E | M | v1 `users-services` (16) + v2 `users` | A |
| 14 | `team-permission` | `/teams-permission` | IAM | L C E | S | v2 `teams`/`team-permission` | A |
| 15 | `your-settings-routes` | `/settings` | User profile | E | S | v2 `users`, `account` | A |
| 16 | `real-time-purge` | `/real-time-purge` | Cache purge | L C | S | v2 `purge` | A |
| 17 | `data-stream-routes` | `/data-stream` | Data Stream | L C E | M | v2 `data-stream` · `data_streaming_sampling` flag · `checkDomainsLimit` guard ×2 | A |
| 18 | `edge-node-routes` | `/edge-node` (+ services) | Edge Nodes | L C E T | M | v2 `edge-node` + v1 | A |
| 19 | `edge-services-routes` | `/edge-services` (+ resources) | Edge Services | L C E T | M | v2 `edge-service` | A |
| 20 | `edge-pulse-routes` | `/edge-pulse` | Edge Pulse | D | S | v1 edge-pulse | A |
| 21 | `activity-history-routes` | `/activity-history` | Audit log | L | S | events GraphQL | C |
| 22 | `workload-routes` | `/workloads` (+ deployments, versions) | Workloads | L C E T | L | v2 `workload` (8) + `deployment` (10) · flags ×5 (`checkout_access_without_flag`, `use_v6_configurations`) | B |
| 23 | `deployment-routes` | `/deployments` (+ `releases/new`) | Releases/deploys | L W | L | v2 `deployment`, `release-impact` (8); release-composer (45 files, 758-line composable) · `use_v6_configurations` | B |
| 24 | `edge-application-routes` | `/applications` (+ versions) | Edge Application | L C E T | L | v2 `edge-app` (20) · flags; v6-vs-legacy + V3 forks | B |
| 25 | `firewall-routes` | `/firewalls` (+ versions) | Edge Firewall | L C E T | L | v2 `edge-firewall` (11) · flags; 1,045-line rules-engine form | B |
| 26 | `edge-functions-routes` | `/functions` | Edge Functions | L C E | L | v2 `edge-function`, Monaco editor, JSONForms args · flags | B |
| 27 | `waf-rules-routes` | `/waf` (+ tuning, versions) | WAF | L C E T | L | v2 `waf` (6) · flags (`enable_waf_tuning_details_save`) | B |
| 28 | `network-lists-routes` | `/network-lists` (+ versions) | Network Lists | L C E | M | v2 `network-lists` · flags | B |
| 29 | `digital-certificates-routes` | `/digital-certificates` (+ CRL) | TLS Certificates | L C E T | L | v2 `digital-certificates` (13) · flags · `certificatesTabGuard` | B |
| 30 | `custom-pages-routes` | `/custom-pages` | Custom Pages | L C E | M | v2 `custom-page` · flags | B |
| 31 | `edge-connectors-routes` | `/connectors` | Edge Connectors | L C E | M | v2 `edge-connectors` · flags ×4 | B |
| 32 | `environment-routes` | `/environments` | Environments | L D | S | v2 `environment` · `use_v6_configurations` | B |
| 33 | `variables-routes` | `/variables` | Variables/secrets | L C E | M | v2 `variables` · `variablesTabGuard`; v6 fork dir | B |
| 34 | `real-time-events-routes` | `/real-time-events/v2/:tab?` | Observability | Dash L | L | `real-time-events-service-v2` (70 files, GraphQL+REST), advanced-filter-v2, AQL parser · v1↔v2 preference guard | C (v2 only) |
| 35 | `real-time-metrics-routes` | `/real-time-metrics/:pageId?/:dashboardId?` | Observability | Dash | L | `real-time-metrics` module (59 files, 1,586-line report registry), GraphQL beholder, c3 charts, OpenLayers map | C |
| 36 | `edge-sql-routes` | `/sql-database` | Edge SQL | L T | L | v2 `edge-sql` (518-line adapter), Monaco, 1,335-line TablesView | D |
| 37 | `edge-storage` | `/object-storage` | Object Storage | L C E T | L | v2 `edge-storage`, jszip, 1,308-line ListView | D |
| 38 | `account-routes` | `/account` | Account settings | E | S | v2 `account` | E |
| 39 | `billing-routes` | `/billing/:tab?`, invoice details | Billing | T L D | L | v1 `billing-services` (14) + GraphQL accounting + Stripe | E |
| 40 | `identity-providers-routes` | `/identity-providers` | SSO (SAML/OIDC) | L C E | M | v1 `identity-providers-services` (12) · `federated_auth` flag guard ×3 | E |
| 41 | `mfa-management-routes` | `/mfa-management` | Admin MFA reset | L | S | v2 `mfa` | E |
| 42 | `clients-management-routes` | `/client/management` | Reseller→client mgmt | L C E | M | v1 accounts-management | E |
| 43 | `groups-management-routes` | `/group/management` | Group accounts | L C E | M | v1 accounts-management | E |
| 44 | `reseller-management-routes` | `/reseller/management` | Reseller accounts | L C E | M | v1 accounts-management | E |
| 45 | `marketplace-routes` | `/marketplace` (+ solution detail) | Marketplace | L D | M | v1 `marketplace-services` | F |
| 46 | `create-new-routes` | `/create` (+ deploy) | Template wizard | W | M | v1 template-engine + script-runner, JSONForms | F |
| 47 | `import-github-routes` | `/github/:vendor/:solution` | Import from GitHub | W | M | v2 `vcs` + script-runner | F |
| 48 | `azion-ai-routes` | `/copilot` | AI Copilot | D | M | `azion-ai-chat` module, SSE `/ai` proxy | F |
| 49 | `domains/index.js` | `/domains` | Legacy Domains (API v3) | L C E | M | v1 `domains-services` · `checkout_access` flag (v3-only accounts) | **✂** |
| — | `real-time-events-routes` (v1 export) | `/real-time-events/:tab?` | Observability (legacy) | Dash | L | `real-time-events-service` (28) · preference guard | **✂** |
| — | v6-fork legacy sides | same paths as B-wave routes | 12 domains | — | — | `use_v6_configurations === false` accounts | **✂** |

**Decision (user, 2026-08-11):** migrate the *winning* side of every fork only. Accounts still flagged onto legacy sides (`checkout_access`, non-v6) keep using the old console for those paths until flag sunset; the per-route strangler makes this safe because fork pairs are either path-disjoint (`/domains` vs `/workloads`) or gated by a sunset prerequisite (see `02-roadmap.md` §cutover rules).

### Route meta contract found (must be reproduced in Nuxt `definePageMeta`)

| Meta | Count | Nuxt equivalent |
|---|---|---|
| `breadCrumbs[]` (incl. `dynamic`, `routeParam`, `toRoute`) | 105 | page meta + breadcrumb composable |
| `title` | 100 | `useHead` via shared page-meta helper |
| `flag` | 25 | page meta + `flags.global.ts` middleware |
| `isPublic` | 8 | page meta + `auth.global.ts` middleware |
| `hideNavigation` / `hideLinksFooter` / `fillViewport` / `hideLoading` | 6/1/1/n | layout selection + page meta |

## 3. Service inventory (§4.2)

Two generations coexist. **v2** (`src/services/v2/`, 211 files) is the current pattern: `<domain>-service.js` class extending `BaseService` (TanStack Query integration) + pure `<domain>-adapter.js`, transport via `httpService` singleton, errors via `ErrorHandler` (JSON:API-aware, self-renders as toast). **v1** (`src/services/<domain>-services/`, 376 files) is one exported async fn per operation over `AxiosHttpClientAdapter`, throws plain strings from a status-code switch. Views are 96% on v2 (295 v2 imports vs 10 v1-importing view files); **`src/router/` is still v1-heavy (26 v1 imports)** because route `props` inject services.

Typing: none compiled — 0 `.ts` in services; ~20% of files have JSDoc. Tests: 249 service test files under `src/tests/services/` + 9 co-located `__tests__` (v1-heavy, mock the transport). "v4 spec" column = whether the domain's endpoints live under `/v4/*` today (evidence: `baseURL` fields in code), i.e. covered by the `azionapi-v4-processor` OpenAPI pipeline that will feed the disposable services layer (`03-services.md`).

| Domain (dir) | Gen | Base endpoint(s) | Consumers | Tests | Error pattern | v4 spec |
|---|---|---|---|---|---|---|
| `v2/workload` (8) + `v2/deployment` (10) + `v2/release-impact` (8) | v2 | `v4/workspace/workloads`, deployments | Workloads, Deployments, version shell | yes | ErrorHandler | ✅ |
| `v2/edge-app` (20) | v2 | `v4/edge_application/*` | Edge Application + sub-tabs | yes | ErrorHandler | ✅ |
| `v2/edge-firewall` (11) | v2 | `v4/edge_firewall/*` | Firewall + rules engine | yes | ErrorHandler | ✅ |
| `v2/waf` (6) | v2 | `v4/edge_firewall/wafs*` | WAF, tuning | yes | ErrorHandler | ✅ |
| `v2/edge-dns` | v2 | `v4/edge_dns/zones*` | Edge DNS | yes | ErrorHandler | ✅ |
| `v2/edge-function` | v2 | `v4/edge_functions/*` | Functions + app/firewall function instances | yes | ErrorHandler | ✅ |
| `v2/digital-certificates` (13) | v2 | `v4/digital_certificates/*` | Certificates, CRL | yes | ErrorHandler | ✅ |
| `v2/network-lists` | v2 | `v4/workspace/network_lists*` | Network Lists | yes | ErrorHandler | ✅ |
| `v2/custom-page` | v2 | `v4/workspace/custom_pages*` | Custom Pages | yes | ErrorHandler | ✅ |
| `v2/edge-connectors` | v2 | `v4/workspace/connectors*` | Connectors | yes | ErrorHandler | ✅ |
| `v2/variables` | v2 | `v4/variables*` | Variables | yes | ErrorHandler | ✅ |
| `v2/environment` | v2 | environments API (`/environment-api` origin) | Environments, version shell | some | ErrorHandler | ⚠️ aux API |
| `v2/data-stream` | v2 | `v4/data_stream/*` | Data Stream | yes | ErrorHandler | ✅ |
| `v2/edge-sql` | v2 | `v4/edge_sql/*` | Edge SQL | yes | ErrorHandler | ✅ |
| `v2/edge-storage` | v2 | `v4/workspace/buckets*` (S3-compat ops) | Object Storage | yes | ErrorHandler | ✅ |
| `v2/purge` | v2 | `v4/workspace/purge*` | Real-Time Purge | yes | ErrorHandler | ✅ |
| `v2/edge-node`, `v2/edge-service` | v2 | `v3` edge orchestrator endpoints | Edge Nodes/Services | yes | ErrorHandler | ✅ `orchestrator-api` (v4, 30 ops) — code still calls v3; migration targets v4 |
| `v2/account`, `v2/users`, `v2/teams`, `v2/team-permission`, `v2/personal-token`, `v2/mfa` | v2 | SSO/IAM (`/api/account`, `/api/user`, `v4/iam/*` mixed) | Account, IAM screens, guards | yes | ErrorHandler | ✅ `auth-api` + `identity-api` + `accounts-api` + `users-api` (v4) |
| `v2/billing`, `v2/payment` | v2 | `/billing`, GraphQL accounting | Billing | some | ErrorHandler | partial — `payments-api` + `billing-api` (v4, small); GraphQL accounting stays aux |
| `v2/marketplace`, `v2/vcs` | v2 | `/api/marketplace`, `/api/vcs` | Marketplace, GitHub import | some | ErrorHandler | ✅ `vcs-api` (v4, 17 ops); `marketplace-api` small |
| `v2/activity-history` | v2 | events GraphQL | Activity History | yes | ErrorHandler | ❌ GraphQL |
| `v2/appcues`, `v2/hubspot`, `v2/feedback`, `v2/new-release` | v2 | 3rd-party/aux | chrome, onboarding | some | ErrorHandler | ❌ aux |
| `real-time-events-service-v2` (70) | **v1-style** | events GraphQL + REST aggregations | RTE v2 screens | yes | string-throw | ❌ GraphQL |
| `real-time-metrics-services` (15) + `modules/real-time-metrics` | v1 | GraphQL beholder | RTM dashboards | yes | string-throw | partial — `metrics-api` (v4) covers dashboards/reports CRUD; the query engine stays GraphQL |
| `auth-services` (16) | v1 | SSO `/api/token*`, `/logout`, switch-account | login, guards, session | yes | string-throw | ❌ SSO API |
| `signup-services` (15) | v1 | SSO signup, activation | Signup | yes | string-throw | ❌ SSO API |
| `users-services` (16), `accounts-management-services`, `identity-providers-services` (12) | v1 | SSO/IAM | IAM screens (E wave) | yes | string-throw | partial |
| `billing-services` (14) | v1 | `/billing`, GraphQL accounting, Stripe | Billing | yes | string-throw | ❌ aux |
| `domains-services` | v1 | `v3/domains` | Legacy Domains | yes | string-throw | ✂ not migrated |
| `edge-application-services` (17) + origins/etc. | v1 | `v3/edge_applications` | legacy fork sides | yes | string-throw | ✂ not migrated |
| `template-engine`, `script-runner`, `marketplace-services` | v1 | `/api/template-engine`, `/api/script-runner`, `/api/marketplace` | Create wizard, Marketplace | some | string-throw | ❌ aux |
| ~15 more small v1 dirs (`edge-pulse`, `credential-services`, `cities` GraphQL, `status-page`, …) | v1 | misc | scattered | partial | string-throw | mixed |

**Cross-cutting v2 infrastructure worth porting (found in `src/services/v2/base/`):** `queryKeys.js` (691-line hierarchical key factory), `queryOptions.js` (cache-type presets), `indexedDbPersister.js` + `encryption.js`, `abortManager.js` (id/group cancellation), `sse/sse-client.js`, `broadcast/` (cross-tab), `cache-sync/` (invalidation map + prefetch scheduler), `auth/sessionManager.js` (22 post-login prefetches). The **contract-drift engine** (`tests/contracts/openapi-drift-engine.js` + 16 yup schemas) is superseded by generating types/fixtures from the processor's OpenAPI (see `03-services.md`).

## 4. Component inventory + webkit gap analysis (§4.3)

The old console composes pages from `src/templates/*-block` (188 files) + `src/components` (111 files), all built on **PrimeVue via webkit 3**. Webkit 4.3 is PrimeVue-free (TanStack Table, vee-validate, hand-rolled overlays), so platform blocks are **rebuilt on the new API, not ported**. Full mapping in `04-components.md`; summary:

### 4.3.a Already in webkit 4.3 → use directly (old name → new)

| Old (console) | New (`@aziontech/webkit/*`) |
|---|---|
| PrimeVue `DataTable` usage inside ListTable | `table-*` compound (16 parts, TanStack) + `paginator-*` |
| PrimeVue Button/Inputs/Dropdown/etc. (everywhere) | `button`, `input-*`, `field-*` (13 vee-validate wrappers), `select-*`, `multi-select-*`, `calendar-*` |
| PrimeVue Dialog / Drawer / OverlayPanel | `dialog-*`, `drawer-*`, `popover-*`, `panel-*` |
| `toast-block` (custom renderer) | `toast` compound + `ToastPlugin` (`useToast`) |
| `create-modal-block` (574 lines, "Create ▾" picker) | `command-menu-*` compound (7 parts) |
| Navbar/sidebar/menus (`src/layout/*`) | `global-header`, `sidebar-*`, `menu-*`, `platform-shell` template |
| Breadcrumbs, tabs, tags, badges, skeletons, empty states | `breadcrumb`, `tab-view`, `tag`, `badge`, `status-indicator`, `skeleton`, `empty-state` |
| `dialog-copy-key`, copy actions | `copy-button`, `dialog-*` |
| Code/log viewers (RTE drawers) | `code-block`, `log-view-*` |
| Sign-in/sign-up scaffolding | `sign-up-card`, `onboarding-form`, `deploy-success`, `plan-success` templates |
| password requirements checklist | `password-requirements` util |

### 4.3.b Upstream candidates (contribute to webkit later — NOT this migration)

| Candidate | Why upstream |
|---|---|
| Advanced filter system (AQL builder + field rows + applied tags) | Exists in **3 divergent copies** in the console; generic "filter a data set" UX; webkit already owns `table-filter`/`table-applied-filters` primitives it would compose with |
| Date-time-range picker with quick presets + auto-refresh interval | Generalization of webkit `calendar-preset`; used by all observability screens |
| Info-label primitives (`big-number`, `text-info` display pairs) | Generic read-only data display, pairs with `item-*` compound |
| One-time secret reveal dialog | Generic security UX (tokens, keys) |

### 4.3.c Stays in `packages/components` (platform blocks) — the build list

Each is console-specific composition/orchestration that webkit (a generic DS) rightly does not own. Justification per item in `04-components.md`.

| Block | Composes (webkit) | Replaces (old console) |
|---|---|---|
| `ListTableBlock` | `table-*`, `paginator-*`, `empty-state`, `skeleton` | `list-table/ListTable.vue` + 753-line `useDataTable` (51 consumers) |
| `FormBlock` (create/edit) + `ActionBar` | `field-*`, `button`, `panel` | `create-form-block` (136 uses) + `edit-form-block` (34) + `action-bar-block` (77) |
| `EntityDrawerBlock` (create/edit/info) | `drawer-*` | `create-drawer-block` (29), `edit-drawer-block` (15), `info-drawer-block` (10) |
| `PageShellBlock` (heading + docs link + actions + content) | `platform-shell`, `overline`, `link` | `content-block` (117) + `page-heading-block` (108) |
| `DeleteDialogBlock` (typed confirmation) | `dialog-*`, `field-text` | `delete-dialog` + `useDeleteDialog` |
| `DangerZoneBlock` | `card-box`, `button` | `danger-card-block` |
| `FormSkeleton` (schema-driven) | `skeleton` | `skeleton-block/*` (8 files, 25 uses) |
| `UnsavedChangesGuard` (composable + dialog) | `dialog-*` | `dialog-unsaved` + `useUnsavedChanges`/`useTabUnsaved`/`useDrawerUnsaved` (16 uses) |
| `JsonFormsRenderers` (webkit renderer set) | `field-*` | `form-fields-inputs/jsonform-custom-render/*` (10 files, vanilla-based) |
| `CodeEditorBlock` (Monaco, locally bundled) | — | `@guolao/vue-monaco-editor` w/ **CDN** Monaco |
| `ChartBlock` (c3 wrappers) + `MapChart` (OpenLayers) | `card-box` | `graphs-card-block` (31 files) |
| `AdvancedFilterBlock` (single unified impl) | `table-filter`, `popover`, `calendar` | the 3 divergent copies (until upstreamed, §4.3.b) |
| `VersioningShell` feature family | `flow-*`, `item-*`, `dialog-*` | `version-shell-block` (11) + `release-composition` (45) + `composables/versioning` (26) — ships as an app feature module first, extracted to a package only when a second consumer exists |

## 5. Tech debt & migration traps (§4.4) — DO NOT PORT

### 5.1 Parallel generations (port only the winner)

| Duplication | Winner | Loser (do not port) |
|---|---|---|
| Services v1 (376 files) vs v2 (211) | v2 pattern (as TS) | all v1 dirs; also `real-time-events-service-v2` is *v1-style* despite the name — rewrite on the new layer |
| Advanced filter ×3 copies (incl. two 900–1,130-line AQL parsers) | single new `AdvancedFilterBlock` | all three |
| `dataTimeRange` vs `dataTimeRange-v2` | one new block | both |
| `ResizableSplitter.vue` ×2 | webkit `scroll-area`/CSS or one block | both |
| 12 × `views/<domain>/v6/` forks + `EdgeApplications/V3/` | v6 side | legacy sides |
| `RealTimeEvents` vs `RealTimeEventsV2` | V2 | V1 + `realTimeEventsVersionGuard` |
| Dead code: `invalid-data-structure-error copy.js`, 8 zero-reference template blocks (`steps-block`, `single-block`, `search-block`, `page-heading-block-tabs`, `banner-full-block`, `main-menu-block`, `activity-history-block`, `dialog-onboarding-scheduling`), `temp/` dirs inside advanced-filter, v1 dirs with no consumers (`custom-pages-services`, `edge-functions-services`, `workload-deployment-service`) | — | all |

### 5.2 Patterns that break under Nuxt (or are just wrong)

| Trap | Where | Replacement |
|---|---|---|
| Route `props` inject service functions (76 sites; router imports 26 v1 service namespaces) | `src/router/routes/**` | pages call service composables; routing stays declarative |
| Flag-forked component at one path (`v6/EditView.vue` vs `TabsView.vue` chosen in route def) | edge-application, workload, etc. | not ported (winner-only scope); cutover rule in roadmap |
| 8 sequential guards with short-circuit + `inject('tracker')`/`useRouter()` outside components | `src/router/hooks/` | ordered Nuxt global middleware + composables via `useNuxtApp` |
| Module-level singletons (`_flags` ref, `themeApplied`, `hasPrefetched`, BroadcastManager) & `setActivePinia()` | `user-flag.js`, guards, `sessionManager` | Pinia stores + plugins; SSR-safe by construction (lint-enforced) |
| 104 direct `localStorage`/`window` refs across 26 shared files; services opening windows; storage access inside service utils | `helpers/`, `services/v2/base/`, stores | single `safe-storage`/`useClientEnv` composables; banned elsewhere by lint |
| `app.config.globalProperties.HelpCenterServices` / `$tracker` / `$sentry` | `main.js` | typed plugin injections / composables |
| Presentation baked into adapters (`severity:'success'`), service→service fan-out inside `adapt()` | v1 services | pure adapters (lint rule exists — keep it `error`) |
| Feature flags read inside services/adapters (10 files) | `v2/workload`, `digital-certificates`, `variables` | flags resolved in composables/pages; services stay flag-free |
| Monaco loaded from jsdelivr CDN | `main.js` | bundle `monaco-editor` locally (CSP + supply chain) |
| Pinia misuse for page state (6 stores: `release` 705 lines, `deploy` persisted, `solution-create`, `edge-dns`, `purge`, `graphql-query`) | `src/stores/` | component/feature state or query cache; only true globals become stores |
| **Committed `.env` with live-looking Sentry auth token + SSO UUIDs + Stripe test key** | repo root | never commit `.env`; CI secrets + gitleaks from Phase 0; rotate the exposed token |
| `build.sourcemap: 'inline'` fallback in prod builds | `vite.config.js` | hidden sourcemaps, upload to Sentry only |
| No global error boundary (`app.config.errorHandler` absent) | `main.js` | Nuxt `error.vue` + `NuxtErrorBoundary` + Sentry |
| Env branching duplicated ×3 (`vite.config.js`, `azion.config.mjs`, `get-environment.js`) | — | one env module + Nuxt runtimeConfig conventions |
| Yarn 1 + 27 `resolutions`; separate `storybook/` sub-package with its own lockfile | root | pnpm 11 workspace, single lockfile, `pnpm.overrides` only when justified |

### 5.3 Upgrade cliffs measured (old → new)

| Dep | Old | New (verified 2026-08-11) | Breaking notes |
|---|---|---|---|
| Vue tooling | Vite 6 + `@vitejs/plugin-vue` 4 | Nuxt 4.5.2 (Vite 7 internally) | — |
| webkit | 3.0.5 (PrimeVue) | 4.3.0 (PrimeVue-free) | component APIs differ; ships raw `.vue` → `build.transpile` |
| Tailwind | 3.3 (`important: true`, config file) | 4.3.3 (CSS-first, `@theme`) | theme 4.3 **requires** TW4; TW3 is the broken combo |
| Pinia | 2 + persistedstate 3 (`paths`) | 4 + persistedstate 4.7 (`pick`) | API rename |
| Sentry | `@sentry/vue` + manual vite plugin | `@sentry/nuxt` 10.70 (module) | server config only if SSR later |
| Forms | yup 1.7 (+129 files of schemas) | zod 3.25.x + `@vee-validate/zod` 4.15 | zod **must stay v3 line**: `@vee-validate/zod` peers `zod ^3.24`, webkit itself bundles zod 3 |
| Storybook | 8.6 (own sub-package) | 10.5.7 standalone app | test-runner → `@storybook/addon-vitest` |
| ESLint | 8-era `.eslintrc*` ×3 | ESLint 10 flat config | `eslint-plugin-xss` (0.1.12) is dead/incompatible — replace with `eslint-plugin-no-unsanitized` 4 + `vue/no-v-html` |

## 6. Platform decisions to resolve (§4.5)

All resolved and justified in `01-architecture.md`; index:

| Decision | Choice (short) |
|---|---|
| SSR vs SPA vs hybrid | **SPA-first (`ssr: false`), SSR-ready by lint discipline** (user-confirmed) |
| Auth/session | Keep HttpOnly-cookie model + edge rewrite; Nuxt middleware `auth.global` + account bootstrap; no token in JS |
| Data fetching | Services layer + TanStack Query v5 (port v2 base); **not** `useFetch`/`useAsyncData` |
| State | Pinia 4 for true globals only (account, theme, chrome); server state lives in query cache |
| i18n | English-only, no vue-i18n (matches current product; retrofit path documented) |
| Feature flags | `client_flags` → `useFlag()` + `definePageMeta({ flags })` + global middleware |
| RBAC | Backend-enforced; minimal `usePermission()` façade; no client authz matrix |
| Errors/loading | Ported `ApiError` (ex-ErrorHandler) + webkit toasts + `error.vue` boundary + query-state skeletons |
| Routing/code-splitting | File-based pages mirroring current URLs 1:1 (strangler requirement); per-page chunks by default |
