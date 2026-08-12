# 03 — Services Layer (disposable by design)

> `@console/services` exists to be deleted. Its job is to give screens a stable, typed, cache-aware data API **today**, shaped so that the future official Azion TypeScript SDK replaces its internals **without touching a single page**. This document defines the boundary that makes that true, the type-generation pipeline that keeps it honest, and the migration order.
>
> User decision (2026-08-11): contracts mirror the Azion SDK domain shape; v3-only and auxiliary APIs live explicitly outside the disposal boundary.

## 1. The ecosystem this plugs into (verified from the repos)

| Repo | Role | Key facts |
|---|---|---|
| `aziontech/azionapi-v4-processor` (local: `/Users/unknown1/www/azion/azionapi-v4-processor`) | OpenAPI source of truth | 20 per-API specs in `schemas_processor/sources/*-api.yaml` (~2.7 MB) merged by `build.sh <env>` into one `openapi.yaml`; 17 quality fixers, several explicitly protecting TypeScript generation (`fix_primitive_oneof`, `fix_description_comments`, `fix_complex_defaults`); schema names normalized (`EdgeApplication`→`Application`); published per env to Edge Storage buckets `storage-open-apiv4-{stage,preview,prod}` and served at `{stage-,preview-,}api.azion.com/v4` |
| `aziontech/azionapi-v4-go-sdk-dev` (local: `…/azionapi-v4-go-sdk-dev`) | Generated Go SDK | openapi-generator-cli 7.23.0; `azion-api` package = merged spec: **89 services / 501 operations** across every console config domain (applications, firewalls, wafs, functions, dns, certificates, network_lists, custom_pages, connectors, workloads+deployments, data_stream, sql, storage, purge, kv, vcs, identity, auth, accounts, metrics, orchestrator, billing/payments…) |
| The handoff | `processor/.github/workflows/generate.yml` | matrix `types: ["go"]` → cross-repo PR into `azionapi-v4-<type>-sdk-dev`. **Adding `"typescript"` to that matrix is the designed extension point** for an official TS SDK — the end state this layer is built to be replaced by |

Implications adopted here:

1. **DTO types are generated, not written.** `openapi-typescript` 7.13 consumes the same processed spec that generates the Go SDK, so our types carry the *same normalized names* the future TS SDK will have. The disposal swap becomes a rename, not a re-mapping.
2. **Do not imitate the Go pre-patch.** The Go pipeline's `scripts/patch.py` strips `enum`/`default`/`pattern` before generating; for TS that would destroy union literal types. We consume the spec as processed, no local patching.
3. **Environment surface**: generate from the **staging/preview** build (all 20 APIs); the `prod` build filters to a subset and would hide domains the console needs pre-GA.

## 2. Package anatomy

```
services/
├── package.json            # name: @console/services — exports "." ONLY (plus ./testing for fixtures)
├── src/
│   ├── index.ts            # THE PUBLIC API: domain composables, ApiError, DTO types, queryKeys
│   ├── contracts/          # ── the stable side of the boundary ──────────────────────────
│   │   ├── generated/
│   │   │   └── api-v4.d.ts # openapi-typescript output — committed, refreshed by gen:api-types
│   │   ├── <domain>.ts     # domain interface + DTOs (types re-exported/narrowed from generated)
│   │   └── errors.ts       # ApiError (port of v2 ErrorHandler, JSON:API-aware)
│   ├── composables/
│   │   └── use-<domain>.ts # TanStack Query hooks typed by contracts — what pages call
│   ├── core/               # ported v2 base, as TS: query-keys factory, cache presets,
│   │                       # IndexedDB persister (+encryption), abort manager,
│   │                       # broadcast/cache-sync, SSE client, http client (ofetch)
│   └── impl/               # ── the disposable side ─────────────────────────────────────
│       ├── http/<domain>/  #   <domain>-api.ts (thin fetchers) + <domain>-adapter.ts (pure)
│       └── sdk/            #   (future) same interfaces implemented over the official TS SDK
└── vitest.config.ts
```

A domain module, end to end (illustrative):

```ts
// contracts/edge-dns.ts
import type { paths } from './generated/api-v4'
export type Zone = paths['/edge_dns/zones/{id}']['get']['responses']['200']['content']['application/json']['data']
export interface EdgeDnsService {
  listZones(params: ListParams): Promise<Paginated<Zone>>
  getZone(id: number): Promise<Zone>
  createZone(input: ZoneInput): Promise<Zone>
  // …
}

// composables/use-edge-dns.ts — the ONLY thing pages import
export function useEdgeDns() {
  const svc = injectService<EdgeDnsService>('edge-dns')   // resolves impl/http today, impl/sdk tomorrow
  return {
    listZones: (p) => useQuery({ queryKey: queryKeys.edgeDns.list(p), queryFn: () => svc.listZones(p) }),
    createZone: () => useMutation({ mutationFn: svc.createZone,
      onSuccess: invalidate(queryKeys.edgeDns.all) })
  }
}
```

## 3. The disposal boundary, explicitly

**Stable (survives the swap):** `contracts/*` (interfaces, DTO types, `ApiError`), `composables/*` (hook signatures), `core/query-keys` (cache identity), the package name and its root export.

**Disposable (deleted by the swap):** everything under `impl/http/` — fetchers, URL building, JSON:API unwrapping, adapters. When `azionapi-v4-typescript-sdk-dev` exists, each domain swap is: implement `EdgeDnsService` over the SDK client in `impl/sdk/edge-dns.ts`, wrap SDK errors into `ApiError`, flip the DI registration, delete `impl/http/edge-dns/`. Pages, tests of pages, stories: untouched.

**Coupling points deliberately prevented** (each is a lint rule or a structural fact):

| Leak this boundary blocks | Mechanism |
|---|---|
| Pages importing fetchers/adapters directly (old console: 8 views import `@/services/axios`) | package exposes only `"."` in its `exports` map + `boundaries/no-private` |
| Response shapes leaking transport details (JSON:API envelopes) into props | adapters unwrap to contract DTOs; contract types come from the spec's *data* schemas |
| UI concerns inside the layer (old adapters returned `severity: 'success'`) | `pure-adapters` rule (`error`), no webkit/vue-component imports allowed in `services/` |
| Feature flags steering service behavior (10 files today) | `no-flags-outside-app` rule; variants are separate contract methods chosen by the caller |
| Cache keys invented ad hoc per page | only `core/query-keys` may create keys (factory port of the 691-line v2 `queryKeys.js`) |

**Outside the disposal boundary — permanent hand-written HTTP** (the SDK will never cover these; keeping them out preserves the "mechanical swap" guarantee): SSO/auth session endpoints (`/api/token*`, `/logout`, switch-account), GraphQL engines (events, metrics beholder, accounting/billing, cities), SSE (`/sse` beholder), template-engine / script-runner / marketplace app-APIs, webhook feedback, Stripe. These live in `impl/http/aux-*` under the same contracts/composables discipline — same DX, no swap expectation.

## 4. Type generation pipeline (`pnpm gen:api-types`)

1. Fetch the processed spec for the target env (default `stage` bucket object `openapi.yaml`; overridable to a local `azionapi-v4-processor` build via `--spec-path` for pre-release work).
2. `openapi-typescript openapi.yaml -o services/src/contracts/generated/api-v4.d.ts` (types only — no runtime client; the runtime stays ofetch inside impl, so the generated artifact is trivially deletable too).
3. Commit the output. CI job `api-types-drift` re-runs generation weekly + on demand and fails with a diff when the platform spec moved — the successor of the old console's hand-maintained `openapi-drift-engine` + 16 yup schemas, with zero hand-maintained schema code.
4. MSW fixtures in `@console/testing` are validated against the same generated types at compile time, so mocks can't drift silently either.

Runtime validation policy: adapters `zod.parse` responses in dev/test builds (loud, early), passthrough with type assertion in production (no double-parse cost on hot paths).

## 5. What ports from v2, what dies

| v2 asset (old repo) | Verdict |
|---|---|
| `BaseService`/`httpService`/`httpClient` (axios singleton) | replaced by ofetch client in `core/http` + per-domain fetchers; global-axios mutation pattern dies |
| `ErrorHandler` (JSON:API messages, toast methods) | **ported** as `ApiError` — minus the toast methods (UI bridge moves to `useApiToast()` in the app; errors don't render themselves) |
| `queryKeys.js` factory (691 lines) | **ported** as typed factory; keys namespaced per domain |
| `queryOptions.js` cache presets, IndexedDB persister + encryption, BroadcastChannel sync, cache-sync invalidation map, `abortManager`, `sse-client` | **ported** as `core/*` (TS) |
| `sessionManager` (login/logout lifecycle, 22 prefetches) | logic moves to app session store + a `prefetch registry` in core; prefetch list re-audited (22 blind prefetches on login is a perf smell — each must justify itself against the wave that migrates it) |
| v1 everything (`AxiosHttpClientAdapter`, string-throw errors, 52 `make-*-base-url` files, service-calls-service adapters) | not ported |
| `real-time-events-service-v2` (70 files, v1-style despite the name) | rewritten as `aux-events` GraphQL module when wave C arrives |
| contract-drift engine + yup schemas | replaced by §4 |

## 6. Migration order (aligned with `02-roadmap.md` waves)

| Order | Wave | Service domains to build | Notes |
|---|---|---|---|
| 1 | W1 | `core/*`, `aux-auth` (verify/refresh/logout/identity), `account` | session bootstrap needs these; login UI itself stays in old console until wave F |
| 2 | W2 | `edge-dns` | the pattern-proving slice; first generated-types consumer |
| 3 | A | `personal-token`, `credentials`, `users`, `teams`, `purge`, `data-stream`, `orchestrator` (edge-node/services — **v4 target**, code currently on v3), `edge-pulse` | mostly small; orchestrator swaps v3→v4 endpoints per the spec |
| 4 | B | `workloads`, `deployments`, `release-impact`, `applications`, `firewalls`, `wafs`, `functions`, `digital-certificates`, `network-lists`, `custom-pages`, `connectors`, `variables`, `environments` (aux) | the big config surface; heaviest adapters (edge-sql-style 500-line transforms are a smell to fix, not port) |
| 5 | C | `aux-events` (GraphQL), `aux-metrics` (GraphQL beholder), `metrics-api` (dashboards CRUD, v4), `activity-history` | GraphQL stays permanent-aux |
| 6 | D | `edge-sql`, `edge-storage` (+ kv) | large payload/object handling; SSE progress |
| 7 | E | `billing` + `payments` (v4 partial + GraphQL accounting aux + Stripe), `identity-providers`, `mfa`, accounts-management | mixed v4/aux |
| 8 | F | `aux-sso-login` (login/signup/password/MFA flows), `vcs`, `marketplace`, `template-engine`/`script-runner` (aux), `ai` (SSE) | closes the auth surface last, per strangler plan |

Definition of done per domain: contract + impl + composables + MSW handlers + unit tests green + consumed by its wave's pages + zero direct fetch calls outside impl.

## 7. Upstream ask (tracked in 07-open-questions)

Propose adding `"typescript"` to the processor's generation matrix (creating `azionapi-v4-typescript-sdk-dev`). Until that SDK exists and stabilizes, this package is the console's SDK; the moment it ships, §3 says exactly what gets deleted.
