# Eleven-Chat Engineering Governance

This is the canonical engineering rulebook for developers and AI coding agents working in this custom LibreChat fork,
including Claude Code, Cursor, Aider, GitHub Copilot, and similar tools.

**Rule language:** MUST / MUST NOT are mandatory; SHOULD / SHOULD NOT are defaults requiring a specific reason to depart
from; MAY is optional. **Source of truth:** current implementation first, then tests, configuration, Git history, and
documentation as supporting evidence. Docs can drift; verify the relevant code. Use
[`docs/developer-guide/README.md`](docs/developer-guide/README.md) to find the focused architecture, backend, frontend,
Redis, database, API, security, background, testing, and deployment guides. `PROJECT_MAP.md`, `CRITICAL_FLOWS.md`, and
`CONTEXT.md` are companion references, not substitutes for source inspection.

## 1. Mission and Engineering Principles

Prioritize: (1) preserve upstream compatibility; (2) minimize and isolate the custom diff; (3) respect actual
architectural boundaries; (4) protect security and data integrity; (5) prefer simple, low-maintenance solutions; (6) test
observable behavior; (7) keep docs/contracts synchronized; and (8) avoid unnecessary dependencies and abstractions.

The smallest patch is not automatically the best patch. A **minimal implementation** changes only what is needed while
preserving correctness, security, lifecycle, and maintainable boundaries. An **unsafe shortcut** saves lines by bypassing
checks, duplicating a large upstream module, hiding failures, or omitting compatibility and regression coverage.

## 2. Golden Rules — Non-Negotiable

1. Read this file, applicable nested instructions, and relevant customization records before planning or editing; recheck
   for nested `AGENTS.md` files when entering a subtree.
2. Inspect the current implementation, callers, tests, and nearest equivalent before changing behavior or introducing a
   pattern.
3. Preserve upstream behavior by default. Never rewrite or copy a large upstream module when a narrow extension, adapter,
   or isolated module will do.
4. Never change unrelated behavior, contracts, UI, configuration, or persisted data as incidental cleanup.
5. Never perform destructive database, Redis, infrastructure, or Git operations without explicit authorization and a
   reviewed safety plan.
6. Never expose secrets or bypass authentication, authorization, tenant isolation, validation, or established security
   controls.
7. Never claim a test, build, typecheck, migration, or verification succeeded unless its result was actually observed. A
   build is not a typecheck.
8. Never introduce an undocumented architecture, API/configuration contract, or dependency; obtain approval for
   dependencies and breaking contracts.
9. Never commit, push, merge, rebase, reset, deploy, or rewrite Git history without explicit authorization.
10. Never call a task complete while regressions, failed required checks, skipped verification, or unresolved risks remain
    unexplained.

**Precedence:** Golden Rules override every other rule. Explicit task instructions may choose among safe options or waive
a recommendation, but do not silently waive a Golden Rule. Stop and explain conflicting mandatory rules before taking a
risky action.

## 3. Repository Architecture and Change Planning

### Verified architecture

This is a substantially customized LibreChat fork. The verified runtime is Node.js 24 with npm workspaces, Turborepo, a
CommonJS Express 5 server, TypeScript packages, and a React/Vite SPA.

| Boundary | Verified owner | Rule for new work |
|---|---|---|
| Server wiring | `api/`, especially `api/server/index.js` and `api/server/routes/` | Legacy CommonJS Express wiring and existing logic live here. Put new backend behavior in `packages/api`; keep `/api` changes to registration and dependency wiring where practical. |
| Backend behavior | `packages/api` (`@librechat/api`) | Use feature modules, typed handlers/services, and injected dependencies. |
| Persistence | `packages/data-schemas` (`@librechat/data-schemas`) | Own Mongoose schemas, models, query/mutation methods, tenant isolation, and migration helpers here. |
| Shared API | `packages/data-provider` (`librechat-data-provider`) | Own shared types, `configSchema`, endpoint builders, `dataService`, request helpers, and React Query keys. It is browser-bundled: no Node-only code or secrets. |
| Shared UI | `packages/client` (`@librechat/client`) | Own app-agnostic primitives and theme tokens; keep domain UI in the app. |
| Web app | `client/` | React 18, Vite, React Router, React Query, legacy Recoil plus Jotai for new state. |
| Supporting services | MongoDB/Mongoose is primary; Redis is optional; Meilisearch is derived search state; RAG uses the external `rag_api` PostgreSQL/pgvector sidecar; provider execution uses `@librechat/agents`. | Verify the affected integration and deployment mode before changing it. |

The browser talks to Express over REST and SSE. Agent chat starts a durable job with one request and subscribes/resumes
its stream with another. A successful start response does not mean the model run completed. Treat `GenerationJobManager`,
persistence ordering, event replay, cancellation, and reconnect behavior as load-bearing; inspect current source and
`CRITICAL_FLOWS.md` before changing them.

### Inspection and approval

Before editing, map the feature's entry points, call sites, data owner, extension points, and trust boundaries across
frontend, API, backend, database, Redis, external services, and workers as applicable. Inspect a similar implementation
and its tests. Separate shared infrastructure from feature logic; avoid circular dependencies, duplicate business rules,
and broad refactors. Explain a deviation from local conventions.

For a **non-trivial feature**, present the proposed file structure and concise plan before writing application code, then
wait for developer approval. Include: (1) existing and requested behavior; (2) files/modules and boundaries; (3)
API/config/database/Redis/background contract changes; (4) security, failure, compatibility, and rollback risks; (5)
tests/docs; and (6) upstream merge risk and isolation strategy. New routes, cross-layer features, dependencies, schema
changes, background processing, security changes, and runtime configuration are non-trivial. For an explicitly authorized
localized fix or documentation-only task, proceed without an unnecessary planning round, but still inspect, test, and
report proportionally.

## 4. Commit Convention and Change Isolation

When committing is explicitly authorized, use `[tag] imperative description` and one coherent concern per commit. Use
separate commits for independent concerns when requested. Avoiding “and” in a subject is a useful heuristic, not an
absolute rule; coherence is the requirement.

| Tag | Appropriate use |
|---|---|
| `[brand]` | Branding or white-label behavior. |
| `[feat]` | New user-facing capability. |
| `[logic]` | Domain rule or behavior change. |
| `[fix]` | Unintended or broken behavior. |
| `[config]` | Config schema, defaults, or operator setting. |
| `[style]` | Visual styling/layout without behavior or copy changes. |
| `[i18n]` | Localization resources or plumbing. |
| `[infra]` | Docker, CI, deployment, build, or operations. |
| `[merge]` | Authorized upstream synchronization/conflict resolution. |
| `[docs]` | Documentation/governance. |
| `[test]` | Tests or test infrastructure only. |
| `[security]` | Security control, fix, or hardening. |
| `[db]` | Schema, data access, migration, or data integrity. |

Every commit body MUST explain why, implementation approach, affected files/subsystems, and relevant risk. Name database
migration and API contract changes explicitly. Do not rewrite published history without authorization; never commit or
push automatically.

**Good subjects** (a body with the required rationale is still mandatory):

```text
[feat] add tenant-scoped agent run history
[fix] preserve terminal events after Redis reconnect
[db] add an idempotent tenant index migration
[security] reject private network targets in outbound fetches
[docs] document resumable stream compatibility rules
```

**Bad examples:** `feat: Added new features and fixes` (wrong format, vague, unrelated); `[fix] update` (no behavior
identified); `[feat] refactor everything and redesign UI` (bundled concerns and unbounded scope).

## 5. Custom Code Identification and Markers

In upstream files, mark the smallest coherent custom block with:

```text
>>> CUSTOM:START [feature-tag] <<<
>>> CUSTOM:END [feature-tag] <<<
```

Tags MUST be unique per feature, lowercase, and hyphenated. Use valid comments for the file type:

| File type | Example |
|---|---|
| JavaScript | `// >>> CUSTOM:START [agent-run-badges] <<<` |
| TypeScript / TSX | `// >>> CUSTOM:START [agent-run-badges] <<<` (place as a normal code comment, not in executable JSX). |
| CSS | `/* >>> CUSTOM:START [agent-run-badges] <<< */` |
| YAML | `# >>> CUSTOM:START [agent-run-badges] <<<` |
| Dockerfile | `# >>> CUSTOM:START [agent-run-badges] <<<` |
| JSON | No comments/markers; valid snippet `{"custom-command":"node scripts/custom.js"}`; inventory the JSON file, feature tag, and rationale in `CUSTOMIZATIONS.md`. |

Never nest markers, put them in executable expressions, change behavior with comments, or leave mismatched/orphan markers.
Do not mark wholly custom files without a concrete benefit. Do not copy an upstream module merely to make the custom code
visible. When replacing upstream behavior, do not blindly comment out the old implementation: preserve it only when safe,
meaningful, and syntactically valid; otherwise make the smallest justified edit and document the original behavior and
reason.

Locate marker lines in code/configuration (Markdown examples and JSON excluded):

```sh
rg -n --hidden --glob '!**/.git/**' --glob '!**/node_modules/**' --glob '!**/dist/**' \
  --glob '!graphify-out/**' --glob '!**/*.md' --glob '!**/*.json' \
  '>>> CUSTOM:(START|END) \[[a-z0-9]+(-[a-z0-9]+)*\] <<<' .
```

Check comment-line syntax, tag matching, nesting, and pairing in tracked and untracked non-ignored text files:

```sh
python3 - <<'PY'
import re, subprocess, sys
pat = re.compile(r'^\s*(?://|#|/\*)\s*>>> CUSTOM:(START|END) \[([a-z0-9]+(?:-[a-z0-9]+)*)\] <<<\s*(?:\*/)?\s*$')
files = subprocess.check_output(['git','ls-files','-z','--cached','--others','--exclude-standard']).split(b'\0')
errors = []
for raw in filter(None, files):
    path = raw.decode('utf-8', 'surrogateescape')
    if path.endswith(('.md','.json','.lock')) or path.startswith(('graphify-out/','node_modules/','dist/','build/')): continue
    try: lines = open(path, encoding='utf-8').read().splitlines()
    except (UnicodeDecodeError, OSError): continue
    stack = []
    for n, line in enumerate(lines, 1):
        if '>>> CUSTOM:' not in line: continue
        m = pat.fullmatch(line)
        if not m: errors.append(f'{path}:{n}: malformed marker'); continue
        kind, tag = m.groups()
        if kind == 'START':
            if stack: errors.append(f'{path}:{n}: nested marker')
            stack.append((tag,n))
        elif not stack: errors.append(f'{path}:{n}: orphan END [{tag}]')
        elif stack[-1][0] != tag: errors.append(f'{path}:{n}: mismatched END [{tag}]'); stack.pop()
        else: stack.pop()
    errors.extend(f'{path}:{n}: unclosed START [{tag}]' for tag,n in stack)
if errors: print('\n'.join(errors)); sys.exit(1)
print('Custom marker pairs are balanced in scanned files.')
PY
```

This lexical check does not parse each language, validate comment placement, inspect ignored/generated files, or catch
unusual string/comment formats. It is a pairing aid, not a substitute for linting, parsing, or reviewing the diff.

## 6. File Organization and Upstream Boundaries

No canonical `custom/` directory is present. Do not invent `api/server/custom/` or `client/src/custom/` without verifying
module/build support. Use these existing homes:

| Concern | Existing home |
|---|---|
| Backend middleware/policy | `packages/api/src/middleware/` or feature directory; legacy wiring remains in `api/server/middleware/`. |
| Backend services/handlers | `packages/api/src/<feature>/`, using existing exports and injected dependencies. |
| HTTP routes | Existing `api/server/routes/` registration and `api/server/index.js` mount convention; logic stays in `packages/api`. |
| Data | `packages/data-schemas/src/{schema,models,methods,migrations}/`. |
| Frontend pages/components/hooks | `client/src/{routes,components/<Feature>,hooks/<Feature>}/`. |
| Shared UI/styles | Domain-independent primitives/theme tokens in `packages/client/`; app styles in existing `client/src/` files. |
| Shared API/types | `packages/data-provider/src/`; keep private backend and app-only logic in their owning packages. |

Prefer extension points, wrappers, factories, and narrow adapters over upstream edits. Do not change upstream signatures,
rename upstream paths, add hidden imports, monkey-patch runtime modules, or introduce undocumented global side effects.
Never modify upstream files solely for organization. Document every necessary upstream modification and its rationale.

## 7. Backend Engineering Rules

- `/api` is legacy CommonJS Express wiring with existing logic; new backend behavior MUST go in TypeScript under
  `packages/api` wherever practical. Treat existing inline route logic as legacy, not as a pattern to copy. Migrate
  touched logic only when scoped and testable.
- Keep transport parsing/authorization integration/orchestration/response handling separate from domain rules where the
  existing handler/service boundary supports it. Put database queries in `packages/data-schemas`, not arbitrary modules.
- Follow existing validation, logging, error, and DI patterns. Pass config, clients, and data methods from the caller; do
  not add globals or new static singletons. Existing MCP singletons are not a pattern to extend.
- Validate untrusted input and enforce server-side authentication, authorization, tenant scope, and ownership before reads
  or side effects. Never write a response before required checks succeed.
- Handle rejected promises explicitly. Bound concurrency, in-memory collections, retries, timeouts, and polling; support
  cancellation, cleanup, and graceful shutdown where applicable.
- Reuse already-loaded request/config/user data, avoid serial reads, and parallelize independent reads only after
  authorization prerequisites are satisfied.
- When user documents change, invalidate the auth user-document cache for every affected user, including bulk role/account
  mutations; follow `packages/data-schemas/src/methods/user.ts`.
- Do not widen package APIs with Mongoose types (`Document`, `FilterQuery`, `Types.ObjectId`); pass plain typed values
  across boundaries. Do not add layers without a concrete maintainability or correctness benefit.

Follow local style: prefer short, single-word filenames/directories where natural; use explicit types and flat early returns;
avoid `any`, broad casts, duplicate types, unnecessary dynamic imports, extra array passes/allocations, and comments that
repeat the code. Keep imports in the repository's
groups (package values shortest-first with `react` first; package types, then local types; local values last; type/local
groups longest-first). Use standalone `import type`, not inline type specifiers, and run the scoped import sorter on
touched files.

For each new backend operation, consider: input validation; authz/tenant/ownership; business invariants;
database/Redis/external side effects; transactions/concurrency/idempotency; error mapping; safe logs/metrics; tests;
API/config docs; and upstream merge impact. New limits/timeouts/toggles SHOULD use `packages/data-provider/src/config.ts`
`configSchema` with a default preserving current behavior. Document justified environment-only controls and update
sanitized examples, config-loading tests, and deployment docs.

## 8. Frontend Engineering Rules

- Follow current React, routing, React Query, API client, component, and styling conventions. Use `packages/data-provider`
  URL builders, `dataService`, shared types, and auth flow; define React Query keys in
  `packages/data-provider/src/keys.ts` and invalidate/update affected queries.
- React Query owns server state; keep local UI state local. Do not duplicate server data in another store without a
  synchronization plan.
- **New client state MUST use Jotai**, even in files importing Recoil. Convert an existing atom with all readers/writers
  together; do not half-convert it. Keep feature-owned state in the feature. Pass app-global preferences/shell state
  through props or host context when a feature only consumes it; do not reach into `client/src/store` for unrelated global
  state. Use `client/src/store/jotai-utils.ts` for persisted atoms. Do not bundle a repository-wide migration.
- Reuse `@librechat/client` primitives and semantic theme tokens. Avoid raw palette colors, copied styles, arbitrary theme
  CSS, and broad redesigns. Theme definitions express semantic appearance/colors, not selectors, app behavior, or alternate
  layouts. Preserve light/dark, reduced motion, responsiveness, and defaults; test a deliberately different theme for new
  reusable variants, and explain why new CSS cannot use an existing primitive/token.
- For action/sort menus use `DropdownPopup` + Ariakit `MenuButton`; do not introduce/reintroduce the Radix `DropdownMenu`
  family for those menus. Preserve dialog trigger/ref and `hideOnClick: false` behavior when an item opens a dialog.
- Use `useLocalize()` for visible copy and add English source keys in `client/src/locales/en/translation.json`. Preserve
  semantic HTML, keyboard/focus behavior, ARIA, localization, and accessible error feedback.
- Handle loading, empty, success, failure, and unauthorized states. Prevent stale async results from overwriting newer
  state; consider cancellation, duplicate submissions, retries, and cleanup of listeners/timers/subscriptions/streams.
  Prefer framework-native rendering to direct DOM changes.

When changing shared components, inspect significant consumers and test them. For new user flows, trace interaction → API
request → state transition → rendered result. Ensure each user-facing backend capability has a reachable frontend entry
point and delivers loading, empty, success, failure, cancellation, and retry states as applicable.

## 9. API Contract and Integration Rules

- Inspect routes, middleware, callers, URL builders, types, and tests before adding/changing an endpoint. Use existing
  validation, response, error, and auth conventions; do not create a parallel validation system. Use appropriate HTTP
  methods/statuses and define missing-resource, invalid-input, conflict, and unauthorized behavior consistently.
- Validate path/query/body/headers/uploaded content as applicable. Enforce auth, permissions, tenant/resource scope,
  ownership, and rate/resource limits server-side; never trust client-supplied identity, roles, permissions, or ownership
  claims.
- Preserve method, route, response shape/meaning, pagination, filters, sorting, and error behavior unless a breaking
  change is explicitly approved with a migration strategy.
- Consider retries, idempotency, timeouts, duplicate delivery, and partial side effects. Never return stacks, query
  errors, provider payloads, secrets, or unsafe metadata.
- For every changed endpoint verify method/route, request and response schemas, authz, status/errors, side effects, retry
  behavior, and compatibility with known callers.
- Update URL builders, shared types, validators, React Query keys, tests, and docs together when a contract changes. The
  checked-in OpenAPI artifact is `packages/api/openapi/agents.openapi.json`, not a complete API reference. If affected,
  update its generator source and run the API workspace's `openapi:check` and `openapi:test` scripts.
- Preserve agent chat's separate job-start and SSE subscribe/resume contracts. Test replay, cancellation, error
  persistence, and terminal ordering when changing stream behavior.

## 10. Database Engineering and Data Integrity

**Verified:** MongoDB through Mongoose is the primary source of truth. `packages/data-schemas` owns schemas, model
factories, query/mutation methods, tenant isolation, and packaged migration helpers. Meilisearch is derived search state;
Redis is not the system of record; `rag_api` owns PostgreSQL/pgvector state.

- Inspect the model, all readers/writers, tenant behavior, and local migration/test pattern first. Follow
  `createModels(mongoose)` and `createMethods(mongoose, deps)`; inject dependencies and keep HTTP/localization concerns
  out of the data layer.
- Preserve tenant-scoped queries, indexes, and context. Use system access only for justified system/migration paths. Never
  broaden a query by dropping tenant/owner predicates.
- Choose nullability, defaults, validation, uniqueness, TTL, and indexes intentionally. Index for demonstrated
  query/integrity needs; consider build cost. Avoid N+1, unbounded, and full-collection reads on request paths.
- Use safe Mongoose/query-builder APIs, not unsafe query-string construction. Handle uniqueness races and concurrent
  writes at the database boundary.
- Use transactions only where deployment supports them. Standalone MongoDB and some DocumentDB deployments may not; follow
  `getTransactionSupport` and existing fallback behavior. Keep transactions short and outside external network calls.
- `null` means documented absence, never a swallowed query failure. Preserve data and avoid destructive schema changes as
  incidental cleanup.
- This repo has packaged migration helpers and operator-run `config/migrate-*.js` scripts, but no universal automatic
  migration framework. Add a migration for data/index changes using the matching workflow; never edit an applied migration
  incompatibly.
- Never run production migrations, destructive queries, index drops, or data rewrites without explicit authorization,
  backup/recovery plan, reviewed scope, and dry-run/batch safety where available. Use expand-and-contract for breaking
  changes when risk warrants it.

Before a model change, identify readers/writers, schema/index impact, defaults/nullability, backfill, deployment
compatibility, recovery, and tests. Keep sensitive values out of logs, fixtures, and docs.

## 11. Redis and Distributed State Rules

**Verified:** Redis is optional (`USE_REDIS`); `USE_REDIS_STREAMS` separately controls resumable stream storage and
defaults from `USE_REDIS`. Redis supports cache/session/rate-limit paths, selected auth/config and MCP caches,
leader/concurrency coordination, and resumable generation jobs/events. Fallbacks are subsystem-specific. Durable schedules
and Agent Trigger Delivery state use MongoDB leases/documents/indexes; this is not a Redis queue-broker architecture.

| Conditional mechanism | Mandatory review |
|---|---|
| Cache | Preserve namespace, TTL, serialization, invalidation, stale/miss behavior, negative caching where used, stampede handling, and outage behavior. Cache only with a clear lifecycle/consistency need. |
| Session/auth | Preserve expiration, logout, revocation, and privilege-change semantics. The ordinary API is JWT-based; inspect the exact route before assuming server-side sessions. Never log tokens. |
| Rate/concurrency | Preserve scope and atomic counters. Determine current fail-open/closed behavior; do not change admission during outage implicitly. |
| Leader/lock | Prefer database constraints/atomic updates when sufficient; do not add locks casually. Use an existing lock primitive/library; verify ownership, lease, fencing, and safe release. Never release another process's lock. |
| Stream/job | Preserve identity, replay, retention, acknowledgement, terminal ordering, and cross-replica behavior. Process-local storage is not shared storage. |
| Background queue | Follow Mongo lease, idempotency, ordering, and retry contracts. Do not add Bull/BullMQ, a second queue, or Redis as an undocumented source of truth. |

Follow existing namespacing and `REDIS_KEY_PREFIX` / `REDIS_KEY_PREFIX_VAR`. Document operational keys/data format; never
put secrets or personal data in keys. Define temporary-data TTL and bound cardinality/memory; do not scan the full
keyspace in request paths. For Redis Cluster, respect hash slots; independent commands are not atomic, so use the
established Lua/transaction/atomic-command pattern when correctness spans operations.

For every new Redis dependency decide whether outage must fail, use an authoritative fallback, skip an optional cache, or
retry under a bound. Never silently treat outage as success if it can bypass authorization, lose required work, or corrupt
state. For multiple replicas, verify cross-replica requirements: resumable jobs/scheduling need shared stream visibility
unless the deployment explicitly asserts single-process operation. Inspect Compose/Helm/runtime flags; do not assume Redis
persistence. Never run `FLUSHALL`, `FLUSHDB`, or broad key deletion without explicit authorization.

## 12. Authentication, Authorization, and Security

- Reuse established server authentication, authorization, capability, role-permission, resource ACL, tenant-context,
  origin/session, and rate-limit mechanisms. Auth is per route/router: a route is not protected merely because neighbors
  are.
- Enforce permissions, ownership, and tenant boundaries server-side before returning data or acting. Client role checks
  are presentation only. Never trust browser-supplied IDs, roles, permissions, or ownership claims.
- Validate all untrusted data. Treat prompts, uploads, tool/provider/MCP outputs, and user-controlled URLs as untrusted.
  Use safe DB access and the existing SSRF-safe outbound fetch/connect checks.
- Preserve CORS, CSRF/origin, cookie/session, TLS, CSP/security headers, upload/path/MIME/size, and resource-limit
  protections. Do not disable TLS checks or security middleware to fix development.
- Never commit `.env` secrets, credentials, signing/encryption keys, private keys, or real provider tokens. Keep examples
  sanitized; do not print secret values during audits.
- Do not log tokens, session IDs, passwords, credentials, prompts, uploaded content, or personal data. Bound payloads,
  resource use, and request rates where applicable.
- A security-sensitive change MUST state trust boundary, attack surface, affected principals, fail-open/closed behavior,
  and verification strategy; add authorization and negative tests.

The security/limitations guides record legacy inconsistencies (including raw error disclosure and CORS/CSP/origin
configuration risks). These are risk notes, not approved patterns. Verify live code; do not copy a gap or silently change
security policy in unrelated work.

## 13. Error Handling, Logging, and Observability

- Follow existing result/error and HTTP/SSE/job contracts. Return plain values on success; reserve `null` for documented
  absence. Represent expected recoverable failures with the owning service's typed/discriminated result and a stable
  machine-readable code when callers must distinguish them; preserve existing shapes rather than mixing `ok`, `valid`,
  and bare `{ message }` contracts. Throw unexpected operational failures and violated invariants.
- Catch only where the caller can recover or translate. Never turn an operational failure into a truthy record, `null`, or
  an HTTP-200-shaped result. At request/stream/job boundaries map failures to established statuses and approved safe
  codes/details; localize actionable user-facing copy by stable code in the UI.
- Use `getSafeErrorMetadata` / `getSafeErrorText` from `packages/api/src/utils/errors.ts` where appropriate; when errors
  can echo submitted content, prefer safe metadata over error text. Never expose raw exceptions, stacks, query text,
  arbitrary `error.message`, provider payloads, credentials, request content, or sensitive metadata.
- Log useful safe context once at the owning boundary; use request IDs/metrics where supported. Distinguish
  transient/permanent errors and retry only with bounded, idempotent semantics. Do not swallow errors, dump payloads, or
  create unbounded/high-cardinality logs.

Examples of existing result patterns include `packages/api/src/admin/auditLog.ts`,
`packages/api/src/agents/openai/service.ts`, and `packages/api/src/skills/sync/errors.ts`; inspect before reuse. Do not
extend the query-failure `{ message }` pattern in `packages/data-schemas/src/methods/prompt.ts`.

For bug fixes identify root cause and verify the corrected behavior; do not suppress symptoms or fabricate success.

## 14. Testing and Validation

Inspect workspace scripts, test config, and CI before selecting commands. Jest is used across workspaces and Playwright
for E2E. Run focused checks first, then broader validation in proportion to risk. Do not assume a root script covers every
workspace.

| Purpose | Verified command |
|---|---|
| Scoped repository checks | `npm run static-checks` |
| CI-like diff checks | `npm run static-checks -- --against <verified-base-ref>` |
| Slower type/config/i18n/dependency gates | `npm run static-checks:full` |
| Turborepo production build | `npm run build` |
| Legacy backend tests | `npm run test:api` |
| React app tests | `npm run test:client` |
| `packages/api` tests | `npm run test:packages:api` |
| `packages/data-provider` tests | `npm run test:packages:data-provider` |
| `packages/data-schemas` tests | `npm run test:packages:data-schemas` |
| Config migration tests | `npm run test:config` |
| `packages/client` tests | `cd packages/client && npm run test:ci` |
| Mock E2E / Redis streaming | `npm run e2e:mock`; `npm run e2e:mock:redis` or `npm run e2e:mock:redis:transport` |
| Startup/auth/config/file/message-loading performance | `npm run lighthouse` |

Run focused Jest in the owning workspace (for example, `cd packages/api && npx jest src/<feature>/<file>.spec.ts`). The
root `test:all` script omits `packages/client`; run that workspace explicitly when changed. Integration tests may require
MongoDB, Redis, providers, or browser binaries; report prerequisite failures as blockers, not code failures.

`tsdown` builds for `packages/api`, `packages/client`, and `packages/data-schemas` do not typecheck. Run applicable
typechecks after building package dependencies:

```sh
npx tsc --noEmit -p packages/api/tsconfig.json
npx tsc --noEmit -p packages/data-schemas/tsconfig.json
npx tsc --noEmit -p packages/data-provider/tsconfig.json
npx tsc --noEmit -p packages/client/tsconfig.json
(cd client && npm run typecheck)
```

`packages/client/tsconfig.json` excludes `*.test.*` and `*.spec.*`, so its typecheck does not cover those files; run its
Jest tests separately. From the root, `npm run build:data-provider` builds the shared API package and runs `tsc`; it does
not replace other package checks. Sort imports only in touched files with `npm run sort-imports -- <paths>`; no-argument
`sort-imports` rewrites all source roots.

Add regression tests for bug fixes; cover important failure, empty/missing, authorization, retry, and cancellation paths.
Prefer real logic, injected dependencies, and spies; use `mongodb-memory-server` for DB behavior and the real MCP SDK
where practical. Mock uncontrollable external HTTP rather than internal layers. For React app component tests, follow
colocated `__tests__` patterns and `client/test/layout-test-utils.tsx` where applicable. Never remove/weaken tests, install
dependencies, or alter the development environment to get a pass without authorization. Explain skipped checks, observed
failures, and pre-existing failures. Do not submit/commit known failing code. Run `npm run lighthouse` for
startup/auth/config/file/message-loading changes; its CI lane adds 250 ms per Mongo query and checks visible conversation
LCP. Follow `e2e/lighthouse/README.md` for reproduction/diagnosis; never raise its budget to hide a regression.

## 15. Documentation Requirements

Keep docs proportional; update the relevant existing guide and preserve the distinction between verified facts, inference,
and unknowns. Create/update these records when warranted:

| Document | Content |
|---|---|
| `ARCHITECTURE.md` | Purpose/rationale, affected modules, data flow, API/persistence, security/failures, useful Mermaid diagrams, upstream risk. |
| `CUSTOM_CHANGELOG.md` | Date, tag, rationale, new files, changed upstream files, API/DB and deployment/migration impact. |
| `CUSTOMIZATIONS.md` | Custom modules, modified upstream files/reasons, feature tags/markers, extension points, tests/docs, known merge risks. |

These documents were absent in the inspected checkout. Do not create them for this governance-only change. Create
`CUSTOMIZATIONS.md` when the first relevant customization/upstream edit needs an inventory; create the others only for
significant features, architecture changes, or established workflow requirements. Keep the inventory current after
creation.

## 16. Database, Redis, and API Change Documentation

Document changes to persistent/shared state in the appropriate existing guide or customization record; link to detail
rather than duplicating a full schema/API reference.

- **Database:** schema/migration and index/constraint impact, old-data compatibility, backfill, deployment order,
  rollback/recovery.
- **Redis:** key pattern/data format, TTL/invalidation, consistency, outage/fallback, and queue/session/lock lifecycle
  where relevant.
- **API:** routes/contracts, request/response, authz, errors/status, side effects, retries, caller compatibility.
- **Configuration:** schema/env field, default, validation, sanitized example, operator/deployment behavior.

Update types, validators, migrations, tests, and docs together. A shared contract change is incomplete while any caller
still expects the old shape.

## 17. Dependency and Configuration Rules

- Reuse existing packages first. Before proposing a dependency, explain need, alternatives, maintenance/security cost,
  runtime/bundle impact, and license. Get explicit approval before add/remove/material upgrade.
- Do not change Node/package manager/framework/runtime versions or production-affecting defaults without approval. The
  verified baseline is Node `24.16.0` (`.nvmrc`) and npm `11.13.0` workspaces. Keep `package-lock.json` consistent after
  an approved manifest change.
- Never hardcode secrets or environment-specific internal addresses; keep templates sanitized and do not print secret
  values during inspection.
- Preserve configuration parsing/validation/defaults. New `librechat.yaml` fields belong in
  `packages/data-provider/src/config.ts` `configSchema` and the corresponding load path, examples, tests, and operator
  docs. Add env-only switches only for justified process-level controls and document name, allowed values, default, and
  failure behavior.
- Do not use `npm run update` to sync development branches; it is the self-host deployment updater. Do not change Docker,
  Compose, Helm, or CI as unrelated cleanup.

## 18. Upstream Synchronization and Merge Safety

Keep upstream changes as the baseline; isolate custom behavior so it can be reviewed and reapplied. Do not assume the fork
is a clean patch on one upstream release: verify history and changed files. Before syncing:

1. Check `git status`, remote URLs, local/remote branches, target, and merge base. Do not assume remote name `upstream` or
   branch name `main`/`dev`.
2. Review divergence/files before changing refs; preserve the working tree. Use a dedicated integration branch when
   appropriate, but create/switch branches only with authorization.
3. Understand both sides of every conflict. Never blindly accept ours/theirs, discard upstream improvements, or preserve
   obsolete custom code only because it is marked.
4. Reapply custom behavior at the narrowest safe extension point; check marker pairs and inventory.
5. Revalidate imports, routes, API contracts, auth/tenant boundaries, schemas/indexes, streams/workers, deployment, and
   frontend consumers.
6. Run affected workspace checks, update inventory/docs, review the final diff, and report migration needs, unresolved
   conflicts, and upstream risk.

At preparation time only `origin` for the Eleven-Chat fork was configured; no canonical LibreChat upstream remote or
`origin/dev` ref was available, and the checkout was shallow. This is a snapshot; verify again before a sync. For a public
LibreChat contribution, read `.github/CONTRIBUTING.md` and current maintainer instructions: its normal `dev` and
maintainer-directed `canary` process is distinct from synchronizing this fork. Ordinary upstream work targets `dev`, not
released `main`; since `gh pr create` defaults to `main`, specify `--base dev` when appropriate. Check current base, diff,
and reviewed head before a maintainer-directed canary retarget; never promote canary work by assumption. Follow its
issue/assignment process; closing keywords do not close issues on non-default branches, so close resolved issues manually.
Report security vulnerabilities through its private channel. Release-bound branches and `target: main` exceptions are not
ordinary backports. Worktrees share a stash stack; avoid bare `git stash pop` and preserve the specific worktree state.

Do not merge, reset, rebase, force-push, or rewrite history without authorization. Write PR descriptions for readers new to
the work: trigger, behavior before/after, and the mechanism; use a focused diff, shallow file tree, call tree, or sequence
and follow `.github/pull_request_template.md`.

A clean review is one signal, not completion. For PR review, read inline threads themselves and audit each against the
current code. After each round, run focused tests and typechecks, push only when authorized, and request the next review
against the exact current remote head; a clean review of an earlier head says nothing about a later push. Reply on resolved
threads with the fixing change and evidence. After two actionable rounds, reassess subsystem invariants (identity, auth,
persistence, retry, cancellation, cleanup, and mixed-version behavior) instead of patching findings serially. Report the
pushed head, local checks, CI/review state, accepted/rejected findings and reasons, and unavailable checks.

## 19. Agent Workflow and Approval Boundaries

**Before:** read root/nested instructions and any `CUSTOMIZATIONS.md`; check `git status`; inspect source, callers, tests,
config, and relevant guides; map the affected boundaries. Use an available code graph/Graphify index to navigate when
useful, but verify current source and graph freshness; a graph is never source-of-truth. Present the Section 3 plan and
proposed structure for non-trivial features and wait for approval. Routine authorized fixes do not need another approval
round.

**During:** keep edits focused, preserve unrelated work, and avoid speculative refactors/destructive commands. Do not
install dependencies, run migrations, change infrastructure, or alter Git history without authorization. Explain
deviations and keep custom code isolated.

**After:** review the full diff for unrelated changes, secrets, generated output, markers, and contract changes; run
applicable tests/typechecks/builds; verify cross-layer/security consistency; update docs; report files, checks actually
run, risks, blockers, and unresolved work. Wait for explicit authorization before committing, pushing, merging, deploying,
or opening a PR. Do not ask for approval on routine decisions already covered here; clarify conflicts affecting safety,
correctness, compatibility, or irreversible actions.

## 20. Forbidden Practices

Unless explicitly requested and the applicable safety, data-preservation, compatibility, and review requirements are
satisfied, agents MUST NOT:

- Rewrite large upstream modules for small features, blindly copy generated code, or delete upstream functionality because
  customization is inconvenient.
- Change unrelated UI, logic, contracts, schemas, defaults, or deployment behavior.
- Bypass auth, authorization, tenant scope, validation, rate limits, or security controls; use Redis as an undocumented
  source of truth; or add unnecessary duplicate clients/stores/caches/queues.
- Make destructive schema changes without migration/recovery; run production migrations, flush Redis, or execute
  `FLUSHALL`, `FLUSHDB`, broad key deletion, or equivalents without approval.
- Swallow errors, fabricate success, expose raw errors/secrets, disable TLS/security protections, or hardcode credentials.
- Introduce undocumented dependencies, circular imports, runtime monkey-patches, or breaking contracts without approval
  and a migration plan.
- Leave broken markers, stale inventory entries, unexplained upstream edits, or weakened tests.
- Claim unperformed verification or automatically commit, push, merge, rebase, reset, deploy, or rewrite history.

## 21. Quick Reference

| Action | Mandatory rule |
|---|---|
| Adding a feature | Inspect architecture, plan, isolate custom logic, and test observable behavior. |
| Modifying upstream code | Minimize diff, mark where syntax permits, document rationale. |
| Changing frontend behavior | Follow UI/state patterns; cover loading/errors, accessibility, localization, and consumers. |
| Adding an API endpoint | Validate input, enforce server authorization, preserve contracts, test errors/side effects. |
| Changing database schema | Use established migration workflow, preserve data, assess compatibility/recovery. |
| Adding Redis behavior | Document keys, TTL, consistency, failure handling, lifecycle, and applicable modes. |
| Fixing a bug | Identify root cause and add regression coverage. |
| Adding a dependency | Obtain approval and document rationale, alternatives, maintenance/security cost. |
| Changing security behavior | Preserve trust boundaries and verify authorization with negative tests. |
| Merging upstream | Inspect divergence, preserve upstream improvements, validate custom behavior, update inventory. |
| Completing a feature | Review diff, run checks, update docs, report risks/unresolved issues. |
| Committing or deploying | Obtain explicit authorization. |

If any rule conflicts with another, Golden Rules take precedence. When in doubt, ask the developer.
