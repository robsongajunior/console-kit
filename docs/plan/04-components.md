# 04 — Components: gap analysis & build catalog

> Rule: **nothing that exists in `@aziontech/webkit` 4.3.0 gets reimplemented.** Webkit exports 218 entries (185 components across actions/inputs/data/content/overlays/navigation/feedback/layout + 5 app templates + composables). Every block below that we build ourselves carries a justification for why it is not webkit's job. Old-console blocks are **rebuilt on webkit 4 APIs, not ported** — the old code is PrimeVue-based (webkit 3) and dies with it.

## 1. Old console → webkit 4.3 direct mapping (use as-is)

| Old console (path · usage count) | webkit 4.3 replacement |
|---|---|
| PrimeVue DataTable inside `list-table/` | `table-root/-header/-body/-row/-cell/-sort-button/-toolbar/-search/-filter/-applied-filters/-column-selector/-export/-refresh-button` (TanStack-backed) + `paginator-*` |
| All PrimeVue inputs/dropdowns/switches across 437 views | `input-*`, `textarea`, `checkbox`, `radio-*`, `switch`, `chip`, `select-*`, `multi-select-*`, `calendar-*`, and the 13 vee-validate `field-*` wrappers |
| PrimeVue Dialog/Sidebar/OverlayPanel | `dialog-*` (8 parts), `drawer-*` (8), `popover-*` (8), `panel-*` (3) |
| `toast-block/index.vue` (custom renderer + severity policy) | `toast` compound + `ToastPlugin`/`useToast` — severity/lifetime policy moves to the `useApiToast` bridge, not a fork of the renderer |
| `create-modal-block/` (574 lines, "Create ▾" global picker) | `command-menu-*` (7 parts) |
| `src/layout/*` navbar/sidebar/menus (15 files) | `global-header`, `sidebar-*`, `menu-*`, `platform-shell` template |
| `page-heading-block` breadcrumbs, tabs | `breadcrumb`, `breadcrumb-item`, `tab-view` |
| `StatusTag`, `InlineTag`, `CurrentBadge` | `tag`, `badge`, `status-indicator` |
| `skeleton-block` primitives, `loading-block` | `skeleton`, `spinner`, `progress-bar` |
| `MessageCard`, `info-banner`, `message-notification` | `message`, `empty-state` |
| `dialog-copy-key`, copy cells | `copy-button`, `dialog-*` |
| RTE/log drawers' code and log surfaces | `code-block`, `log-view-*` (4 parts) |
| `CollapsibleCard`, `GroupedList`, detail rows | `accordion-*` (4), `item-*` (11), `card-box` |
| Sign-in/up scaffolding, deploy/plan success screens | `sign-up-card`, `onboarding-form`, `deploy-success`, `plan-success` templates |
| Password rules checklist (signup/reset) | `password-requirements` util |
| Release tree visualization (`ReleaseCompositionTree`) | `flow-root/-node/-parallel/-anchor` (evaluate in wave B; fallback is custom within the versioning feature) |
| 403/404 art | `svg/error-403`, `svg/error-404` |

## 2. Upstream candidates (contribute to webkit later — pointer only, per non-goals)

| Candidate | Evidence it is generic |
|---|---|
| **Advanced filter block** (field rows + operators + applied-filter tags + AQL text mode) | exists 3× in the old console with 900–1,130-line parsers; webkit already owns `table-filter`/`table-applied-filters` primitives it should compose with |
| **Date-time-range picker** with quick presets + auto-refresh interval | generalizes webkit's `calendar-preset`; every observability product needs it |
| **Info-label pair primitives** (`big-number`, label/value text rows) | generic read-only display, complements `item-*` |
| **One-time secret reveal dialog** | tokens/keys UX, product-agnostic |

Until upstreamed they live in `@console/components` with webkit-compatible APIs so the future move is a re-export swap.

## 3. `@console/components` build catalog (the platform layer)

Everything here is **console-platform composition**: it encodes how *this product* does lists, forms and pages — service contracts, tracker events, query states — which a general-purpose DS must not know about. That is the gap justification pattern; per-block specifics below. All blocks: props/emits typed, no store access, no service imports (data arrives via props/injected fetchers — boundary rules `01-architecture.md` §2).

| Block | Replaces (usage today) | Composes | Why not webkit |
|---|---|---|---|
| **`ListTableBlock`** | `ListTable.vue` + 753-line `useDataTable` (51 screens) | `table-*`, `paginator-*`, `empty-state`, `skeleton` | webkit gives table mechanics; this block binds them to the **console data contract**: server-side pagination/sort/filter params, TanStack Query result states, row actions with confirm flow, CSV/XLSX export, column-set persistence (user preference), tracker events |
| **`FormBlock`** (`create`/`edit` variants) + **`FormActionBar`** | `create-form-block` (136) + `edit-form-block` (34) + `action-bar-block` (77) | `field-*`, `button`, `panel` | owns the console form lifecycle: zod schema → vee-validate context, load→`setValues` for edit, submit→service mutation, success toast + redirect policy, dirty tracking, scroll-to-first-error, skeleton while loading. Product workflow, not DS material |
| **`EntityDrawerBlock`** (create/edit) + **`InfoDrawerBlock`** | `create-drawer-block` (29), `edit-drawer-block` (15), `info-drawer-block` (10) | `drawer-*` + `FormBlock` internals | same lifecycle in drawer form-factor, incl. "created from within another form" flows (e.g. create certificate while editing workload) |
| **`PageShellBlock`** | `content-block` (117) + `page-heading-block` (108) | `platform-shell`, `overline`, `link`, `breadcrumb` | standard page skeleton: heading, description, docs link (documentation catalog), primary actions, breadcrumb wiring from route meta |
| **`DeleteDialogBlock`** | `delete-dialog` + `useDeleteDialog` | `dialog-*`, `field-text` | type-resource-name-to-confirm destructive flow + mutation + toast + query invalidation |
| **`DangerZoneBlock`** | `danger-card-block` | `card-box`, `button` | destructive-actions card wired to `DeleteDialogBlock` |
| **`FormSkeleton`** | `skeleton-block/*` (8 files, 25 uses) | `skeleton` | renders a skeleton **from the zod schema shape** — schema-driven, console-specific |
| **`UnsavedChangesGuard`** | `dialog-unsaved` + 3 composables (16 uses) | `dialog-*` | binds form dirty state to Nuxt route leave + `beforeunload` |
| **`AdvancedFilterBlock`** | 3 divergent copies + 2 AQL parsers | `popover`, `field-*`, `calendar-*`, `table-applied-filters` | single reimplementation, one parser, URL-serializable filter state (upstream candidate, §2) |
| **`DateTimeRangeBlock`** | `dataTimeRange` ×2 | `calendar-*`, `segmented-button` | presets + auto-refresh interval (upstream candidate, §2) |
| **`JsonFormsRenderers`** | `jsonform-custom-render/*` (10 files, vue-vanilla based) | `field-*` | see §4 |
| **`CodeEditorBlock`** | CDN Monaco via `@guolao/vue-monaco-editor` | — (client-only island) | Monaco bundled locally (CSP/supply-chain), lazy chunk, theme sync with `data-theme`, diff mode for versioning screens |
| **`ChartBlock`** + **`MapChartBlock`** | `graphs-card-block` (31 files: 11 c3 types + OpenLayers) | `card-box` | c3/OpenLayers wrappers as client-only lazy islands with the console's report-spec props; charts lib replacement is out of scope (debt note in 07) |
| **`CopyKeyDialog`** | `dialog-copy-key` | `dialog-*`, `copy-button` | one-time secret reveal (upstream candidate §2) |

**Deliberately NOT a package component:** the **versioning/release framework** (`version-shell-block` 11 files + `release-composition` 45 + `composables/versioning` 26 + 705-line store). It is domain logic spanning 9 resources, coupled to deployment services and release state. It becomes `apps/console/app/features/versioning/` (a feature module with its own composables and machine), rebuilt in wave B on webkit primitives (`flow-*`, `item-*`, `dialog-*`). Extraction to a package happens only if a second app ever consumes it — packages are for reuse, not for prestige.

**Not rebuilt at all** (dead in the old repo — zero references): `steps-block`, `single-block`, `search-block`, `page-heading-block-tabs`, `banner-full-block`, `main-menu-block`, `activity-history-block`, `dialog-onboarding-scheduling`, the two stray `ResizableSplitter`s, `temp/` filter dirs.

## 4. JSONForms strategy (scope decided from evidence)

Current reality: JSONForms 3.8 is used **only** where the form schema arrives at runtime — marketplace/template-engine wizards (`engine-jsonform.vue`) and edge-function/firewall-function **argument forms** (4 screens), with 5 custom vanilla renderers. Every one of the ~170 CRUD forms is hand-built vee-validate.

**Decision — JSONForms only for runtime-schema forms; hand-written (`FormBlock` + zod) for everything else.**
Justification: static CRUD forms gain nothing from a JSON-schema indirection (types, autocomplete and lint all get worse; agents generate `FormBlock` forms from zod schemas trivially via the generator), while dynamic forms *cannot* be hand-written because the schema is data. The 3.8 stack (`@jsonforms/core|vue`) stays; `@jsonforms/vue-vanilla` is dropped in favor of our renderer set.

**Renderer set** (`@console/components/jsonforms`): control renderers wrapping webkit `field-text`, `field-select`, `field-textarea`, `field-password`, `field-switch`, `field-checkbox`, `input-number` + layout renderers (vertical/group) + testers ranked above the defaults. Contract: renderers emit the same value/validation surface `FormBlock` expects, so a JSONForms island can sit inside a hand-written form (exactly the edge-function-args case today).

## 5. Storybook (`apps/storybook`)

| Decision | Value |
|---|---|
| Scope | **`packages/components` only.** Console screens/features are documented by the app itself (routes) and E2E; duplicating them as stories doubles maintenance without users. Webkit primitives already have their own Storybook (webkit.azion.app) — we link, not mirror. |
| Version/framework | Storybook 10.5.7 + `@storybook/vue3-vite` (standalone Vite; the Nuxt module targets app storybooking we're not doing) |
| CSS parity | preview imports the same CSS contract as the app: theme → webkit styles → icons → components `@source` |
| Interaction tests | **stories are the component tests**: `@storybook/addon-vitest` runs play-functions via Vitest browser mode in `pnpm test:unit` — one artifact, doc + test |
| a11y | `@storybook/addon-a11y` with `test.todo→error` promotion: violations fail CI for every block (new code has no excuse; the DS beneath is webkit's a11y responsibility) |
| Data | MSW handlers from `@console/testing` for blocks that demo query states (loading/error/empty) |
| Publishing | deployed to Azion (stage bucket) on merge, same CLI pipeline as the app |

## 6. Build order

Blocks are built when their first consumer wave needs them, never speculatively: W2 (Edge DNS slice) forces `PageShellBlock`, `ListTableBlock`, `FormBlock`+`ActionBar`, `EntityDrawerBlock`, `DeleteDialogBlock`, `FormSkeleton`, `UnsavedChangesGuard` — i.e. the slice proves the whole core kit. Wave A adds `InfoDrawerBlock`, `CopyKeyDialog`, `DangerZoneBlock`; wave B adds `CodeEditorBlock`, `JsonFormsRenderers` (+ versioning feature); wave C adds `AdvancedFilterBlock`, `DateTimeRangeBlock`, `ChartBlock`/`MapChartBlock`.
