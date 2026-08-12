# 05 — AI-First Repository Design

> Requirement: agents work here autonomously and safely, as a first-class property of the repo. Every subsection ends in a concrete artifact with a path — if a requirement doesn't map to a file, a command or a CI job, it doesn't exist. The webkit ecosystem already ships agent infrastructure (`webkit-mcp` MCP server, `catalog.json`, a `.claude/` rules bundle installed by `webkit init`) — we build on it instead of inventing a parallel one.

## 7.1 Context for agents

| Artifact | Content contract |
|---|---|
| `CLAUDE.md` (root) | ≤ ~150 lines: what the repo is, the command surface (`06-conventions.md` §6), the dependency matrix (one diagram), "generate, don't hand-write" rule, links to per-package contexts and ADR index. **No prose that lint already enforces** — context budget is spent on what machines can't check. |
| `AGENTS.md` (root) | same content, tool-agnostic filename; one is a pointer to the other to avoid drift. |
| `apps/console/CLAUDE.md` | page/feature anatomy, middleware order semantics, how to add a route (meta contract), strangler URL-parity invariant ("never change a public URL"). |
| `services/CLAUDE.md` | the disposal boundary in 10 lines; "types come from `gen:api-types`, never hand-write DTOs"; ApiError contract; aux vs v4 domain distinction. |
| `packages/components/CLAUDE.md` | "check webkit first" (link `catalog.json` + webkit MCP), block = component+story+test+export quartet, no services/stores imports. |
| `docs/adr/NNNN-*.md` | MADR-short (Context / Decision / Consequences, ≤ 1 page). Seeded in Phase 0 with the ~12 decisions from `01-architecture.md` so agents can cite *why*, not just *what*. New ADR via `pnpm gen:adr`. |
| Predictable placement | guaranteed by generators + `06-conventions.md` §2 layout; the test is behavioral: an agent told "add a field to Edge DNS zone form" must need zero placement questions — Phase 2 exit criterion. |

## 7.2 Recurring-task automation

**Deterministic generators are the API for boilerplate** (plop 4, `tools/generators/`): `gen:service` (contract + impl + composable + MSW handlers + tests), `gen:crud` (pages + feature folder + schemas + e2e journey skeleton + parity checklist stub), `gen:component` (block + story + play-test + export), `gen:adr`. Agents (and humans) invoke generators and then edit — hand-rolled boilerplate is a review flag. Generator templates are updated whenever a pattern changes; **the generator is the executable spec of the pattern** (Phase 2 exit criterion regenerates Edge DNS to prove it).

**Slash commands / skills** (`.claude/commands/*.md`, thin wrappers that chain generator + verify + context):

| Command | Flow it encodes |
|---|---|
| `/new-service <domain>` | gen:service → wire types from generated spec → `pnpm verify` |
| `/new-crud <domain>` | gen:crud → fill zod schemas from contract → stories/tests → verify |
| `/new-component <name>` | check webkit catalog first (MCP) → gen:component → story+a11y green |
| `/migrate-route <prefix>` | the wave playbook: freeze check → gen → parity checklist → smoke → cutover PR for the edge rule |
| `/fix-sentry <issue-id>` | the §7.4 runbook |
| `/parity-check <prefix>` | run checklist diff against old console (screens, URLs, tracker events) |

**MCP servers** (`.mcp.json` at root): `webkit-mcp` (ships inside `@aziontech/webkit` — component/props/catalog queries, replaces guessing the DS API), **Sentry MCP** (issue → stack/breadcrumbs/tags for §7.4), **GitHub MCP** (issues/PRs/CI runs), **Figma MCP** (design references when building blocks). Each is listed with its purpose in CLAUDE.md so agents know when to reach for which.

## 7.3 Fast, deterministic feedback loops

| Command | Budget (warm turbo cache) | Notes |
|---|---|---|
| `pnpm lint` / `typecheck` | < 30 s | affected-only via turbo |
| `pnpm test:unit` | < 2 min | node + browser projects; MSW, no network |
| `pnpm test:e2e:smoke` | < 5 min (Phase 2), < 8 min (wave A+) | `@smoke` grep, real browser, stage-shaped mocks |
| `pnpm verify` | < 5 min | the full gate; identical command locally and in CI (`pre-merge` runs exactly this) |

Enforced properties:

- **Actionable failures:** custom lint rules must say the fix, not just the sin ("Import `useEdgeDns` from `@console/services` instead of reaching into impl/ — see services/CLAUDE.md"). Error-message quality is reviewed like UI copy.
- **Fail fast:** turbo stops on first failed task; vitest bails per-file; e2e smoke runs only when the app is in the affected graph.
- **Zero tolerated flakiness:** a flaky test is quarantined-or-deleted within a day (`06-conventions.md` §5); retries are not enabled in CI for unit/component tests.
- **Test strategy an agent can execute alone:** unit = pure logic in node env; component = Storybook play-tests via `@storybook/addon-vitest` (writing the story IS writing the test); e2e = Playwright journeys from the `gen:crud` skeleton. **Mocks are not invented:** MSW 2.15 handlers in `@console/testing` typed against `contracts/generated/api-v4.d.ts`, fixtures checked by the same types — when the platform spec drifts, the `api-types-drift` CI job says so explicitly (`03-services.md` §4).
- **Coverage ratchet** (ported mechanism): floors only rise; new-code target 90%; agents get a binary local signal (`pnpm check:ratchet`) instead of a subjective "add more tests".

## 7.4 Sentry → automated fix pipeline

Instrumentation (Phase 1) that makes errors *actionable by an agent*:

| Piece | Artifact |
|---|---|
| Release = git SHA, hidden sourcemaps uploaded on every deploy | `@sentry/nuxt` module config in `nuxt.config.ts` + deploy workflow |
| Tags on every event: `domain` (route prefix), `feature`, `owner` | Sentry `beforeSend` reading route meta + a build-time `owners-map.json` generated from `CODEOWNERS` |
| Breadcrumbs that reconstruct state: route changes, query-key fetches/mutations (+ status), tracker events, ApiError details (endpoint, status, JSON:API codes) | query-client + router + analytics hooks in the Sentry plugin |
| Replay with the ported masking policy (passwords, HMAC keys, `[data-sentry-mask]`) | Sentry plugin config |
| Alert routing: new issue / regression → GitHub issue labeled `sentry` + `domain:<x>` via Sentry's GitHub integration | Sentry project config (documented in `docs/RUNBOOK.md`) |

**Agent runbook** (`.claude/commands/fix-sentry.md`): (1) pull issue via Sentry MCP — stack, tags, breadcrumbs, release SHA; (2) map stack through sourcemaps to files (release = SHA → exact code state); (3) **write the failing test first** — unit/component repro from breadcrumb state, MSW fixture from the ApiError payload; (4) fix until `pnpm verify` green; (5) PR titled `fix(<domain>): <issue title>` linking the Sentry issue, including the repro test; (6) humans own merge + the Sentry resolve. Guardrails §7.6 apply (an agent fixing an auth-middleware crash produces a PR a human must approve — CODEOWNERS makes that structural).

## 7.5 Performance & health

| Concern | Mechanism (artifact) |
|---|---|
| Web vitals in prod | Sentry browser tracing (LCP/CLS/INP per route, tagged by `domain`) — dashboards linked in RUNBOOK |
| Bundle budgets | `size-limit` 13 in `apps/console`: entry chunk + top route chunks; baseline recorded at Phase 2 exit, budget = baseline +10%; **CI fails on breach** (`pre-merge` job `budgets`) |
| Lighthouse | `@lhci/cli` 0.15 against built preview of `/login`-shaped public page, home, one list route; assertions (perf ≥ 85, a11y ≥ 95) **fail the build**; runs in `pre-merge` when app affected + nightly |
| Heavy-lib discipline | Monaco/c3/OpenLayers/exceljs/jszip only via dynamic import inside client-only islands (lint: `no-restricted-imports` top-level ban in pages/features) |
| Synthetic availability | scheduled Playwright prod smoke (login-less public probe + cookie-authenticated journey with a vault-stored test account) every 15 min via GitHub Actions cron; failure → Slack + PagerDuty webhook (`.github/workflows/synthetic.yml`) |
| Budget-burst runbook | `docs/RUNBOOK.md`: bundle analyzer artifact is attached by the failed CI job → identify the import → island-ify or split → if legitimate growth, budget bump is a PR touching `size-limit` config, which is CODEOWNERS-protected (a human decision by construction) |

## 7.6 Guardrails — what agents cannot change alone

No DB migrations exist in this repo; the equivalent high-blast-radius surfaces are auth, money, the edge config, CI and the guardrails themselves.

| Protected surface | Path globs | Mechanism |
|---|---|---|
| Auth/session | `apps/console/app/middleware/*`, `services/src/impl/http/aux-auth/**`, session store | CODEOWNERS → platform team; branch protection requires code-owner review |
| Edge/deploy config (the strangler!) | `apps/console/azion.config.mjs`, `azion/**` | CODEOWNERS + cutover PRs use a dedicated template with rollback plan |
| CI & guardrails | `.github/**`, `CODEOWNERS`, `packages/config/**` (lint laws), `size-limit`/LHCI configs, coverage/ratchet baselines | CODEOWNERS; plus `guardrails` CI job: if PR author is a bot/agent identity (or PR labeled `agent`) **and** diff touches protected globs **without** a human-approved review, the job fails — enforcement is a required check, not etiquette |
| Security deps & secrets | `pnpm-workspace.yaml` catalogs, lockfile bumps of auth/crypto/payment deps; any `.env*` | Renovate-style PRs land unprotected deps only; gitleaks blocks committed secrets (the old repo shipped a live `.env` — never again); secrets exist only in CI/vault |
| Billing/payment code | `features/billing/**`, Stripe integration | CODEOWNERS → billing owner; wave E's extended bake applies to changes too |

How CI validates an agent PR (the same `pre-merge` humans get, plus): `guardrails` job above; PR template checklist (generator used? verify output pasted? Sentry/issue linked?); commit trailer `Co-Authored-By` retained for auditability. Agents get **write access to branches, never to `main`** — merge is always a human act on a green, reviewed PR.
