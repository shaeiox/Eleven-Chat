# 13 — Architecture Decisions, Constraints, and Limitations

> **Purpose.** Record *why* the system is shaped the way it is, where that "why" is actually
> written down, and which constraints, risks, and technical debt a maintainer should know about.
> Companion to [01 — Architecture](./01-architecture.md).
>
> **Discipline used throughout.**
> - **Verified**: read in source at commit `f28809c` during the research pass behind this guide.
> - **Inferred**: strongly supported by code shape, tests, or adjacent comments, but not stated
>   in writing or not traced end to end.
> - **Unknown**: the behavior is deliberate, but no recorded rationale was found.
>
> Severity and confidence ratings in §4 are **this document's own calibrated judgment**, not
> findings from a security audit, a load test, or an incident. Nothing here has been fixed. The
> remediation notes say what to investigate next; they are not implemented changes.
>
> Cross-references: [08 — Auth & security](./08-auth-security.md) ·
> [02 — Backend](./02-backend.md) · [06 — Business logic](./06-business-logic.md) ·
> [04 — Redis](./04-redis.md) · [`PROJECT_MAP.md`](../../PROJECT_MAP.md) ·
> [`CRITICAL_FLOWS.md`](../../CRITICAL_FLOWS.md)

---

## Contents

1. [Explicitly documented decisions](#1-explicitly-documented-decisions)
2. [Strongly supported inferences](#2-strongly-supported-inferences)
3. [Unknown historical rationale](#3-unknown-historical-rationale)
4. [Technical debt and risk register](#4-technical-debt-and-risk-register)
5. [Corrections to existing documentation](#5-corrections-to-existing-documentation)
6. [Open questions for follow-up](#6-open-questions-for-follow-up)

---

## 1. Explicitly documented decisions

These decisions have a rationale **written down**, either in a repo document or in a code comment
next to the implementation. The "Where written" column is the authority. If you change the
decision, update that location as well.

### D1. Generation is a durable server-side job, separate from the HTTP connection

- **Where written:** `CRITICAL_FLOWS.md` Flow 1, "Why this flow is shaped the way it is". In code:
  `useResumableSSE.ts:884-892` ("Navigation away does NOT abort the generation") and the
  save-before-final comment in `api/server/controllers/agents/request.js:3048-3134`.
- **Decision:** `POST /api/agents/chat` creates a `GenerationJobManager` job and returns at once.
  SSE (`GET /stream/:streamId`) is just one subscriber. Stop, resume, and multi-tab viewing are all
  attach/detach operations on that job.
- **Stated rationale:** survive network blips and tab close, let several tabs watch one
  generation, and share one mechanism between Stop and reconnect.
- **Stated cost:** an explicit job lifecycle (`claimGeneration` → `createJob` →
  `emitChunk`/`subscribe` → `completeJob`/`abortJob`) and ordering invariants that are "preserved
  by convention and a code comment, not a type system guarantee" (`CRITICAL_FLOWS.md`, "Where to be
  extra careful").

### D2. Durable background work uses Mongo-leased queues, not a message broker

- **Where written:** `packages/api/src/agents/triggers/README.md`, "Guarantees": "Mongo owns queue
  state, leases, retry history, and dead letters across restarts and replicas"; at-least-once
  delivery with stable idempotency identity; bounded exponential backoff honoring `Retry-After`;
  durable dead letters; "Each replica retains bounded Mongo polling even when idle."
- **Decision:** schedules (`Schedule`/`ScheduleRun`), trigger deliveries
  (`AgentTriggerDelivery`), and queued follow-up turns (`QueuedTurn`) are all Mongo collections
  advanced by compare-and-swap leases (`claimToken`, `leaseUntil`) and polling loops
  (`setTimeout`-driven). The research pass found no `bull`/`bullmq`/`agenda`/`node-cron`.
- **What is documented vs. not:** the *guarantees* and the *mechanism* are written down. A
  comparison with a broker (why not BullMQ on the Redis that is already optional) is **not**
  written anywhere found. That part is §2 B3.

### D3. The access JWT carries no role. The user is re-read on every request.

- **Where written:** `CRITICAL_FLOWS.md` Flow 2, "Why it's shaped this way"; AGENTS.md, "Backend
  auth cache".
- **Rationale:** a revoked role or a password reset takes effect on the very next request.
- **Consequence written into contributor rules:** any code that changes a user document must
  invalidate the auth user-doc cache (`CacheKeys.AUTH_USER_DOC`, built with
  `throwOnErrors: true` so that failures surface).

### D4. A millisecond-resolution `issuedAtMs` claim decides token retirement

- **Where written:** comment on `generateToken`, `packages/data-schemas/src/methods/user.ts:781-801`.
- **Rationale:** standard `iat` has one-second resolution and cannot order a mint against a
  password reset that lands in the same second.

### D5. Config: strict schema, hot reload that never kills the process, per-principal caching, explicit fail-closed

- **Where written:** `CRITICAL_FLOWS.md` Flow 5; `packages/api/src/app/loader.ts`
  (`ConfigReloadError`); `packages/api/src/app/service.ts:292-296` (`resolveStrictAppConfig`).
- **Decision:** `configSchema.strict()` rejects unknown keys. An invalid config at startup
  exits; an invalid **reload** throws and keeps the last good config. `failClosed` defaults to
  `false` (fail open to the base config) and is turned on per route through
  `strictConfigMiddleware`.

### D6. YAML-derived config stays per container even when Redis is on

- **Where written:** `.env.example`, `FORCED_IN_MEMORY_CACHE_NAMESPACES` (default
  `CONFIG_STORE,APP_CONFIG`): keeps YAML-derived config per container, "safe for blue/green
  deployments".

### D7. One phased shutdown coordinator instead of scattered signal handlers

- **Where written:** module comment, `packages/api/src/app/shutdown.ts:36-42`: competing
  `process.on('SIGTERM')` handlers "race with the HTTP drain because Node dispatches listeners in
  registration order and any one of them can call `process.exit` before the HTTP server has
  finished closing."
- **Decision:** register `pre-drain`/`post-drain` tasks with priorities. A 60 s force-exit timer
  is the outer bound. `api/server/index.js:149-167` documents the 10 s teardown reserve: "Abandoning
  an unrecorded drain fences the next generation permanently."

### D8. The schedule engine refuses to arm on an unsafe topology

- **Where written:** `packages/api/src/schedules/service.ts:98-100` and surrounding comments;
  `api/server/index.js:516-526` (error log: write routes "PERMANENTLY unavailable in this
  process").
- **Decision:** arm only if the job store is Redis-backed or `SCHEDULES_SINGLE_PROCESS=true`.
  The standard entry point starts the engine in every replica, and a process-local store would let
  one replica "reconcile" another replica's generation as interrupted.

### D9. Post-listen initialization failure exits the process

- **Where written:** comment in the `app.listen` callback, `api/server/index.js` (around `:489-496`):
  without explicit handling, a failure would leave the server "listening but only partially
  initialized — passing liveness checks while serving broken requests."

### D10. Route modules are required late

- **Where written:** comment at `api/server/index.js:276-277` (and the `PROJECT_MAP.md` §3
  diagram note): `performStartupChecks()` must run before route modules load, because they build
  rate limiters from config at require time (for example
  `api/server/middleware/limiters/emailChangeLimiter.js:38`).

### D11. Capability helpers are not exported from the middleware barrel

- **Where written:** `api/server/middleware/roles/index.js:1-11`: importing `capabilities.js`
  through the barrel can create "a circular-require that silently returns an empty exports
  object. Always import capability helpers directly."

### D12. Some data is deliberately never cached

- **Where written:** `api/cache/getLogStores.js`. `CacheKeys.PROMPT_GROUPS_ACCESS` is always
  `disabledCache` because "a failed shared invalidation cannot fail closed."

### D13. Edits and regenerations never rewrite history (message tree)

- **Where written:** `CRITICAL_FLOWS.md` Flow 4, "Why a tree, not a list".
- **Decision:** both create sibling nodes under a `parentMessageId`. The active branch is
  client-only view state. **Stated trade-off:** the whole conversation's rows are fetched every
  turn and the branch is walked in application code.

### D14. Single provider dispatch table plus the external `@librechat/agents` runtime

- **Where written:** `CRITICAL_FLOWS.md` Flow 3; doc comment on `providerConfigMap`
  (`packages/api/src/endpoints/config/providers.ts:29-50`), which explains the load-bearing
  VertexAI → `initializeGoogle` mapping; the fail-loud ambiguous custom-endpoint name check
  (`providers.ts:155-200`).
- **Stated cost:** dependence on an external package's internals for SDK calls.

### D15. Contributor architecture rules (AGENTS.md)

These are explicit rules. §4 records where the code does not yet follow them.

| Rule | Text (abridged) |
|---|---|
| Wiring vs. behavior | "`/api` holds wiring, not behavior"; new logic goes in `packages/api` |
| Data boundary | Keep Mongoose types out of exported signatures outside `data-schemas`; "stop widening" the existing leak |
| Configurable levers | New limits, timeouts, toggles, and capabilities get a `configSchema` field with a behavior-preserving default; "env-only switches need a reason" |
| Dependency injection | "Modules take their dependencies rather than reaching for them"; MCP static singletons are "the shape to stop extending" |
| Failure contracts | No `null`/`{message}` for failures; never forward raw `error.message` to clients |
| Client state | New state is Jotai; app-global state is passed in |

### D16. SSRF defense is two layers, with an accepted, documented gap at preflight

- **Where written:** doc comment on `validateEndpointURL`, `packages/api/src/auth/domain.ts:547-551`
  (DNS-rebinding limitation of the preflight check), closed at connect time by
  `createSSRFSafeUndiciConnect` (`packages/api/src/auth/agent.ts:196`).

### D17. The baseline security headers never include CSP

- **Where written:** `packages/api/src/security/headers.ts:34,155`: CSP is always disabled in the
  helmet baseline "so the CSP-independent headers never depend on a directive allow-list staying
  current". A separate nonce-based CSP exists for the SPA shell (`security/csp.ts`).
- **Not written:** why that separate CSP is **off by default**. See R6.

---

## 2. Strongly supported inferences

These patterns are consistent and clearly intentional, but no document or comment states the
reason. Treat the "likely reason" as a hypothesis.

### B1. Modulith with extraction seams, not microservices

- **Evidence:** typed packages with independent builds and tests and no workspace cycles
  (verified from every `package.json`); `PROJECT_MAP.md` §1 says the packages "could be extracted
  later"; DI factories at every seam (`createModels`, `createMethods`, `createAppConfigService`,
  `createStreamServices`).
- **Likely reason:** keep self-hosting as one container plus Mongo while making the code
  modular enough to split later.

### B2. Redis is always optional and sits behind interfaces

- **Evidence:** every Redis consumer has a fallback (`cacheFactory.ts`; `IJobStore` with
  `InMemoryJobStore`/`RedisJobStore`; `ServerConfigsCacheInMemory`/`ServerConfigsCacheRedis`;
  `LeaderElection.isLeader()` returns `true` when `USE_REDIS` is off). A Redis construction
  failure in `createStreamServices.ts:105-111` falls back to memory instead of failing startup.
- **Likely reason:** keep the upstream LibreChat single-container deployment working while
  supporting horizontal scale. The cost of this choice is R10.

### B3. No broker, so background processing works with Mongo alone

- **Evidence:** D2's mechanism. All durable queue state lives in Mongo even when Redis is
  configured. Redis carries only live, TTL-bounded state (`RedisJobStore.ts:1830-1849` TTLs; no
  Redis key holds a permanent record).
- **Likely reason:** follows from B2. A broker would turn Redis into a required, durable
  dependency, and Mongo is already required.

### B4. Background work re-enters through HTTP self-loopback

- **Evidence:** the trigger execution host `fetch`es `POST /api/agents/chat` with a minted token
  and `x-lc-agent-trigger: 1` (`packages/api/src/agents/triggers/host.ts:628-659`), and the
  controller branches on `req._isAgentTrigger` (`request.js:983-1107`).
- **Likely reason:** reuse the single, heavily guarded admission path (authentication, ACL,
  enforced model specs, PII/moderation filters, rate limits, job claim) instead of keeping a
  second, in-process entry into the agent runtime that could drift from it.

### B5. "Agents-first": every chat turn is an agent run

- **Evidence:** ephemeral agents, the `/api/agents/chat/:endpoint` route, a single `Run.create`
  call site (`packages/api/src/agents/run.ts:3145`); the legacy Assistants path is the only other
  model-invoking route.
- **Likely reason:** one execution engine for tools, MCP, subagents, HITL, and usage accounting.

### B6. The database arbitrates concurrency through unique indexes

- **Evidence:** the `ScheduleRun.capacitySlot` unique partial index
  (`scheduleRun.ts:180-186`) and its comment in `capacity.ts:16` ("count active runs, compare to
  cap, then insert" races across schedules); `Schedule.slot` for `maxPerUser`
  (`schedule.ts:320-323`); a unique `deliveryKey` for trigger idempotency.
- **Likely reason:** correct under concurrency across replicas with no distributed lock service.

### B7. Some locks live in Mongo, not Redis

- **Evidence:** `skillSyncStatus.lockOwner/lockExpiresAt`, `openidRefreshFlight.lockExpiresAt`.
- **Likely reason:** they must survive a Redis flush or outage, and Redis is optional (B2).

### B8. Code age decides where code lives (a strangler-style migration)

- **Evidence:** newer subsystems (S3/CloudFront/Mistral OCR storage, user preferences, skills
  import, upload routing) live in `packages/api` behind factories. Older ones (Local, Firebase,
  and Azure storage; legacy Assistants controllers; `BaseClient.js`) are still CJS in `api/`.
  Unsafe error disclosure (R2) clusters in the same older code.
- **Likely reason:** AGENTS.md's rule applies to new work and to code "you touch". There is no
  bulk migration.

### B9. The legacy Assistants path keeps upstream's one-POST SSE model

- **Evidence:** `useAdaptiveSSE.ts:19-41`; the only `require('openai')` calls are in
  `Endpoints/assistants/initalize.js` and `Endpoints/azureAssistants/initialize.js`.
- **Likely reason:** backward compatibility with the upstream Assistants API surface. It is not
  being invested in.

---

## 3. Unknown historical rationale

These are deliberate but carry no recorded "why". Do not guess in code reviews. Ask the
maintainer, or record the answer here once it is known.

| # | Observation | Evidence | Why it matters |
|---|---|---|---|
| U1 | `@librechat/agents` is an external npm package (`^4.0.4`), not a workspace | `api/package.json`, `packages/api/package.json` | Adding a wire protocol or fixing SDK behavior means a separate release cycle |
| U2 | `checkAdmin` still exists and is still exported after admin routes moved to capabilities | `api/server/middleware/roles/admin.js`, `roles/index.js:13-16` | See R1 |
| U3 | Per-user provider keys still use v1 fixed-IV encryption | `packages/data-schemas/src/methods/key.ts:3,121` | See R5 |
| U4 | `cors()` takes no options | `api/server/index.js:360` (identical in `experimental.js`) | See R3 |
| U5 | `/api/auth/refresh` omits `requireSameOrigin`, which its sibling session-minting routes use | `api/server/routes/auth.js:73,81,156,188` | See R4 |
| U6 | The SPA-shell CSP ships disabled (`CSP_ENABLED` unset means off) | `packages/api/src/security/csp.ts:201`; `.env.example:119` | See R6 |
| U7 | `api/server/experimental.js` is a Node `cluster` entry point ("simulate multi-pod environment") that flushes Redis on boot and never arms the schedule engine | `experimental.js:104,111-183,189-190,363` | Its intended status (test harness or future production mode) is unclear |
| U8 | The two Dockerfiles pin different `uv` versions (0.9.5 vs. 0.6.13) | `Dockerfile`, `Dockerfile.multi` | Images can differ in MCP stdio tooling behavior |
| U9 | `package.json` still names the upstream repository URL | `package.json:68` | Tooling or fingerprinting may treat the fork as upstream |
| U10 | The `search/` proof of concept (FerretDB/Postgres/ClickHouse) has no integration path | `search/` ("infra only, no app code"), `PROJECT_MAP.md` | Possible future replacement for Meilisearch |
| U11 | The four `seedDatabase` steps and plugin → skill initialization run serially | `api/models/index.js:21-26`; `api/server/index.js:244-260` | See R14. It is not known whether the order is required. |
| U12 | The transaction-capability probe exists, but deletion cascades use idempotent sequential writes rather than transactions | `packages/data-schemas/src/utils/transactions.ts`; `deleteConvos` comments | Partly explained in comments ("deleting each wave before discovering the next closes the child-creation race"). The general policy is not written down. |

---

## 4. Technical debt and risk register

**Severity scale (this document's judgment):**
- **High**: plausible security or data-integrity impact in a default or common deployment.
- **Medium**: a correctness or operational risk under a specific configuration or a plausible
  future change.
- **Low**: maintainability, consistency, or documentation drift.

**Confidence:** **Verified** means the defect or condition was read in code. **Inferred** means
the impact is reasoned, not demonstrated.

### Summary

| ID | Item | Area | Severity | Confidence |
|---|---|---|---|---|
| [R1](#r1-checkadmin-is-dead-code-and-two-docs-still-describe-it-as-live) | `checkAdmin` dead code; stale docs | Auth | Low | Verified |
| [R2](#r2-raw-errormessage-forwarded-to-clients) | Raw `error.message` sent to clients | Errors / security | Medium | Verified (sites) / Inferred (leak content) |
| [R3](#r3-cors-mounted-with-default-wildcard-options) | `cors()` with default wildcard | Security | Medium | Verified |
| [R4](#r4-apiauthrefresh-lacks-requiresameorigin) | `/api/auth/refresh` lacks `requireSameOrigin` | Security | Low | Verified |
| [R5](#r5-per-user-provider-keys-use-legacy-fixed-iv-aes-cbc) | Per-user keys on v1 fixed-IV AES-CBC | Secrets | Medium | Verified |
| [R6](#r6-csp-implemented-but-disabled-by-default) | CSP built but off by default | Security | Medium | Verified |
| [R7](#r7-business-logic-living-in-api-cjs) | Business logic in `api/` CJS | Maintainability | Medium | Verified |
| [R8](#r8-enforced-agent-id-derived-independently-in-two-places) | Enforced agent id derived twice | Authorization correctness | Medium | Verified (duplication) / Inferred (risk) |
| [R9](#r9-provider-file-size-defaults-hard-coded-outside-configschema) | Provider file-size defaults hard-coded | Configuration | Low | Verified |
| [R10](#r10-multi-replica-correctness-depends-entirely-on-redis-being-configured) | Multi-replica correctness depends on Redis | Operations | **High** (for multi-replica deployments) | Verified |
| [R11](#r11-subagent-executions-are-process-bound) | Subagents bound to one process | Scalability / resilience | Medium | Verified |
| [R12](#r12-redis-cluster-status-guard-is-best-effort) | Redis Cluster status guard is best-effort | Correctness | Low–Medium | Verified (documented in code) |
| [R13](#r13-mcp-static-singletons) | MCP static singletons | Maintainability / testability | Low | Verified |
| [R14](#r14-serial-startup-chain) | Serial startup chain | Startup latency | Low | Verified (shape) / Unknown (cost) |
| [R15](#r15-no-limiter-ahead-of-jwt-verification-on-protected-routers) | No limiter ahead of JWT verification | Availability | Low | Verified (order) / Unknown (mitigation) |
| [R16](#r16-full-conversation-fetched-on-every-turn) | Full conversation fetched on every turn | Performance | Low–Medium | Verified |
| [R17](#r17-no-migration-framework-breaking-index-changes-need-manual-runs) | No migration framework | Operations | Medium | Verified |
| [R18](#r18-very-large-load-bearing-files) | Very large load-bearing files | Maintainability | Medium | Verified |
| [R19](#r19-tenant-header-misconfiguration-fails-open-to-no-tenant-scope) | Tenant header misconfiguration fails open | Multi-tenancy | Medium (multi-tenant only) | Verified |
| [R20](#r20-stream-delta-coalescing-can-drop-unflushed-deltas-on-crash) | Delta coalescing can lose deltas on crash | Streaming | Low | Verified (documented) |
| [R21](#r21-dev-compose-ships-insecure-defaults) | Dev compose ships insecure defaults | Deployment | Low (dev) / High if reused in prod | Verified |
| [R22](#r22-no-encryption-generation-authenticates-ciphertext) | No encryption generation authenticates ciphertext | Secrets | Low | Verified |

---

### R1. `checkAdmin` is dead code and two docs still describe it as live

- **Evidence:** defined in `api/server/middleware/roles/admin.js:3-14` and exported from
  `roles/index.js:13-16`. A repo-wide grep finds **no route that uses it**. Every admin router
  applies `requireCapability(SystemCapabilities.ACCESS_ADMIN)` instead (for example
  `admin/config.js:17,36`, `admin/audit.js:10,27`, `admin/roles.js`, `admin/auth.js:42`).
  `PROJECT_MAP.md` §6 and `CRITICAL_FLOWS.md` Flow 2 item 3 still describe `checkAdmin` as the
  `/api/admin/*` gate.
- **Current impact:** no security impact. The capability check is at least as strict. A
  contributor who follows the docs could add a role-string gate where the codebase has
  standardized on capabilities.
- **Potential future impact:** two admin-authorization models drift apart. A role may hold
  `ACCESS_ADMIN` without being `ADMIN`, or the reverse. Whether the seeded `ADMIN` role gets
  `ACCESS_ADMIN` by default was not traced.
- **Severity / confidence:** Low / Verified.
- **Next step:** confirm `checkAdmin` has no dynamic consumers (tests, plugins), then decide
  whether to remove or deprecate it. Update both docs (§5). Trace the default role seeds for
  `ACCESS_ADMIN`.

### R2. Raw `error.message` forwarded to clients

- **Evidence:** the research pass found **13 files** in `api/` that send `error.message` in a
  response body, bypassing `getSafeErrorMetadata`/`getSafeErrorText`
  (`packages/api/src/utils/errors.ts`). Examples:
  `api/server/controllers/agents/v1.js:916,1010,1048,1428,1721,1744,1907,2228`;
  `controllers/assistants/v1.js:149,168,271,373`; `assistants/v2.js:133,361`;
  `controllers/mcp.js:286,364,482,512,598,677`; `controllers/PluginController.js:49,116`;
  `controllers/tools.js:67,100`; `controllers/ModelController.js:20`; `routes/categories.js:11`;
  `routes/memories.js:260,291,408,430`; `routes/agents/actions.js:82,224`;
  `routes/files/files.js:92,160,346`; `routes/assistants/actions.js:91`. A looser grep at the same
  commit matches 17 files under `api/server`. The extra matches were not reviewed.
- **Current impact:** violates AGENTS.md ("never forward ... arbitrary `error.message` to a
  client"). What leaks depends on the thrown error: Mongoose/driver text, MCP SDK or upstream
  provider text (`mcp.js:280-287` forwards any non-capacity error), and possibly URLs or
  identifiers. This is **Inferred**; no actual secret leak was demonstrated. The leaks cluster in
  older controllers, but `agents/v1.js` is the live agent-builder CRUD controller, not only
  legacy code.
- **Potential future impact:** a dependency upgrade that adds request URLs or headers to error
  messages would silently start sending them to browsers. Clients may also come to depend on
  these unstable strings.
- **Severity / confidence:** Medium / Verified for the sites, Inferred for the sensitivity.
- **Next step:** inventory each site and classify whether the message is app-controlled (such as
  the 409 duplicate case at `v1.js:914`) or arbitrary. Map arbitrary ones to stable codes plus a
  fixed message, following `packages/api/src/user/preferences.ts:70` and
  `api/server/routes/convos.js:231-234`. Add tests asserting that sensitive text never reaches the
  response. Consider a lint rule against `error.message` in `res.*` calls.

### R3. `cors()` mounted with default (wildcard) options

- **Evidence:** `app.use(cors())` with no options, `api/server/index.js:360` (also in
  `experimental.js`). The `cors` package default sends `Access-Control-Allow-Origin: *` without
  credentials. (The backend research pass described this as "reflects any Origin". The security
  pass's reading, a wildcard, matches the package's documented default.) No allow-list was found
  anywhere in this call path. Whether a reverse proxy or `createSecurityHeaders()` changes CORS
  headers was not traced.
- **Current impact:** limited. The main API uses Bearer tokens that browsers do not attach
  across origins. The refresh cookie is `httpOnly` + `SameSite=Strict`, and a wildcard origin
  cannot be combined with credentialed requests. Any origin can call endpoints that need no
  authentication and read the response.
- **Potential future impact:** if anyone adds `credentials: true` or a cookie-only authenticated
  endpoint, cross-origin exposure appears without any change to this line. This is a latent
  footgun.
- **Severity / confidence:** Medium / Verified (configuration); the impact today is Inferred to
  be low.
- **Next step:** decide whether wildcard CORS is intended for a Bearer-only API and document it,
  or scope it to `[DOMAIN_CLIENT, DOMAIN_SERVER, ADMIN_PANEL_URL]`, the same trusted-origin list
  `requireSameOrigin` uses. Make it configurable per AGENTS.md. Check proxy configs
  (`client/nginx.conf`, Helm ingress) for CORS overrides.

### R4. `/api/auth/refresh` lacks `requireSameOrigin`

- **Evidence:** `requireSameOrigin` guards `/login` (`auth.js:73`), `/2fa/verify-temp` (`:156`),
  and `/passkey/login/verify` (`:188`). `router.post('/refresh', refreshController)` (`:81`) has
  no guard, and `auth.cross-site.test.js` has no cross-site case for `/refresh`.
- **Current impact:** defense-in-depth inconsistency, not a demonstrated bypass. Browsers do not
  send the `SameSite=Strict` refresh cookie on cross-site requests, so a cross-site POST fails at
  the missing-token check (`packages/api/src/auth/localRefresh.ts:112-114`).
- **Potential future impact:** exposure if the cookie policy is relaxed (`Lax`/`None` for
  embedding or cross-domain admin panels), or in non-standard webviews.
- **Severity / confidence:** Low / Verified.
- **Next step:** confirm that no legitimate cross-origin caller (the admin panel on another
  origin, or mobile/webview clients) relies on `/refresh`. If none does, add the guard and a
  matching test.

### R5. Per-user provider keys use legacy fixed-IV AES-CBC

- **Evidence:** `packages/data-schemas/src/methods/key.ts:3,121` imports and calls the v1
  `encrypt`, which uses AES-CBC with a fixed IV from `CREDS_IV` (`packages/data-schemas/src/crypto/index.ts:27-65`).
  2FA secrets (`api/server/controllers/TwoFactorController.js:43-50`) and admin secrets
  (`packages/api/src/admin/secrets.ts`) use `encryptV3` (AES-256-CTR, random IV).
- **Current impact:** anyone with read access to the `keys` collection or a backup can tell
  when two users stored the same key (deterministic ciphertext). Anyone with write access can
  tamper with ciphertext undetected. Both require database access.
- **Potential future impact:** a DB or backup leak reveals more than it should. Rotating
  `CREDS_KEY`/`CREDS_IV` is harder while three generations are in use.
- **Severity / confidence:** Medium / Verified.
- **Next step:** design a read-both/write-v3 migration, following the pattern of `getTOTPSecret`
  in `api/server/services/twoFactorService.js:161-174`. Decide on a lazy re-encrypt or a
  `config/migrate-*.js` batch script. Consider an AEAD generation as well (R22).

### R6. CSP implemented but disabled by default

- **Evidence:** a nonce-based SPA-shell CSP (`createCspPolicy`/`issueCsp`/`applyCspNonce`,
  `packages/api/src/security/csp.ts`, wired at `api/server/index.js:305-369,464`) returns `null`
  unless `CSP_ENABLED` is set (`csp.ts:201`). `.env.example:119` shows `# CSP_ENABLED=false`. A
  report-only mode exists.
- **Current impact:** default deployments serve the app shell without a CSP, so XSS
  mitigation relies only on React escaping and the other helmet headers.
- **Potential future impact:** a rendering path that bypasses escaping (markdown, artifacts,
  MCP App content) would have no second layer of defense.
- **Severity / confidence:** Medium / Verified.
- **Next step:** find out why it is off by default (U6). Measure report-only violations on a
  staging deployment. Consider a default-on report-only mode. Under AGENTS.md, a policy toggle
  like this belongs in `configSchema`, not only in env.

### R7. Business logic living in `api/` CJS

- **Evidence:**
  - `api/server/routes/convos.js:284-456`: deletion orchestration with retry loops, generation
    draining with `GenerationJobManager.abortJob`, an owner-deletion fence, and a persistence
    sweep, all inline in a route file.
  - `api/server/middleware/requireJwtAuth.js:91-215`: recursive multi-strategy authentication
    with fallback logging.
  - `api/server/controllers/agents/request.js` (3,679 lines) and `controllers/agents/client.js`
    (6,424 lines): the core chat controller and client.
  - `api/app/clients/BaseClient.js` (2,101 lines): the history tree-walk and legacy truncation.
  - Overall, `api/` has about 88k non-test lines against about 281k in `packages/api/src`.
  - Contrast: `api/server/services/Files/routing.js` ("Wiring only") and
    `packages/api/src/user/preferences.ts` (an injected-dependency factory).
- **Current impact:** this logic is untyped (CJS), harder to unit test without supertest and
  module mocks, and cannot be reused from other ingress adapters. `CONTEXT.md` describes Chat
  Completions, Responses, and Channels sharing an execution authority, and logic held in route
  files works against that.
- **Potential future impact:** behavior that should be shared gets duplicated across ingresses.
  The ordering invariants in `request.js` (save before `final`, error persisted before
  publication) can regress during refactors because nothing structural enforces them.
- **Severity / confidence:** Medium / Verified.
- **Next step:** move logic only when touching it (AGENTS.md). Good first candidates are the
  `convos.js` deletion helpers, which are already self-contained functions over injected
  primitives, moved into a `packages/api/src/conversations/` factory with tests that use
  `mongodb-memory-server`.

### R8. Enforced agent id derived independently in two places

- **Evidence:** when `modelSpecs.enforce` is on, the enforced `agent_id` is resolved in
  `api/server/middleware/accessResources/canAccessAgentFromBody.js:18-31,177` (for the ACL check)
  and again in `api/server/middleware/buildEndpointOption.js:88-135` (for building the run). Both
  call `resolveModelSpecForEndpoint`, but neither passes its result to the other.
- **Current impact:** none known. Both read the same config and the same `req.body.spec`.
  `CONTEXT.md` ("Effective agent selection") says authorization and agent loading "must consume
  this same identity", and right now that holds only because the two code paths agree.
- **Potential future impact:** a change to one path (spec matching, endpoint normalization,
  ephemeral-agent handling) but not the other could authorize agent A and run agent B.
- **Severity / confidence:** Medium / Verified (duplication), Inferred (exploitability).
- **Next step:** compute the effective selection once, early in the chat router chain, store it
  on the request, and have both middlewares read it. Add a test that changes spec resolution and
  asserts that the ACL check and the run use the same id.

### R9. Provider file-size defaults hard-coded outside `configSchema`

- **Evidence:** provider ceilings are inline literals in `packages/api/src/files/validation.ts`
  (`mbToBytes(32)` at `:74`, `4.5`/`32` at `:205`, `10` at `:253`, `20` at `:277,306,343,380`,
  `5` at `:393`). General file limits *are* declared in shared config
  (`packages/data-provider/src/file-config.ts:553-590`, for example a 512 MB `defaultSizeLimit`).
  A configured endpoint limit replaces the provider ceiling, including upward
  (`configuredFileSizeLimit ?? providerLimit`). Tests show upward overrides are intentional
  (`packages/api/src/files/encode/document.spec.ts:420-449`, "allows API changes").
- **Current impact:** an operator reading `librechat.yaml`/`configSchema` cannot see the
  effective per-provider defaults.
- **Potential future impact (open question, Inferred):** `getConfiguredFileSizeLimit`
  (`packages/api/src/files/encode/utils.ts:59-76`) merges the deployment `fileConfig` with
  defaults. If a deployment declares any `fileConfig`, the merged endpoint limit may always be
  populated (with the 512 MB default), which would silently replace the stricter provider
  ceilings. **This is not verified.** It depends on whether `fileConfig` is always present on
  `req.config` and on `getEndpointFileConfig`'s return for unconfigured endpoints.
- **Severity / confidence:** Low / Verified (placement); the override question is Unknown.
- **Next step:** write a focused test that sets a `fileConfig` with no endpoint
  `fileSizeLimit`, uploads a 15 MB PDF to OpenAI, and checks which limit applies. Consider moving
  the provider defaults into `file-config.ts` next to the other defaults.

### R10. Multi-replica correctness depends entirely on Redis being configured

- **Evidence:** a single flag, `GenerationJobManager.isRedis`, switches the job store and event
  transport between process-local memory and Redis (`createStreamServices.ts:72-137`). Without
  Redis:
  - The chat POST and its SSE GET can land on different replicas, and a reconnect on another
    replica cannot see the job. Replay uses an in-process `emissionSequence` instead of the
    persisted `durableEventSequence`.
  - `limiterCache` returns `undefined`, so rate limits fall back to per-process memory.
    Effective limits multiply with the replica count (**Inferred** from
    `cacheFactory.ts:197-238`).
  - The concurrency limiter, violation scores, and `express-session` handshake state become
    per process.
  - `LeaderElection.isLeader()` returns `true` on **every** replica, so leader-gated jobs run
    everywhere (`packages/api/src/cluster/LeaderElection.ts:62`; consumers include
    `files/sweep.ts`, `code/lifecycle.ts`, `mcp/registry/MCPServersInitializer.ts`).
  - The schedule engine refuses to arm unless overridden (D8).
  - **A Redis construction failure at startup silently degrades to in-memory** with only an
    error log (`createStreamServices.ts:105-111`).
- **Current impact:** Helm defaults to `replicaCount: 1` with autoscaling off
  (`helm/librechat/values.yaml:5,215-218`), so default installs are safe. Any operator who
  scales out without `USE_REDIS=true` gets resume, Stop, limit, and leader behavior that is wrong
  only some of the time, depending on load-balancer routing.
- **Potential future impact:** HPA (`maxReplicas: 100` is pre-set) turns a configuration miss
  into split-brain behavior that is hard to diagnose.
- **Severity / confidence:** **High** for multi-replica deployments, none for single-replica /
  Verified.
- **Next step:** document "replicas > 1 ⇒ `USE_REDIS=true`" as a hard requirement in deployment
  docs and Helm values. Consider failing startup, or failing `/readyz`, when
  `isRedis !== USE_REDIS_STREAMS` after a fallback. Consider a Helm template guard that ties
  `replicaCount > 1` or autoscaling to Redis settings.

### R11. Subagent executions are process-bound

- **Evidence:** the "live subagent task owner" (`CONTEXT.md`) is a per-process
  `TaskThreadLease` holding the executor, its abort controller, and a control queue
  (`packages/api/src/agents/subagentThreads.ts:206-234`). Redis is used only to route
  control/activity envelopes to that owner (`RedisSubagentTaskControlTransport`,
  `subagentThreadStore.js:110-160`, only when `USE_REDIS`). Without Redis, steer and cancel
  requests reach a child only on the replica that owns it. With Redis, the executor still cannot
  migrate. Shutdown cancels locally owned children (`destroyTaskControlTransport`, up to 45 s).
- **Current impact:** a crash or deploy terminates running children. The durable completion
  wakeup is registered *before* execution (`subagentCompletionWakeup.ts`), so the parent is still
  notified, but the child's work is lost. Without Redis, a steer or cancel sent from a replica
  other than the owner cannot reach the child.
- **Potential future impact:** long-running subagent workloads limit how often you can deploy and
  prevent rebalancing across replicas.
- **Severity / confidence:** Medium / Verified (mechanism); deploy-frequency impact is Inferred.
- **Next step:** measure typical child durations. Document the deploy-time behavior. Evaluate
  whether children could checkpoint (the LangGraph checkpoint machinery behind event actors may
  be reusable).

### R12. Redis Cluster status guard is best-effort

- **Evidence:** `packages/api/src/stream/interfaces/IJobStore.ts:1169-1172`: on Redis Cluster,
  `transitionStatus` is not fully atomic because the membership sets (`stream:running`, ...) live
  in a different hash slot from the per-stream job hash.
- **Current impact:** the code documents this. On single-node or Sentinel Redis, the Lua scripts
  are atomic. On Cluster, global index sets can briefly disagree with job status.
- **Potential future impact:** cleanup sweeps or recovery lanes that rely on set membership could
  act on stale membership under Cluster.
- **Severity / confidence:** Low–Medium / Verified (self-documented).
- **Next step:** list which consumers of the global sets treat membership as authoritative and
  confirm they re-check the job hash. Add a Cluster scenario to `test:cache-integration:*`.

### R13. MCP static singletons

- **Evidence:** `MCPManager.getInstance()` (`packages/api/src/mcp/MCPManager.ts:140,204`), with
  `MCPServersRegistry.getInstance()` called from at least six sites in it. AGENTS.md calls this
  "the shape to stop extending".
- **Current impact:** tests must reset global state. Per-tenant or per-request MCP variation goes
  through shared mutable state.
- **Potential future impact:** each new MCP feature added to the singleton makes the later
  dependency-injection refactor larger.
- **Severity / confidence:** Low / Verified.
- **Next step:** for new MCP code, accept a manager/registry interface as a parameter and resolve
  the singleton only at the `api/` wiring edge.

### R14. Serial startup chain

- **Evidence:** `api/server/index.js` startup has no `Promise.all`. `seedDatabase` runs four
  sequential steps (`api/models/index.js:21-26`), and plugins then skills are awaited serially
  (`index.js:244-260`). The request path does parallelize independent reads (for example
  `packages/data-schemas/src/methods/conversation.ts:3603-3608`).
- **Current impact:** unmeasured. It matters under the Lighthouse lane's 250 ms-per-query
  latency injection and for cold starts during autoscaling.
- **Severity / confidence:** Low / Verified (shape), Unknown (whether the steps depend on each
  other or how much it costs).
- **Next step:** time each startup phase. Check whether `seedSystemGrants` depends on roles,
  and whether skills depend on plugins (U11), before parallelizing anything.

### R15. No limiter ahead of JWT verification on protected routers

- **Evidence:** for example `api/server/routes/convos.js:179` begins with
  `router.use(requireJwtAuth)`, and every request then does a `getUserById`. By contrast,
  `/login` runs `loginLimiter` and `checkBan` before bcrypt (`auth.js:70-80`).
- **Current impact:** an unauthenticated flood of requests carrying syntactically valid tokens
  costs a DB read each. Proxy-level limits may already cover this, but that was not traced.
- **Severity / confidence:** Low / Verified (ordering), Unknown (mitigation elsewhere).
- **Next step:** check ingress/nginx rate limiting. Measure the cost of a forged-token flood with
  the auth user-doc cache warm and cold.

### R16. Full conversation fetched on every turn

- **Evidence:** `loadHistory` fetches every message row (`db.getMessages({conversationId, user})`)
  and walks the branch in memory (`api/app/clients/BaseClient.js:1499-1549`). `CRITICAL_FLOWS.md`
  Flow 4 calls this out as a deliberate trade-off.
- **Current impact:** fine for typical sizes. Heavily edited or regenerated conversations pay for
  every abandoned branch on every turn.
- **Severity / confidence:** Low–Medium / Verified.
- **Next step:** add per-turn metrics for rows fetched vs. path length. Consider a
  path-only query or caching once the ratio is known.

### R17. No migration framework; breaking index changes need manual runs

- **Evidence:** packaged migrations (`packages/data-schemas/src/migrations/*.ts`) run only through
  operator CLIs (`config/migrate-*.js`). Some data migrations are inlined into the startup seed
  (`initializeRoles`, `packages/data-schemas/src/methods/role.ts:76-104`). `UPGRADING.md`
  documents `Index build failed` after upgrading from v0.8.7 or earlier, fixed only by running
  `npm run migrate:tenant-indexes` with all replicas stopped.
- **Current impact:** upgrades can produce silently unenforced unique constraints until an
  operator acts. Index-build failures are logged, not fatal
  (`packages/data-schemas/src/models/index.ts:156-174`). `checkMigrations()` runs after the server
  starts listening, but what it checks was not traced.
- **Severity / confidence:** Medium / Verified.
- **Next step:** read `checkMigrations()` to see whether it can detect pending index migrations
  and fail readiness. Consider recording applied migrations in a collection.

### R18. Very large load-bearing files

- **Evidence:** `packages/api/src/stream/GenerationJobManager.ts` (9,785 lines),
  `api/server/controllers/agents/client.js` (6,424), `stream/implementations/RedisJobStore.ts`
  (5,987), `api/server/controllers/agents/request.js` (3,679). `CRITICAL_FLOWS.md` already calls
  `request.js` "the most load-bearing file in the backend and the least decomposed".
- **Current impact:** changes have a wide blast radius, reviews take longer, and timing
  invariants are hard to test in isolation.
- **Severity / confidence:** Medium / Verified.
- **Next step:** list the invariants each file enforces (save ordering, fencing, drain) as named
  tests before any decomposition.

### R19. Tenant header misconfiguration fails open to no tenant scope

- **Evidence:** `packages/api/src/middleware/preAuthTenant.ts:42` ignores `X-Tenant-Id` unless
  `TRUST_TENANT_HEADER=true`. A rejected or absent header leaves the request with no tenant
  context instead of returning an error.
- **Current impact:** this is correct for single-tenant deployments. A multi-tenant deployment
  that forgets the env var runs pre-auth routes (`/api/config`, `/api/auth/*`, `/oauth/*`, public
  share) **without tenant scoping**, and nothing fails loudly.
- **Severity / confidence:** Medium for multi-tenant deployments / Verified.
- **Next step:** consider a startup warning, or a fail-closed mode, when tenant data exists but
  `TRUST_TENANT_HEADER` is unset. Under AGENTS.md this toggle is a candidate for `configSchema`.

### R20. Stream delta coalescing can drop unflushed deltas on crash

- **Evidence:** `UPGRADING.md` (as summarized in the testing research): Redis-backed streams
  coalesce deltas in a 25 ms window by default (`STREAM_DELTA_COALESCE_MS`). A process crash can
  lose unflushed deltas. Terminal barriers still flush. In-memory streams are unaffected.
- **Current impact:** small. The final message is persisted from the complete content, so this
  affects live display only (**Inferred**).
- **Severity / confidence:** Low / Verified (documented).
- **Next step:** none required. Make sure custom stream subscribers support `chunk_batch` frames.

### R21. Dev compose ships insecure defaults

- **Evidence:** `docker-compose.yml` runs `mongod --noauth` and bakes static pgvector credentials
  (`myuser`/`mypassword`) into the file.
- **Current impact:** none in local development. Copying this file to production would expose an
  unauthenticated database.
- **Severity / confidence:** Low (dev), High if reused in production / Verified.
- **Next step:** add a prominent comment, or point production users to `deploy-compose.yml` /
  Helm with secrets.

### R22. No encryption generation authenticates ciphertext

- **Evidence:** v1 and v2 are AES-CBC and v3 is AES-256-CTR, all without a MAC or AEAD
  (`packages/data-schemas/src/crypto/index.ts:27-148`). `PROJECT_MAP.md` warns "don't add a 4th
  casually".
- **Current impact:** an attacker with DB write access can tamper with stored secrets undetected.
  This requires database compromise.
- **Severity / confidence:** Low / Verified.
- **Next step:** if R5's migration goes ahead, consider making the target an AEAD such as
  AES-256-GCM rather than v3, so the data only has to move once.

### R23. Helm readiness probe uses `/health`, not `/readyz`

- **Evidence:** the Helm chart's `readinessProbe` is configured against `/health`
  (`helm/librechat/values.yaml`), which returns 200 as soon as the process starts listening — not
  once startup (config load, DB connection, readiness gates) has actually finished. Only `/readyz`
  waits for startup to complete, and today it is polled only by CI's `docker-smoke.yml`, not by the
  Helm probe itself. See [`12-configuration-deployment.md`](./12-configuration-deployment.md) §8.
- **Current impact:** in a Kubernetes rollout, traffic can be routed to a pod before it has
  finished connecting to Mongo/loading config, causing early requests to fail or hit the fail-open
  config fallback (**Inferred**).
- **Severity / confidence:** Medium / Verified (probe target confirmed; blast radius inferred).
- **Next step:** point the Helm `readinessProbe` at `/readyz` instead of `/health`.

### R24. No `terminationGracePeriodSeconds` set, shorter than the app's shutdown budget

- **Evidence:** `helm/librechat` does not set `terminationGracePeriodSeconds` anywhere, so
  Kubernetes uses its default of 30 seconds. The application's own phased shutdown
  (`packages/api/src/app/shutdown.ts`) budgets up to 60 seconds to drain in-flight generations,
  schedules, and subagents before force-exiting. See
  [`01-architecture.md`](./01-architecture.md) §11 and
  [`09-background-processing.md`](./09-background-processing.md) §5.
- **Current impact:** Kubernetes can `SIGKILL` a pod mid-drain, before the app's own 60-second
  safety timer would have force-exited it, potentially losing an in-flight generation's chance to
  record a clean terminal state (**Inferred** — depends on real-world generation durations at
  rollout time).
- **Severity / confidence:** Medium / Verified (config gap confirmed; real-world frequency inferred).
- **Next step:** set `terminationGracePeriodSeconds` to at least 65 (slightly above the app's own
  60-second budget) in the Helm chart's pod spec.

---

## 5. Corrections to existing documentation

Found by the research pass. These are **not** edits to those files; they are recorded here so
readers know.

| Document | Claim | Current reality | Evidence |
|---|---|---|---|
| `PROJECT_MAP.md` §6, `CRITICAL_FLOWS.md` Flow 2 (item 3) | `checkAdmin` gates `/api/admin/*` | Admin routes use `requireCapability(SystemCapabilities.ACCESS_ADMIN)`; `checkAdmin` is unused | R1 |
| `CRITICAL_FLOWS.md` Flow 2 gotcha | Capabilities are excluded from the barrel "to force call sites to notice ordering" | The stated primary reason is avoiding a **circular require** | `api/server/middleware/roles/index.js:1-11` |
| `PROJECT_MAP.md` §9 | `client/src/store/agents.ts` and `ptc.ts` are "Mixed" Recoil + Jotai | Both currently import only `recoil` | Frontend research pass |
| `PROJECT_MAP.md` §6 / `CRITICAL_FLOWS.md` | ACL bits are 1/2/4/8 | There is also `VIEW_INSIGHTS = 16` | `packages/data-provider/src/accessPermissions.ts:68` |
| `PROJECT_MAP.md` §5 | Refresh tokens tracked via `refreshTokenHash` | The refresh token is itself a JWT signed with `JWT_REFRESH_SECRET`, **and** it is tracked server-side in the `Session` model | `AuthService.js:705-746`; `packages/api/src/auth/localRefresh.ts:35-41` |
| `PROJECT_MAP.md` §10 | `.env.example` about 1,480 lines | About 1,500 lines | Minor drift |
| `PROJECT_MAP.md` §5 | `api/models/index.js` is 29 lines | About 33 lines, still pure wiring | Minor drift |
| `PROJECT_MAP.md` §3 | Default port 3080 | Also: `PORT=0` is supported for automatic assignment | `api/server/index.js:99-103` |

---

## 6. Open questions for follow-up

These could not be settled from the code read so far. Each one has a concrete way to find the
answer.

1. Does the seeded `ADMIN` role hold `SystemCapabilities.ACCESS_ADMIN` by default, and can a
   deployment drift the two apart? Trace `seedDefaultRoles` / `seedSystemGrants`.
2. Does any reverse proxy or `createSecurityHeaders()` rewrite CORS headers? Read
   `packages/api/src/security/headers.ts`, `client/nginx.conf`, and the Helm ingress.
3. Can a declared `fileConfig` silently replace the stricter provider file limits (R9)? Write the
   focused test described there.
4. What does `checkMigrations()` check, and could it fail readiness on pending index migrations
   (R17)?
5. Is `api/server/experimental.js` intended to become a production entry point (U7)?
6. Are the `CONTEXT.md` lifecycle claims for queued turns, the trigger capability shield, and
   event actors fully implemented as written? The research pass confirmed their storage shape,
   but not every lifecycle branch.
7. Do the serial startup steps have ordering dependencies (U11/R14)?
