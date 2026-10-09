# 10 — Feature Development Guide

How to add features to Eleven-Chat **the way the existing code already does it**. Every workflow
below points at real files and real patterns in this repository; nothing here introduces a new
library, framework, or convention. When the code base shows two coexisting styles, this guide
says so and names the one that matches current contributor guidance in
[`AGENTS.md`](../../AGENTS.md).

> **Read [`AGENTS.md`](../../AGENTS.md) first.** It is the canonical contributor contract
> (branching, module boundaries, config levers, error disclosure, client state ownership,
> verification). This guide is the *applied* version of it: where AGENTS.md states a rule, this
> guide shows which existing files embody it and in what order to touch them.

**Evidence labels used throughout**

| Label | Meaning |
|---|---|
| **Verified** | Read directly in the source during the research for this guide (file paths given). |
| **Inferred** | Reasoned from adjacent verified code; not traced line by line. |
| **Unknown** | Not established; treat as an open question, check the code before relying on it. |
| *Illustrative* | A worked example **built from** real patterns, describing code that does **not** exist in the repo. Every illustrative block is labelled as such. Code blocks marked "verbatim" are copied from the repo. |

Line numbers drift; treat any `file:line` as "last known location" and search by symbol name.

**Related guides:** [00 Overview](./00-overview.md) · [02 Backend](./02-backend.md) ·
[03 Frontend](./03-frontend.md) · [05 Database](./05-database.md) ·
[06 Business logic](./06-business-logic.md) ·
[09 Background processing](./09-background-processing.md) ·
[11 Testing & debugging](./11-testing-debugging.md)

---

## Contents

0. [Ground rules and the reference vertical slice](#0-ground-rules-and-the-reference-vertical-slice)
1. [Adding a new backend endpoint](#1-adding-a-new-backend-endpoint) — *full end-to-end example*
2. [Adding or modifying business logic](#2-adding-or-modifying-business-logic)
3. [Adding a database entity or field](#3-adding-a-database-entity-or-field) — *full end-to-end example*
4. [Adding a database migration](#4-adding-a-database-migration)
5. [Adding a Redis-backed capability](#5-adding-a-redis-backed-capability)
6. [Adding a frontend page](#6-adding-a-frontend-page)
7. [Adding a frontend component](#7-adding-a-frontend-component)
8. [Connecting a UI action to a backend operation](#8-connecting-a-ui-action-to-a-backend-operation) — *real trace + full end-to-end example*
9. [Form validation, loading and error states](#9-form-validation-loading-and-error-states)
10. [Adding background processing](#10-adding-background-processing)
11. [Writing unit and integration tests](#11-writing-unit-and-integration-tests)
12. [Updating API and developer documentation](#12-updating-api-and-developer-documentation)
13. [Documentation maintenance checklist](#13-documentation-maintenance-checklist)
14. [Pre-PR checklist](#14-pre-pr-checklist)

---

## 0. Ground rules and the reference vertical slice

### 0.1 Where code goes (from AGENTS.md "Module boundaries and configuration")

| Layer | Workspace | Holds | Must not hold |
|---|---|---|---|
| Express wiring (legacy CJS) | `api/` | `require`s, route registration, middleware order, the call into a TS factory | Branches, helpers, validation, service calls — "Minimum means how much behavior `/api` gains, not how small the diff is" |
| Backend behavior (TS) | `packages/api` (`@librechat/api`) | Handlers, services, policy, orchestration; receives its DB methods/config/clients as arguments | Imports of app singletons; new static singletons (the `packages/api/src/mcp` singletons are "the shape to stop extending") |
| Database contracts (TS) | `packages/data-schemas` (`@librechat/data-schemas`) | Schemas, model factories, `create<Domain>Methods(mongoose)` query methods, migrations | HTTP concerns, localization |
| Shared API surface (TS) | `packages/data-provider` (`librechat-data-provider`) | `configSchema`, endpoint URL builders, `dataService` functions, request/response types, React Query keys | Mongoose types |
| Shared UI primitives | `packages/client` (`@librechat/client`) | App-agnostic components (`Button`, `Dialog`, `DropdownPopup`, `Skeleton`, `Spinner`, toast) | Domain knowledge (conversations, agents, …) |
| React app | `client/` | Pages, feature components, React Query hooks, Jotai/Recoil state | Mongoose types, raw palette colors |

The build dependency chain is `librechat-data-provider → @librechat/data-schemas →
@librechat/api / @librechat/client` (Verified, `turbo.json`). `api/` consumes `@librechat/api` and
`@librechat/data-schemas` through their built `dist/` (`"main": "dist/index.cjs"`, Verified), so
**after editing a TS package you must rebuild it before the Express server sees the change**:

```bash
npm run build:data-provider   # also type-checks (tsdown && tsc)
npm run build:data-schemas    # tsdown only — no type check
npm run build:api             # tsdown only — no type check
npm run build:client-package  # tsdown only — no type check
npm run build:packages        # all four, in dependency order
```

Because three of those builds skip type checking (Verified, each `package.json`), always run
`npx tsc --noEmit` inside every workspace you changed (AGENTS.md "Verification").

### 0.2 The reference vertical slice: user preferences (Verified, end to end)

One small, recent feature touches **every** layer and follows every AGENTS.md rule. Use it as the
map for most workflows below: "the stateful-code-environment preference" — a user picks a default
code workspace in Settings and the server persists it.

```mermaid
flowchart TD
  subgraph client["client/ (React)"]
    C1["Nav/Settings/StatefulWorkspaceDefault.tsx<br/>Select + useToastContext"]
    C2["data-provider/Auth/mutations.ts<br/>useUpdateUserPreferencesMutation<br/>onSuccess → setQueryData([QueryKeys.user])"]
    C3["Nav/Settings/__tests__/StatefulWorkspaceDefault.spec.tsx"]
  end
  subgraph dp["packages/data-provider"]
    D1["api-endpoints.ts<br/>userPreferences() → /api/user/preferences"]
    D2["data-service.ts<br/>updateUserPreferences() → request.patch"]
    D3["keys.ts<br/>MutationKeys.updateUserPreferences"]
    D4["types.ts<br/>TUpdateUserPreferencesRequest/Response"]
    D5["config.ts configSchema<br/>endpoints.agents.statefulCodeSessions.allowedEnvironments"]
  end
  subgraph api["api/ (CJS wiring)"]
    A1["server/routes/user.js<br/>router.patch('/preferences', requireJwtAuth, configMiddleware, handler)"]
  end
  subgraph pkgapi["packages/api"]
    P1["src/user/preferences.ts<br/>createUserPreferencesHandler(deps)"]
    P2["src/user/preferences.spec.ts"]
  end
  subgraph ds["packages/data-schemas"]
    S1["src/methods/user.ts<br/>updateUserStatefulCodeEnvironment()<br/>+ invalidateAuthUserDocCache"]
    S2["src/methods/user.methods.spec.ts<br/>(mongodb-memory-server)"]
  end
  C1 --> C2 --> D2 --> D1
  D2 -.HTTP PATCH.-> A1 --> P1 --> S1
  P1 -. reads req.config .-> D5
```

| Layer | File | What to copy from it |
|---|---|---|
| DB method | `packages/data-schemas/src/methods/user.ts` (`updateUserStatefulCodeEnvironment`) | Plain args in, `IUser \| null` out (null = documented absence), `runValidators: true`, **auth user-doc cache invalidation** after a user mutation |
| DB test | `packages/data-schemas/src/methods/user.methods.spec.ts` | Real `MongoMemoryServer`; tests success, `null` for missing user, and cache invalidation |
| Handler | `packages/api/src/user/preferences.ts` | `create<Thing>Handler(deps)` factory with a typed `Deps` interface; 401 → 400 → 403 (config-gated) → 404 → 200; fixed safe 500 message |
| Handler test | `packages/api/src/user/preferences.spec.ts` | Injected `jest.fn()` deps, hand-built `req`/`res`; no module mocking |
| Wiring | `api/server/routes/user.js` | Import factory from `@librechat/api`, inject `~/models` method, mount with `requireJwtAuth` + `configMiddleware` |
| Config | `packages/data-provider/src/config.ts` | `allowedEnvironments` lever read through `req.config` |
| Client API | `packages/data-provider/src/{api-endpoints,data-service,keys,types}.ts` | URL builder, thin `request.patch`, `MutationKeys` entry, request/response types |
| Client hook | `client/src/data-provider/Auth/mutations.ts` | `useMutation([MutationKeys.x], fn, { ...options, onSuccess: patch cache then call caller's onSuccess })` |
| Component | `client/src/components/Nav/Settings/StatefulWorkspaceDefault.tsx` | `useLocalize` for copy, `@librechat/client` primitives, toast on success/error, rollback local state on error, ARIA labelling |
| Component test | `client/src/components/Nav/Settings/__tests__/StatefulWorkspaceDefault.spec.tsx` | Colocated `__tests__`, mocked hook boundary |

The handler verbatim (Verified, `packages/api/src/user/preferences.ts`, abridged only by `…`):

```ts
// verbatim excerpt — packages/api/src/user/preferences.ts
export interface UserPreferencesHandlerDeps {
  updateStatefulCodeEnvironment: (
    userId: string,
    environment: StatefulCodeEnvironment,
  ) => Promise<IUser | null>;
}

export function createUserPreferencesHandler(
  deps: UserPreferencesHandlerDeps,
): (req: UserPreferencesRequest, res: Response) => Promise<Response> {
  return async (req: UserPreferencesRequest, res: Response): Promise<Response> => {
    const userId = req.user?.id;
    if (!userId) {
      return res.status(401).json({ message: 'Unauthorized' });
    }
    // … 400 when the body value is not a known environment …
    const allowedEnvironments = resolveAllowedStatefulCodeEnvironments(
      req.config?.endpoints?.agents?.statefulCodeSessions?.allowedEnvironments,
    );
    if (!allowedEnvironments.includes(environment)) {
      return res.status(403).json({ /* … */ });
    }
    try {
      const updatedUser = await deps.updateStatefulCodeEnvironment(userId, environment);
      if (!updatedUser) {
        return res.status(404).json({ message: 'User not found' });
      }
      return res.status(200).json({ updated: true, preferences: { /* … */ } });
    } catch (error) {
      logger.error('[UserPreferences] Error updating preferences:', error);
      return res.status(500).json({ message: 'Failed to update user preferences' });
    }
  };
}
```

And its wiring (Verified, `api/server/routes/user.js`):

```js
// verbatim excerpt — api/server/routes/user.js
const { createUserPreferencesHandler } = require('@librechat/api');
const { updateUserStatefulCodeEnvironment } = require('~/models');

const updateUserPreferences = createUserPreferencesHandler({
  updateStatefulCodeEnvironment: updateUserStatefulCodeEnvironment,
});

router.patch('/preferences', requireJwtAuth, configMiddleware, updateUserPreferences);
```

### 0.3 Commands you will run in every workflow

| Purpose | Command | Source |
|---|---|---|
| Focused tests, owning workspace only | `cd <workspace> && npx jest <path-or-pattern>` (or `npm run test:ci -- <pattern>`) | AGENTS.md "Testing"; scripts Verified in each `package.json` |
| Type check (build does not) | `cd <workspace> && npx tsc --noEmit` | AGENTS.md "Verification" |
| Import order, scoped | `npm run sort-imports -- <paths you touched>` | AGENTS.md — **no-arg run rewrites every source root** |
| Reproduce PR static checks | `npm run static-checks -- --against origin/dev` | AGENTS.md; `scripts/static-checks.mts` |
| Slow gates (TS projects, config tests, i18n, depcheck) | `npm run static-checks:full` | Verified, root `package.json` |
| Startup/auth/config/file/message-loading changes | `npm run lighthouse` | AGENTS.md; adds 250 ms per Mongo query |

Avoid `npm run test:all` for routine work (runs every workspace sequentially). Note that the plain
`npm test` script in `packages/api`, `packages/data-schemas` and `client` runs Jest in `--watch`
mode (Verified), so prefer `npx jest <path>` for one-shot runs.

---

## 1. Adding a new backend endpoint

### Files to inspect first

- `api/server/routes/index.js` — pure pass-through barrel of ~45 route modules (Verified).
- `api/server/index.js` — the `app.use('/api/...', routes.x)` mount block (Verified, e.g.
  `app.use('/api/user', routes.user)`, `app.use('/api/presets', routes.presets)`). Route modules are
  required **late**, after `performStartupChecks()`, because rate limiters are built at `require`
  time from config (Verified, E2 §1).
- `api/server/routes/user.js` + `packages/api/src/user/preferences.ts` — the factory pattern (§0.2).
- `api/server/routes/skills.js` — second confirmation: `createImportHandler`,
  `createSkillUploadHandler`, `generateCheckAccess` imported from `@librechat/api`, wired with
  `~/models` functions (Verified, E2 §3f).
- `api/server/routes/convos.js` `POST /update` → `createRenameConversationHandler` in
  `packages/api/src/conversations/rename.ts` — the factory pattern with config + content filtering
  + a runtime dependency (`GenerationJobManager…bind(...)`) (Verified).
- `api/server/middleware/index.js` — middleware barrel (`requireJwtAuth`, `configMiddleware`,
  `strictConfigMiddleware`, validators, limiters).
- `api/server/middleware/roles/capabilities.js` — `requireCapability(...)`. **Import it directly**,
  not via the middleware barrel (the barrel deliberately omits it to avoid a circular require —
  Verified comment in `roles/index.js`).
- `packages/api/src/middleware/error.ts` (`ErrorController`) and `packages/api/src/utils/errors.ts`
  (`getSafeErrorMetadata`, `getSafeErrorText`).

### Recommended location

| Piece | Location |
|---|---|
| Handler / service logic | `packages/api/src/<feature>/<name>.ts`, exported through `packages/api/src/<feature>/index.ts` and `packages/api/src/index.ts` (`export * from './user'` etc., Verified) |
| Query / write | `packages/data-schemas/src/methods/<domain>.ts` (see [§3](#3-adding-a-database-entity-or-field)) |
| Route wiring | `api/server/routes/<name>.js` (new module) or an existing router file |
| Mount | `api/server/routes/index.js` + `api/server/index.js` |
| New lever (limit, timeout, toggle) | `packages/data-provider/src/config.ts` `configSchema` |
| Request/response types | `packages/data-provider/src/types.ts` (shared with the client) |

### The existing pattern to follow (Verified — E2 §8 "Where a new backend route is actually added today")

1. Write the DB method in `packages/data-schemas` and register it in `createMethods` — it then
   appears on `require('~/models')` (`api/models/index.js` wraps `createMethods(mongoose, deps)`).
2. Write a `create<Thing>Handler(deps)` factory in `packages/api` with a typed `Deps` interface.
   Dependencies are plain functions (DB methods, services), never Mongoose models or types.
3. In `api/server/routes/<name>.js`, build the handler once at module load by injecting
   `~/models` functions, then register it with auth/config/limiter middleware.
4. Add the module to `api/server/routes/index.js` and mount it in `api/server/index.js`
   (`[preAuthTenantMiddleware,]` only if it must be reachable pre-auth).
5. Add any lever to `configSchema` with a default that reproduces current behavior.
6. Tests: factory spec in `packages/api`, DB spec in `packages/data-schemas`, optional thin
   supertest route test in `api/server/routes/`.

**Both styles exist today (Verified).** Simple CRUD routes such as `GET /api/convos`,
`POST /api/convos/pin` and `POST /archive` are still inline `async (req, res) => {…}` handlers in
`api/server/routes/convos.js`. The factory-in-`packages/api` style is the one that matches AGENTS.md
and is the one to imitate. Do not copy the inline style just because the file you are editing uses
it.

### Validation and authorization

- **Authentication is per-router, not global** (Verified, E2 §2). Put `router.use(requireJwtAuth)`
  at the top of a new router, or add it per route as `user.js` does.
- **Middleware order matters for cost.** `/api/auth/login` runs `loginLimiter` and `checkBan`
  *before* the password strategy (Verified). On `requireJwtAuth`-first routers such as `convos.js`,
  unauthenticated floods pay full JWT verification first. If your endpoint is expensive or
  pre-auth, put a limiter from `api/server/middleware/limiters/` ahead of the work.
- **Resource ACLs**: use `canAccessResource` (`api/server/middleware/accessResources/`, bitmask
  1=view, 2=edit, 4=delete, 8=share) for shared resources; `validateConvoAccess` for
  conversation-scoped routes (used by `POST /api/convos/update`, Verified).
- **Admin routes**: gate with `requireCapability(SystemCapabilities.<FLAG>)`. `checkAdmin`
  (`api/server/middleware/roles/admin.js`) is defined but **not used by any route** (Verified, E2
  finding 1) — do not reintroduce it.
- **Config-gated authorization** belongs in the handler, reading `req.config` (attached by
  `configMiddleware`). Use `strictConfigMiddleware` (fail-closed) when a config read failure must
  block the request instead of falling back to base config (Verified, E1 §3).
- **Do not write a response before authorization succeeds** (AGENTS.md "Code style and
  performance").
- Request-body validation is hand-written type guards in the handler (`isStatefulCodeEnvironment`
  in `preferences.ts`; `typeof conversationId !== 'string'` in `rename.ts`) or a named validator
  middleware (`validateMessageReq`, `validateModel`, …). Some legacy controllers catch `ZodError`
  (`api/server/controllers/agents/v1.js`). **Inferred:** there is no single repo-wide request schema
  layer; follow the guard style of the nearest neighbour.

### Errors (AGENTS.md "Service failures and user-facing errors")

- Return a fixed, safe message on unexpected failure and log the real error server-side, exactly
  as `preferences.ts` (`'Failed to update user preferences'`) and `GET /api/convos`
  (`'Error fetching conversations'`) do.
- Expected failures the client must distinguish get **stable codes**:
  `rename.ts` returns `{ error: 'conversation_not_found' }` and
  `409 conversation_title_ownership_not_ready` (Verified).
- When logging an error that may echo submitted content, log `getSafeErrorMetadata(error)` (the
  rename handler's `logger` dep is typed to accept only that, Verified).
- Thrown Mongoose `ValidationError` → 400 and duplicate key `11000` → 409 are mapped centrally by
  `ErrorController` if you call `next(err)` (Verified, E2 §5).
- **Do not copy** `res.status(500).json({ error: error.message })`. E2 found it in
  `api/server/controllers/agents/v1.js`, `assistants/v1.js`, `assistants/v2.js`, `mcp.js`,
  `PluginController.js`, `tools.js`, `ModelController.js`, `routes/memories.js`,
  `routes/files/files.js` and others — these predate the policy and are known gaps.

### DB / Redis implications

- Request paths must reuse `req.user`/`req.config` instead of re-reading them, avoid serial DB reads,
  and start independent reads together with `Promise.all` — the in-repo example is
  `getConvosByCursor` in `packages/data-schemas/src/methods/conversation.ts` (Verified).
- Any handler that mutates user documents must cause **auth user-doc cache invalidation**
  (AGENTS.md "Backend auth cache"). Put that in the data-schemas method, as
  `updateUserStatefulCodeEnvironment` does, so every caller gets it.

### Tests to add

- `packages/api/src/<feature>/<name>.spec.ts` — each status branch with injected `jest.fn()` deps
  (template: `preferences.spec.ts`, `conversations/rename.spec.ts`).
- `packages/data-schemas/src/methods/<domain>.spec.ts` — real `MongoMemoryServer`.
- Optional `api/server/routes/<name>.test.js` — supertest wiring test (template:
  `api/server/routes/presets.test.js`, see [§11](#11-writing-unit-and-integration-tests)).
- If the endpoint is on the startup/auth/config/file/message-loading path: `npm run lighthouse`.

### Docs to update

[`02-backend.md`](./02-backend.md) (route table), the API reference if the suite has one,
`librechat.example.yaml` for any new config field, and this guide if you establish a new pattern.

### Common mistakes and debugging tips

- **Business logic in route files.** `api/server/routes/convos.js` contains retry loops
  (`readGenerationForDeletion`, `retryPostDeleteCancellation`), fan-out/drain orchestration
  (`confirmAgentGenerationsDrained`, `drainDeletedAgentGenerations`) and fencing
  (`withAgentOwnerDeletionFence`) written inline (Verified, E2 §4). It works, but it cannot be
  unit-tested without supertest and cannot be reused by other callers. Contrast
  `api/server/services/Files/routing.js`, explicitly commented *"Wiring only"*. New work follows
  `routing.js`, not `convos.js`.
- **Forgetting to rebuild** `packages/api` after editing it: the Express server runs the old
  `dist/`. Run `npm run build:api` (or `build:packages` if data-schemas changed too).
- **Importing `requireCapability` from the middleware barrel** returns an empty object at require
  time (circular require). Import from `~/server/middleware/roles/capabilities`.
- **Mongoose types in exported signatures** of `packages/api` (`FilterQuery`, `Types.ObjectId`,
  `Document`) — AGENTS.md says stop widening that leak; take plain typed objects.
- **Route returns 404 from the SPA** instead of your handler: you added the router file but not
  the `app.use` mount (the `/api` 404 handler `apiNotFound` runs after all mounts).

### End-to-end example A — a new endpoint (*Illustrative, built from real existing patterns*)

> *Illustrative.* "Saved conversation filters" does **not** exist in this repo. It is used across
> examples A (endpoint), B ([§3](#3-adding-a-database-entity-or-field), entity) and C
> ([§8](#8-connecting-a-ui-action-to-a-backend-operation), UI) so the three form one coherent
> feature. Each step names the real file it is modelled on.

Goal: `GET /api/saved-filters` lists the caller's saved filters; `POST /api/saved-filters` saves
one, capped by a configurable per-user maximum.

**Step 1 — config lever** (modelled on `interface.runningChatRename`, Verified three-touchpoint
pattern). An `interface` field must be added in **three** places, or it is silently dropped:

```ts
// Illustrative — packages/data-provider/src/config.ts, inside interfaceSchema = z.object({ … })
/** Most saved conversation filters a user may keep; 0 disables saving. */
savedFilterLimit: z.number().int().min(0).max(100).default(0),

// Illustrative — same file, in interfaceSchema's trailing .default({ … }) object
savedFilterLimit: 0,

// Illustrative — packages/data-schemas/src/app/interface.ts, inside loadDefaultInterface()
savedFilterLimit: interfaceConfig?.savedFilterLimit ?? defaults.savedFilterLimit,
```

The default `0` reproduces today's behavior (no saved filters). Add a case to
`packages/data-schemas/src/app/interface.spec.ts` (it already has a table test for
`runningChatRename`, Verified). Rebuild: `npm run build:data-provider`.

**Step 2 — handler factory** (modelled on `createUserPreferencesHandler` and
`createRenameConversationHandler`):

```ts
// Illustrative — packages/api/src/conversations/savedFilters.ts
import { logger } from '@librechat/data-schemas';
import type { Response } from 'express';
import type { ServerRequest } from '~/types';
import { getSafeErrorMetadata } from '~/utils/errors';

export interface SavedFilterInput {
  name: string;
  query: Record<string, string>;
}

export interface SavedFilterHandlerDeps {
  listSavedFilters: (userId: string) => Promise<Array<SavedFilterInput & { id: string }>>;
  countSavedFilters: (userId: string) => Promise<number>;
  createSavedFilter: (
    userId: string,
    input: SavedFilterInput,
  ) => Promise<SavedFilterInput & { id: string }>;
}

function isSavedFilterInput(value: unknown): value is SavedFilterInput {
  /* hand-written guard, as isStatefulCodeEnvironment does: non-empty name ≤ 64 chars,
     query is a flat string map */
}

export function createSavedFilterHandlers(deps: SavedFilterHandlerDeps) {
  const list = async (req: ServerRequest, res: Response): Promise<Response> => {
    const userId = req.user?.id;
    if (!userId) {
      return res.status(401).json({ error: 'unauthorized' });
    }
    try {
      return res.status(200).json({ filters: await deps.listSavedFilters(userId) });
    } catch (error) {
      logger.error('[SavedFilters] list failed', getSafeErrorMetadata(error));
      return res.status(500).json({ error: 'saved_filters_unavailable' });
    }
  };

  const create = async (req: ServerRequest, res: Response): Promise<Response> => {
    const userId = req.user?.id;
    if (!userId) {
      return res.status(401).json({ error: 'unauthorized' });
    }
    if (!isSavedFilterInput(req.body)) {
      return res.status(400).json({ error: 'invalid_saved_filter' });
    }
    const limit = req.config?.interfaceConfig?.savedFilterLimit ?? 0;
    if (limit === 0) {
      return res.status(403).json({ error: 'saved_filters_disabled' });
    }
    try {
      if ((await deps.countSavedFilters(userId)) >= limit) {
        return res.status(409).json({ error: 'saved_filter_limit_reached' });
      }
      return res.status(201).json(await deps.createSavedFilter(userId, req.body));
    } catch (error) {
      logger.error('[SavedFilters] create failed', getSafeErrorMetadata(error));
      return res.status(500).json({ error: 'saved_filters_unavailable' });
    }
  };

  return { list, create };
}
```

Export it from `packages/api/src/conversations/index.ts` (`export * from './savedFilters'`), as
`rename.ts` is exported today. A duplicate name hitting the unique index (example B) will throw a
`MongoServerError 11000`; either catch it here and return a stable `409 saved_filter_exists`, or
call `next(error)` and let `ErrorController` map it to 409.

**Step 3 — wiring** (modelled on `api/server/routes/user.js` and `presets.js`):

```js
// Illustrative — api/server/routes/savedFilters.js
const express = require('express');
const { createSavedFilterHandlers } = require('@librechat/api');
const { requireJwtAuth, configMiddleware } = require('~/server/middleware');
const { listSavedFilters, countSavedFilters, createSavedFilter } = require('~/models');

const router = express.Router();
const handlers = createSavedFilterHandlers({
  listSavedFilters,
  countSavedFilters,
  createSavedFilter,
});

router.use(requireJwtAuth);
router.get('/', handlers.list);
router.post('/', configMiddleware, handlers.create);

module.exports = router;
```

```js
// Illustrative — api/server/routes/index.js: add `const savedFilters = require('./savedFilters');`
// and `savedFilters,` to module.exports.
// Illustrative — api/server/index.js, in the mount block next to app.use('/api/presets', …):
app.use('/api/saved-filters', routes.savedFilters);
```

No branch, helper or validation was added to `api/` — only requires, construction and mounting.

**Step 4 — tests**: `packages/api/src/conversations/savedFilters.spec.ts` covering 401, 400, 403
(limit 0), 409 (at limit), 201, and 500 with an assertion that the response body is the fixed code
(not the thrown message); data-schemas spec from example B; optional supertest test modelled on
`presets.test.js`.

**Step 5 — verify**: `npm run build:data-provider && npm run build:data-schemas && npm run build:api`,
`npx tsc --noEmit` in `packages/data-provider`, `packages/data-schemas`, `packages/api`,
`npm run sort-imports -- <touched paths>`, `npm run static-checks -- --against origin/dev`.

---

## 2. Adding or modifying business logic

See [06 Business logic](./06-business-logic.md) for the catalogue of existing rules (message tree
walking, sealed code-environment decisions, enforced model specs, credit reservation, scheduled-run
admission, tool-approval layering, rate limits, multi-tenancy) and where each is enforced and tested.

### Files to inspect first

- The rule's current home, found via [06](./06-business-logic.md). Most live under
  `packages/api/src/<domain>/` (`agents/`, `conversations/`, `schedules/`, `auth/`, `acl/`,
  `app/`), with persistence in `packages/data-schemas/src/methods/`.
- The rule's existing spec next to it (`*.spec.ts`).
- `CONTEXT.md` at the repo root for fork-specific terms (subagent thread, queued turn, trigger
  capability shield, …).

### Recommended location

`packages/api/src/<domain>/` for policy and orchestration; `packages/data-schemas/src/methods/` only
for logic that is inherently about the query (e.g. `saveConvo` stripping server-owned fields before
writing, `deleteConvos` deleting roots before children). Never `api/`.

### Pattern to follow (Verified)

- **Factories with injected dependencies**: `createUserPreferencesHandler(deps)`,
  `createRenameConversationHandler(deps)`, `createAppConfigService(deps)`
  (`packages/api/src/app/service.ts`), `createSubagentThreadTaskStore(...)` wired in
  `api/server/services/Endpoints/agents/subagentThreadStore.js` with DB methods injected.
- **Contract shapes** (AGENTS.md): plain value on success; `null` only for documented absence
  (`getConvo`); typed results for expected failures (`deleteConvos` returns
  `{ acknowledged, deletedCount, messages, conversationIds }` and an `allowEmpty` option chooses
  between error and empty success); throw for operational failures.
- **Do not extend** `packages/data-schemas/src/methods/prompt.ts`'s `{ message }`-on-failure
  contract — AGENTS.md names it as the negative example.
- **Integrations arrive as interfaces** (`IJobStore` + `InMemoryJobStore`/`RedisJobStore`;
  `ServerConfigsRepositoryInterface`; injected `fetchFn` in `skills/sync/github.ts`). A second
  implementation is a new argument, not a new `if`.

### Sequence

1. Locate the rule and its tests ([06](./06-business-logic.md)).
2. Write a failing spec for the missed behavior (AGENTS.md: "keep fixes small and test the missed
   behavior").
3. Change the `packages/api` module; if it needs new data, add a data-schemas method first.
4. If the rule introduces a lever, add a `configSchema` field ([§1 step 1](#end-to-end-example-a--a-new-endpoint-illustrative-built-from-real-existing-patterns)).
5. If callers live in `api/`, change only the injected arguments there.

### Validation / authorization considerations

Several rules are enforced in two places today (E9 §9: enforced-agent-id resolution computed in two
call sites; file-size limits as both constants and an admin override). When you change one, search
for the other.

### Common mistakes

- Moving logic *into* `api/` because the caller is there. Extract it to `packages/api` and inject.
- Reaching for `MCPManager.getInstance()`-style singletons in new code; accept the dependency.
- Catching an exception and returning `null`/`{ message }` — an outage then looks like "no data".

---

## 3. Adding a database entity or field

### Files to inspect first

- `packages/data-schemas/src/schema/<name>.ts` — pure `Schema<T>` + `schema.index(...)`, no model
  registration (Verified, E4 §3).
- `packages/data-schemas/src/models/<name>.ts` — `create<Name>Model(mongoose)`: applies
  `applyTenantIsolation(schema)` and registers idempotently (Verified).
- `packages/data-schemas/src/models/index.ts` — `createModels(mongoose)` composes all factories.
- `packages/data-schemas/src/methods/<domain>.ts` + `methods/index.ts` — `createMethods(mongoose, deps)`
  composes ~45 domain factories into one `AllMethods` object.
- `packages/data-schemas/src/types/<name>.ts` + `types/index.ts` — the `I<Name>` interface.
- `packages/data-schemas/src/schema/index.ts` — schema barrel.
- `api/db/models.js` (`createModels(mongoose)`) and `api/models/index.js` (`createMethods(mongoose, {…deps})`)
  — the two wiring call sites.

The smallest complete template is **Banner** (Verified): `schema/banner.ts`, `models/banner.ts`,
`methods/banner.ts`, `types/banner.ts`, each registered once in its barrel
(`schema/index.ts:8`, `models/index.ts:36,73,124`, `methods/index.ts:80,276,515,585`).

```ts
// verbatim — packages/data-schemas/src/models/banner.ts
export function createBannerModel(mongoose: typeof import('mongoose')): Model<IBanner> {
  applyTenantIsolation(bannerSchema);
  return mongoose.models.Banner || mongoose.model<IBanner>('Banner', bannerSchema);
}
```

### Schema conventions observed (Verified, E4 §3 and §7)

| Concern | Existing example | Rule of thumb |
|---|---|---|
| Tenant scoping | `tenantId: { type: String, index: true }` on nearly every schema; compound uniques include `tenantId` (`{ email: 1, tenantId: 1 }`) | Add `tenantId` and include it in unique indexes. `ToolFavorite` deliberately omits it, with an in-file comment explaining why — do the same if you deviate |
| Timestamps | `{ timestamps: true }` on most | Use it unless you have an explicit lifecycle field |
| Format validation | `User.email` `match: [/\S+@\S+\.\S+/, 'is invalid']` | |
| Custom validator | `PromptGroup.command` regex `validator` + `maxlength: [Constants.COMMANDS_MAX_LENGTH, …]` | |
| Conditional `required` | `Schedule.cadence.hour`: `required: isStructuredCadence` — and the comment warns that on a nested path `this` is the **document** | Read that comment before writing a nested validator |
| Path safety | `SkillFile.relativePath` rejects `..`, absolute paths, non-allowlisted chars | |
| Write-once | `immutable: true` on `File.metadata.runFile`, AuditLog leaves | |
| Invariants under concurrency | `ScheduleRun` partial unique indexes (`{scheduleId}` where `status:'started'`; `{capacitySlot}`) — "enforced by the DB rather than a racy count" | Prefer a partial unique index over read-then-check |
| Large/private fields | `select: false` (`User.password`, `User.pinnedOrder`, `Balance.reservations`) | Use for secrets and large display-only arrays |
| Expiry | TTL index on a `Date` field (`expiredAt`, `expiresAt`) | No soft-delete flag exists anywhere; use TTL or explicit hard delete |
| Relations | Mixed: some string matches (`Conversation.user`), some `ObjectId` refs (`File.user`) | Match the neighbouring entity's convention |

### Method conventions (Verified, E4 §4)

- `create<Domain>Methods(mongoose, deps?)` returns plain async functions that close over
  `mongoose.models.<Model>`; export `type <Domain>Methods = ReturnType<typeof create…>`.
- Use `.lean()` and return plain objects; `null` for documented absence; **throw** on operational
  failure (`getConvo` and `getBanner` both log and rethrow a generic `Error`).
- External dependencies (cache accessor, model-name matcher) come through `CreateMethodsDeps`
  (`getCache: getLogStores` in `api/models/index.js`) — the data layer never imports Redis.
- A user mutation calls `invalidateAuthUserDocCache(userId)` (Verified in
  `updateUserStatefulCodeEnvironment`).

### Adding a field to an existing entity (Verified template)

`updateUserStatefulCodeEnvironment` (`packages/data-schemas/src/methods/user.ts`) is the template:

```ts
// verbatim — packages/data-schemas/src/methods/user.ts
async function updateUserStatefulCodeEnvironment(
  userId: string,
  environment: StatefulCodeEnvironment,
): Promise<IUser | null> {
  const User = mongoose.models.User;
  const updated = await User.findByIdAndUpdate(
    userId,
    { $set: { 'personalization.statefulCodeEnvironment': environment } },
    { new: true, runValidators: true },
  ).lean<IUser>();
  if (updated) {
    await invalidateAuthUserDocCache(userId);
  }
  return updated;
}
```

Sequence for a new field: (1) add it to the `I<Name>` type and the schema (with `default` only if
existing documents should read as that value); (2) add a targeted `$set` method — do not reuse a
generic "save whatever the client sent" path (`saveConvo` strips server-owned fields precisely
because of this); (3) decide whether existing documents need a backfill ([§4](#4-adding-a-database-migration));
(4) if the field is client-visible, add it to the shared type in `packages/data-provider/src/types.ts`.

### DB implications to decide up front

- **Indexes and DocumentDB.** Index builds run in the background; `models/index.ts` attaches an
  `'index'` listener so a failed build (e.g. DocumentDB < 5.0 rejecting `partialFilterExpression`)
  is logged with a migration hint rather than silently leaving a uniqueness rule unenforced
  (Verified). Check logs after adding a partial index.
- **Changing an existing unique index** needs an explicit migration ([§4](#4-adding-a-database-migration));
  Mongoose will not drop the old one.
- **Transactions are optional.** Use `getTransactionSupport(...)` from
  `packages/data-schemas/src/utils/transactions.ts` and degrade without one, as
  `packages/api/src/acl/accessControlService.ts` does. Most multi-document writes here are
  idempotent ordered writes instead of transactions (Verified, E4 §5).
- **Search index.** Only `Conversation` and `Message` carry the Meilisearch plugin; a `meiliIndex`
  field elsewhere does nothing.
- **Hot-path reads.** New reads on startup/auth/config/file/message loading → `npm run lighthouse`.

### Tests to add

`packages/data-schemas/src/methods/<domain>.spec.ts` against a real `MongoMemoryServer`
([§11](#11-writing-unit-and-integration-tests)): success, documented absence, validation failure,
index uniqueness, tenant isolation (there are `*.tenant.spec.ts` siblings, e.g.
`accessRole.tenant.spec.ts`), cache invalidation for user mutations.

### Common mistakes

- Registering a model with `mongoose.model(...)` inside the schema file — registration belongs in
  `models/<name>.ts` so tenant isolation is applied exactly once.
- Forgetting one of the four barrels (schema, model, methods, types). Symptom:
  `require('~/models').yourMethod` is `undefined`, or `mongoose.models.X` is undefined at call time.
- Adding the field to the schema but not `runValidators: true` on the update — validators do not
  run on `findByIdAndUpdate` by default.
- Copying `import { createBannerMethods, type BannerMethods } from './banner'` (inline `type`) from
  `methods/index.ts`: AGENTS.md asks for standalone `import type` in code you write.

### End-to-end example B — a new entity (*Illustrative, built from real existing patterns*)

> *Illustrative.* `SavedFilter` does not exist. Modelled on Banner (files), ConversationTag
> (`{ tag, user, tenantId }` unique), and the user-method spec (tests).

```ts
// Illustrative — packages/data-schemas/src/types/savedFilter.ts
import type { Document } from 'mongoose';

export interface ISavedFilter extends Document {
  user: string;
  name: string;
  query: Record<string, string>;
  tenantId?: string;
  createdAt?: Date;
  updatedAt?: Date;
}
```

```ts
// Illustrative — packages/data-schemas/src/schema/savedFilter.ts
import { Schema } from 'mongoose';
import type { ISavedFilter } from '~/types';

const savedFilterSchema: Schema<ISavedFilter> = new Schema<ISavedFilter>(
  {
    user: { type: String, required: true },
    name: { type: String, required: true, trim: true, maxlength: 64 },
    query: { type: Map, of: String, default: {} },
    tenantId: { type: String, index: true },
  },
  { timestamps: true },
);

savedFilterSchema.index({ user: 1, name: 1, tenantId: 1 }, { unique: true });
savedFilterSchema.index({ user: 1, createdAt: -1 });

export default savedFilterSchema;
```

```ts
// Illustrative — packages/data-schemas/src/models/savedFilter.ts
import { Model } from 'mongoose';
import type { ISavedFilter } from '~/types';
import { applyTenantIsolation } from '~/models/plugins/tenantIsolation';
import savedFilterSchema from '~/schema/savedFilter';

export function createSavedFilterModel(mongoose: typeof import('mongoose')): Model<ISavedFilter> {
  applyTenantIsolation(savedFilterSchema);
  return (
    mongoose.models.SavedFilter || mongoose.model<ISavedFilter>('SavedFilter', savedFilterSchema)
  );
}
```

```ts
// Illustrative — packages/data-schemas/src/methods/savedFilter.ts
import type { Model } from 'mongoose';
import type { ISavedFilter } from '~/types';

type SavedFilterRecord = { id: string; name: string; query: Record<string, string> };

export function createSavedFilterMethods(mongoose: typeof import('mongoose')) {
  const model = () => mongoose.models.SavedFilter as Model<ISavedFilter>;
  const toRecord = (doc: ISavedFilter): SavedFilterRecord => ({
    id: String(doc._id),
    name: doc.name,
    query: Object.fromEntries(Object.entries(doc.query ?? {})),
  });

  async function listSavedFilters(user: string): Promise<SavedFilterRecord[]> {
    const docs = await model().find({ user }).sort({ createdAt: -1 }).lean<ISavedFilter[]>();
    return docs.map(toRecord);
  }

  async function countSavedFilters(user: string): Promise<number> {
    return model().countDocuments({ user });
  }

  async function createSavedFilter(
    user: string,
    input: { name: string; query: Record<string, string> },
  ): Promise<SavedFilterRecord> {
    const doc = await model().create({ user, name: input.name, query: input.query });
    return toRecord(doc.toObject());
  }

  return { listSavedFilters, countSavedFilters, createSavedFilter };
}

export type SavedFilterMethods = ReturnType<typeof createSavedFilterMethods>;
```

No `try/catch → return null` here: a query failure throws to the handler, which maps it to a safe
500 (AGENTS.md). `Map` is converted to a plain object so no Mongoose type leaks to `packages/api`.

Registration (each mirrors the Banner lines listed above):

```ts
// Illustrative barrel edits
// types/index.ts:   export * from './savedFilter';
// schema/index.ts:  export { default as savedFilterSchema } from './savedFilter';
// models/index.ts:  import { createSavedFilterModel } from './savedFilter';
//                   SavedFilter: ReturnType<typeof createSavedFilterModel>;   (in the models type)
//                   SavedFilter: createSavedFilterModel(mongoose),           (in createModels)
// methods/index.ts: import { createSavedFilterMethods } from './savedFilter';
//                   import type { SavedFilterMethods } from './savedFilter';
//                   SavedFilterMethods &                                      (in AllMethods)
//                   ...createSavedFilterMethods(mongoose),                    (in createMethods)
```

No change to `api/db/models.js` or `api/models/index.js` is needed — they already spread whatever
`createModels`/`createMethods` return, so `require('~/models').listSavedFilters` exists after
`npm run build:data-schemas`.

Spec skeleton (structure verbatim from `user.methods.spec.ts`'s `beforeAll`/`afterAll`/`beforeEach`):

```ts
// Illustrative — packages/data-schemas/src/methods/savedFilter.spec.ts
import mongoose from 'mongoose';
import { MongoMemoryServer } from 'mongodb-memory-server';
import { createSavedFilterModel } from '~/models/savedFilter';
import { createSavedFilterMethods } from './savedFilter';

let mongoServer: MongoMemoryServer;
let methods: ReturnType<typeof createSavedFilterMethods>;

beforeAll(async () => {
  mongoServer = await MongoMemoryServer.create();
  await mongoose.connect(mongoServer.getUri());
  createSavedFilterModel(mongoose);
  methods = createSavedFilterMethods(mongoose);
});

afterAll(async () => {
  await mongoose.disconnect();
  await mongoServer.stop();
});

beforeEach(async () => {
  await mongoose.connection.dropDatabase();
});

describe('saved filters', () => {
  test('lists only the caller’s filters, newest first', async () => { /* … */ });
  test('rejects a duplicate name for the same user', async () => {
    await mongoose.models.SavedFilter.syncIndexes();
    await methods.createSavedFilter('u1', { name: 'mine', query: {} });
    await expect(methods.createSavedFilter('u1', { name: 'mine', query: {} })).rejects.toThrow();
  });
  test('allows the same name for different users', async () => { /* … */ });
});
```

Run: `cd packages/data-schemas && npx jest src/methods/savedFilter.spec.ts`, then `npx tsc --noEmit`.

---

## 4. Adding a database migration

**There is no migration framework** — no `migrate-mongo`, no versions collection (Verified, E4 §6).
Three mechanisms exist; pick by the kind of change.

### The mechanisms (all Verified)

| Mechanism | Where | Runs | Existing examples |
|---|---|---|---|
| **A. Packaged migration + operator CLI** | Logic in `packages/data-schemas/src/migrations/<name>.ts`, exported from `migrations/index.ts`; a runner in `config/migrate-<name>.js`; npm scripts `migrate:<name>` and `migrate:<name>:dry-run` in the root `package.json` | **Manually**, by an operator, with replicas stopped where indexes change | `migrateTenantIndexes` / `migrate:tenant-indexes`, `dropSupersededPromptGroupIndexes`, `createMCPAuthorityLookupIndexes`, `backfillMCPServerNormalizedNames` |
| **B. Idempotent block inside a startup seed method** | e.g. `initializeRoles()` in `packages/data-schemas/src/methods/role.ts`, run on every start via `seedDatabase()` (`api/models/index.js`) | **Every startup**, under `runAsSystem` | Renaming `permissions.{PROMPTS,AGENTS}.SHARED_GLOBAL` → `.SHARE` via the raw driver, a no-op once migrated |
| **C. Startup detection + warning** | `api/server/services/start/migration.js` → `checkAgentPermissionsMigration` / `checkPromptPermissionsMigration` from `@librechat/api` | After listen, **logs** a warning telling the operator to run the CLI; never migrates | Agent / prompt permission migrations |

The runner shape (verbatim, `config/migrate-tenant-indexes.js`):

```js
require('dotenv').config();
process.env.MONGO_AUTO_INDEX = 'false';
process.env.MONGO_AUTO_CREATE = 'false';
const mongoose = require('mongoose');
const { migrateTenantIndexes } = require('@librechat/data-schemas');
const connect = require('./connect');

(async () => {
  try {
    await connect();
    const result = await migrateTenantIndexes(mongoose.connection, {
      dryRun: process.argv.includes('--dry-run'),
    });
    process.exitCode = result.errors.length > 0 ? 1 : 0;
  } catch (error) {
    console.error('Tenant index migration failed:', error);
    process.exitCode = 1;
  } finally {
    await mongoose.disconnect();
  }
})();
```

### When to use which

| Change | Use | Why |
|---|---|---|
| Rename or reshape a field on a **small, global** collection (roles, categories) | **B** | Cheap enough to check every boot; must be idempotent and guarded so it is a no-op afterwards |
| Drop/replace an index, especially a unique one | **A** | Needs ordering control (auto-index off), may conflict with running replicas; `UPGRADING.md` documents `migrate:tenant-indexes` exactly this way |
| Backfill a derived field across a large or per-user collection | **A**, with `--dry-run` and (if batching) `--batch-size` like `migrate:agent-permissions:batch` | Startup must not scan large collections (AGENTS.md: avoid serial DB reads on startup) |
| Backfill that the app cannot run correctly without | **A + C** | C makes the requirement visible in logs at boot without blocking it |
| New optional field with a schema `default` | Usually **none** | Mongoose applies defaults on read/save of new docs; only backfill if queries filter on it |

### Sequence for mechanism A

1. Write `packages/data-schemas/src/migrations/<name>.ts` taking `(connection, { dryRun })` and
   returning a result with an `errors` array (as `migrateTenantIndexes` does). Make it rerunnable:
   `UPGRADING.md` states the tenant-index command "only builds/drops known index names, never
   deletes documents" and completes safely on rerun.
2. Export it from `migrations/index.ts` (re-exported at the package root).
3. Add `config/migrate-<name>.js` modelled on the runner above.
4. Add `migrate:<name>` and `migrate:<name>:dry-run` to the root `package.json`.
5. Spec it next to the migration (`tenantIndexes.spec.ts`, `mcpServerNames.spec.ts` exist) against
   `MongoMemoryServer`, including an "already migrated" run.
6. Document operator steps in `UPGRADING.md` (replicas stopped, backup, Docker one-off
   `docker compose run --rm --no-deps -w /app api npm run migrate:<name>`) — this is how the
   existing tenant-index migration is documented (Verified, E10 §8).
7. Optionally add a mechanism-C check so boot logs say the migration is pending.

### Sequence for mechanism B

Copy the guard shape in `initializeRoles()`: read the raw document (`Model.collection.findOne`) only
when strict-mode would hide the legacy field, compute `$set`/`$unset`, write only when there is
something to change. Raw driver calls need an `eslint-disable-next-line no-restricted-syntax` with a
justification, as in `role.ts`; raw access bypasses tenant isolation, so use it only on genuinely
global collections.

### Common mistakes

- Expecting a migration to run on deploy. Only B runs automatically.
- Putting a large backfill in B — it then runs on every boot of every replica, and the Lighthouse
  lane (250 ms per query) will catch it if it touches the startup path.
- Forgetting `MONGO_AUTO_INDEX=false` in a runner that changes indexes, letting Mongoose rebuild
  the old index concurrently.

---

## 5. Adding a Redis-backed capability

Redis is **always optional** here (Verified, E5): every use is gated by `USE_REDIS` (and
`USE_REDIS_STREAMS` for the job store, defaulting to `USE_REDIS`) with a working in-memory, file, or
Mongo fallback, and Redis is **never the system of record**. Keep both properties.

### Files to inspect first

- `packages/api/src/cache/cacheConfig.ts` — all `USE_REDIS*` / `REDIS_*` flags,
  `FORCED_IN_MEMORY_CACHE_NAMESPACES`.
- `packages/api/src/cache/cacheFactory.ts` — `standardCache`, `violationCache`, `sessionCache`,
  `limiterCache`.
- `api/cache/getLogStores.js` — the `CacheKeys` → cache-instance registry (legacy wiring).
- `packages/api/src/middleware/concurrency.ts` — atomic Lua counter with in-memory fallback.
- `packages/api/src/flow/manager.ts` — Lua compare-and-set over JSON blobs.
- `packages/api/src/cluster/LeaderElection.ts` — `SET NX EX` election; `isLeader()` is `true`
  without Redis.
- `packages/api/src/stream/createStreamServices.ts` — interface-swap factory with fallback.

### Choose the template

| Need | Template | Fallback behavior |
|---|---|---|
| TTL'd cache over Mongo or an upstream | `standardCache(namespace, ttl, fallbackStore?, { throwOnErrors? })` + a `CacheKeys` entry + registration in `getLogStores.js` | In-memory `Keyv` per namespace (swept every 30 s when Redis is off) |
| Cache whose failure is a correctness bug | `standardCache(..., { throwOnErrors: true })` (as `AUTH_USER_DOC`) | Errors propagate instead of reading as a miss |
| Atomic counter / limit | `concurrency.ts`: `CHECK_AND_INCREMENT_SCRIPT` (`INCR`, `EXPIRE`, `DECR` back if over limit) + `DECREMENT_SCRIPT` | Same logic on `standardCache(CacheKeys.PENDING_REQ)` |
| Atomic state transition on a JSON value | `flow/manager.ts` Lua scripts (`CLAIM_FLOW`, `GUARDED_COMPLETE_FLOW`, …): decode, check a guard (`createdAt` + state), mutate, re-`SET … PX` | Same `Keyv` API in memory |
| Rate limiting | `limiterCache(prefix)` with `express-rate-limit` | Returns `undefined` → in-memory limiter |
| Cross-replica coordination of a subsystem | An interface with two implementations, chosen in a factory (`IJobStore` → `InMemoryJobStore`/`RedisJobStore`) | Factory catches construction failure and falls back |

Decision rule: **do not cache authorization data that cannot fail closed.**
`CacheKeys.PROMPT_GROUPS_ACCESS` is deliberately `disabledCache` with a comment that "a failed shared
invalidation cannot fail closed" (Verified).

### Sequence (simple cache)

1. Add the namespace to `CacheKeys` in `packages/data-provider` and rebuild it.
2. Register it in `api/cache/getLogStores.js` with an explicit TTL (existing TTLs range 30 s–30 min).
3. In `packages/api`, receive the cache accessor as a dependency (data-schemas does this via
   `CreateMethodsDeps.getCache`); do not import `getLogStores` into TS modules.
4. Invalidate on every write path that changes the underlying data.

### Sequence (atomic operation)

1. Write the Lua script as a constant next to its caller, documenting atomicity in a comment (as
   `concurrency.ts` does: "Single round-trip, fully atomic").
2. Build keys from `CacheKeys` + ids. `ioredisClient` already applies `REDIS_KEY_PREFIX`; pub/sub
   channel names are **not** prefixed (gotcha documented in `RedisEventTransport.ts`).
3. On Redis Cluster, keep keys that one script touches in one hash slot with `{hash-tags}` (as
   `stream:{streamId}:*`); global sets on other slots make the guard best-effort (documented in
   `IJobStore.ts`).
4. Make TTL extension extend-only (`if TTL(key) < target then EXPIRE`) as `RedisJobStore` does.
5. Implement the same semantics for the in-memory branch.

### Config lever rule

AGENTS.md: "a limit, timeout, toggle or capability introduced in code earns a field on
`configSchema` … with a default that reproduces today's behavior." For a Redis-backed capability that
means: the **lever** (limit, TTL, window, on/off) goes into `configSchema`, with a default equal to
what the no-Redis/no-feature path does today; `USE_REDIS` keeps selecting the **backend**, exactly as
it does for every existing use case. Existing counter-examples to not copy: `CONCURRENT_MESSAGE_MAX`
/ `LIMIT_CONCURRENT_MESSAGES` and `REDIS_*` TTL knobs are env-only (Verified, `.env.example`) — they
predate the rule. A new env-only switch "needs a reason" (AGENTS.md).

### Tests

- Unit-test both branches with the in-memory path.
- Real Redis: add `*.cache_integration.spec.ts` under `packages/api/src/{cache,middleware}/` (picked
  up by `npm run test:cache-integration:core`), `src/cluster/` (`:cluster`), `src/mcp/` (`:mcp`), or
  `*.stream_integration.spec.ts` (`:stream`). These are excluded from the default `test:ci` and run
  in CI by `cache-integration-tests.yml` (Verified). Local Redis single/cluster/TLS tooling lives in
  `redis-config/`.
- E2E with Redis streams: `npm run e2e:mock:redis`.

### Common mistakes

- Making Redis the only copy of something durable. Every existing key has a TTL of minutes to 24 h;
  final data goes to Mongo (`saveMessage` after a generation, Verified).
- Assuming a Redis-less multi-replica deployment shares state — it does not; the schedule engine
  refuses to arm in that topology unless `SCHEDULES_SINGLE_PROCESS=true` (Verified, E6).
- Putting config-derived values in a Redis cache: `CONFIG_STORE`/`APP_CONFIG` are forced in-memory
  by default for blue/green safety.

---

## 6. Adding a frontend page

### Files to inspect first

- `client/src/routes/index.tsx` — `createBrowserRouter([...])`; lazy loaders at the top of the file.
- `client/src/lib/assets/lazy` — `importWithRecovery` (stale-chunk recovery after deploys).
- `client/src/routes/RouteErrorBoundary.tsx` — `errorElement` on every top-level branch.
- `client/src/routes/Root.tsx` — authenticated shell; `RootLayout` returns `null` when
  unauthenticated.
- `client/src/routes/useAuthRedirect.ts` — component-side login redirect.
- `client/src/routes/Dashboard.tsx` — admin/settings routes, spliced in as `dashboardRoutes`.

### Pattern to follow (Verified — the `insights` route)

```tsx
// verbatim — client/src/routes/index.tsx
const loadInsightsView = () =>
  importWithRecovery(() => import('~/components/Insights')).then((m) => ({
    Component: m.default,
  }));
// … under Root's children:
{
  path: 'insights',
  lazy: loadInsightsView,
},
```

`prompts/:promptId`, `skills/*`, `projects` and `projects/:projectId` use the same shape (Verified).

### Sequence

1. Create `client/src/components/<Feature>/index.tsx` with a default export.
2. Add a `load<Feature>View` loader using `importWithRecovery`.
3. Register `{ path: '<feature>', lazy: load<Feature>View }` under `Root`'s children (authenticated),
   under `StartupLayout`/`LoginLayout` (pre-auth), or in `Dashboard.tsx` (admin).
4. Add navigation (sidebar: `client/src/components/UnifiedSidebar/`; header menus must use
   `DropdownPopup` + `Ariakit.MenuButton`, AGENTS.md "Frontend rules").
5. Add every visible string to `client/src/locales/en/translation.json` only (English keys; other
   locales sync via Locize).
6. Data via React Query hooks ([§8](#8-connecting-a-ui-action-to-a-backend-operation)); loading,
   empty, error states ([§9](#9-form-validation-loading-and-error-states)).

### Authorization

There is no router-level guard. Inside `Root`, the shell is already gated; pages that depend on a
role permission or interface toggle must check it themselves (from the startup config or user
query) **and** the backend must enforce it — UI hiding is not authorization.

### Common mistakes

- `import()` without `importWithRecovery` — a deploy then breaks open tabs with chunk-load errors
  instead of recovering.
- Wrapping the route change in a React transition: `App.jsx` deliberately passes
  `useTransitions={false}` to `RouterProvider` (load-bearing comment, Verified).
- Shipping a backend capability with no frontend entry point (AGENTS.md "Review and completion").

### Tests

Component tests in the page's `__tests__/` using `client/test/layout-test-utils.tsx` where layout
matters; Playwright specs under `e2e/` for navigation-level behavior (mock backend:
`npm run e2e:mock`).

---

## 7. Adding a frontend component

### Where it goes (Verified, E7 §10)

| Put it in | When | Example |
|---|---|---|
| `packages/client/src/components/` | Generic, app-agnostic primitive with no domain knowledge | `Button`, `Dialog`, `DropdownPopup`, `Skeleton`, `DataTable`, `Combobox` |
| `client/src/components/<Feature>/` | Knows about conversations, agents, messages, settings, … — even if it looks generic | `SidePanel/Agents/AgentPanelSkeleton.tsx` composes `Skeleton` but is agent-panel-shaped |

If the design system cannot express a reusable need, **deepen the shared primitive or the token
registry** (`packages/client/src/theme/tokens.css`) rather than copying classes into a feature
(AGENTS.md "Frontend theming and styling"). Then `npm run build:client-package`.

### Rules that reviewers check (AGENTS.md)

- Semantic Tailwind roles only (`text-text-secondary`, `bg-surface-primary`, …); no raw palette
  utilities or hard-coded colors — `tokens.css` is lint-enforced.
- Light/dark (class-based `.dark`) and `prefers-reduced-motion` (`useMediaQuery('(prefers-reduced-motion: reduce)')`
  from `@librechat/client`, as `Root.tsx` does).
- `useLocalize()` for every string; semantic HTML, keyboard behavior, ARIA labels
  (`StatefulWorkspaceDefault.tsx` wires `aria-labelledby`/`aria-describedby`, Verified).
- Action menus: `DropdownPopup` + `Ariakit.MenuButton` (`Chat/Menus/HeaderMenu.tsx`). Never the
  Radix `DropdownMenu` family for action or sort menus. For a menu item that opens a dialog keep the
  Share/Export contract: `hideOnClick: false`, item ref, button render, dialog `triggerRef`.

### State (AGENTS.md "Client state ownership")

- **New state is Jotai**, even in a file importing Recoil. Persisted atoms use
  `client/src/store/jotai-utils.ts` (`createStorageAtom`, `createStorageAtomWithEffect`,
  `createTabIsolatedAtom`, `initializeFromStorage`). There is no Jotai `<Provider>`; atoms use the
  default store (Verified).
- Feature-owned state stays inside the feature (e.g. `components/Chat/Subagents/state.ts`).
- App-global shell state (`enterToSend`, `maximizeChatSpace`, `showScrollButton`, artifact
  visibility) is **passed in by props or a small context**, not read from `~/store` inside your
  component. Verified example: `ChatView.tsx` reads `enterToSend` once and passes it down;
  `SubagentThreadPanel.tsx` receives it as a prop.
- Converting existing state: one atom plus every file that reads or writes it, in one change.

### Tests

Colocated `__tests__/<Component>.spec.tsx` covering loading, success and failure (AGENTS.md
"Frontend rules"). `packages/client` excludes spec files from `tsc`, so type errors in tests there
surface only at Jest time.

---

## 8. Connecting a UI action to a backend operation

### The chain (Verified, E7 §5)

```
Component
  → client/src/data-provider/**/*.ts           React Query hook (useQuery / useMutation / useInfiniteQuery)
    → dataService.<fn>()                       packages/data-provider/src/data-service.ts
      → endpoints.<fn>()                       packages/data-provider/src/api-endpoints.ts (URL builder)
      → request.<verb>()                       packages/data-provider/src/request.ts
        → axios (proactive refresh + one 401 refresh-and-retry; 2FA-setup redirect on 403)
```

Keys live in `packages/data-provider/src/keys.ts` (`QueryKeys`, `DynamicQueryKeys`,
`MutationKeys`) — never inline strings (AGENTS.md). Encode dynamic URL parameters in the URL
builder.

### Real worked trace — renaming a conversation (Verified, verbatim citations)

| Step | Code |
|---|---|
| UI | `client/src/components/Chat/Rename.tsx` `handleSubmit`: trims, early-returns if unchanged, `await updateMutation.mutateAsync({ conversationId, title: next })` |
| Hook | `client/src/data-provider/mutations.ts` `useUpdateConversationMutation(id)` → `(payload) => dataService.updateConversation(payload)` |
| Service | `packages/data-provider/src/data-service.ts`: `return request.post(endpoints.updateConversation(), { arg: payload });` |
| URL | `packages/data-provider/src/api-endpoints.ts`: ``export const updateConversation = () => `${conversationsRoot}/update`;`` |
| Route | `api/server/routes/convos.js`: `router.post('/update', validateConvoAccess, configMiddleware, createRenameConversationHandler({ saveConvo: db.saveConvo, getConvo: db.getConvo, getActiveRunIds: GenerationJobManager.getCleanupBlockingJobIdsForConversations.bind(GenerationJobManager), logger }))` |
| Handler | `packages/api/src/conversations/rename.ts`: 400 on bad input, 401, `404 conversation_not_found`, `409` title-ownership guard while a run is active (unless `interface.runningChatRename`), content-filter inspection, then `deps.saveConvo(..., { titleSource: 'manual' })` |
| DB | `packages/data-schemas/src/methods/conversation.ts` `saveConvo` (strips server-owned fields; `titleSource: 'manual'` sets `titleSetByUser`) |
| Cache | Hook `onSuccess`: `cancelQueries` for the single convo, `markTitleGenerationProcessed`, field-level `applyRename` merge guarded by `titleRevision`, `setQueryData`, `updateConvoInAllQueries`, then `invalidateQueries` for paginated lists (cursor encodes old ordering) |
| Error | `Rename.tsx` `catch`: `logger.error(...)`, `showToast({ message: localize('com_ui_rename_failed'), severity: NotificationSeverity.ERROR, showIcon: true })`; dialog stays open so the user can retry |

The cache step, verbatim from `client/src/data-provider/mutations.ts`:

```ts
const applyRename = (previous?: t.TConversation): t.TConversation => {
  if (
    previous &&
    (previous.titleRevision ?? 0) > (updatedConvo.titleRevision ?? Infinity)
  ) {
    return previous;
  }
  return {
    ...(previous ?? updatedConvo),
    title: updatedConvo.title,
    titleSetByUser: updatedConvo.titleSetByUser,
    titleRevision: updatedConvo.titleRevision,
    updatedAt: updatedConvo.updatedAt,
  };
};
queryClient.setQueryData<t.TConversation>([QueryKeys.conversation, targetId], applyRename);
updateConvoInAllQueries(queryClient, targetId, applyRename);
/* A title-keyset cursor encodes the old ordering; patching loaded rows
 * cannot repair boundaries that have not been fetched yet. */
queryClient.invalidateQueries({ queryKey: [QueryKeys.allConversations] });
queryClient.invalidateQueries({ queryKey: [QueryKeys.archivedConversations] });
queryClient.invalidateQueries([QueryKeys.projectConversations]);
```

Lessons to carry over: patch only the fields the response owns (a whole-object write can revert a
concurrent change); guard against out-of-order responses with a revision field; patch what is
rendered and invalidate what is paginated.

The simpler variant, `useUpdateUserPreferencesMutation` in `client/src/data-provider/Auth/mutations.ts`
(Verified), shows the hook-options convention: it is keyed `[MutationKeys.updateUserPreferences]`,
spreads the caller's `options`, merges `data.preferences` into `[QueryKeys.user]` with
`setQueryData`, then calls `options?.onSuccess?.(data, ...args)` so the component can toast.

### Sequence

1. Types in `packages/data-provider/src/types.ts` (`T<Thing>Request`/`T<Thing>Response`).
2. URL builder in `api-endpoints.ts`.
3. Thin function in `data-service.ts` calling `request.get/post/patch/delete`.
4. Key in `keys.ts` (`QueryKeys` for reads, `MutationKeys` when hook instances must observe each
   other's writes — the comment on `updateFavorites`/`updatePinnedOrder` explains why).
5. `npm run build:data-provider` (type-checks).
6. Hook in `client/src/data-provider/` (or the matching subfolder: `Auth/`, `Messages/`, `SSE/`),
   exported via the folder's barrel so components import from `~/data-provider`.
7. Component calls the hook; maps success/failure to UI ([§9](#9-form-validation-loading-and-error-states)).
8. Backend endpoint per [§1](#1-adding-a-new-backend-endpoint).

### End-to-end example C — UI for saved filters (*Illustrative, built from real existing patterns*)

> *Illustrative.* Completes examples A and B. Modelled on the preferences and rename chains above.

```ts
// Illustrative — packages/data-provider/src/types.ts
export type TSavedFilter = { id: string; name: string; query: Record<string, string> };
export type TSavedFiltersResponse = { filters: TSavedFilter[] };
export type TCreateSavedFilterRequest = { name: string; query: Record<string, string> };

// Illustrative — packages/data-provider/src/api-endpoints.ts
// (same convention as the real `export const user = () => `${BASE_URL}/api/user`;`)
export const savedFilters = () => `${BASE_URL}/api/saved-filters`;

// Illustrative — packages/data-provider/src/data-service.ts
export function getSavedFilters(): Promise<t.TSavedFiltersResponse> {
  return request.get(endpoints.savedFilters());
}
export function createSavedFilter(payload: t.TCreateSavedFilterRequest): Promise<t.TSavedFilter> {
  return request.post(endpoints.savedFilters(), payload);
}

// Illustrative — packages/data-provider/src/keys.ts
//   QueryKeys:    savedFilters = 'savedFilters',
//   MutationKeys: createSavedFilter = 'createSavedFilter',
```

```ts
// Illustrative — client/src/data-provider/SavedFilters/queries.ts (+ index.ts barrel)
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { QueryKeys, MutationKeys, dataService } from 'librechat-data-provider';
import type { UseMutationOptions } from '@tanstack/react-query';
import type * as t from 'librechat-data-provider';

export const useSavedFiltersQuery = (enabled: boolean) =>
  useQuery<t.TSavedFiltersResponse>([QueryKeys.savedFilters], () => dataService.getSavedFilters(), {
    enabled,
  });

export const useCreateSavedFilterMutation = (
  options?: UseMutationOptions<t.TSavedFilter, Error, t.TCreateSavedFilterRequest>,
) => {
  const queryClient = useQueryClient();
  return useMutation<t.TSavedFilter, Error, t.TCreateSavedFilterRequest>(
    [MutationKeys.createSavedFilter],
    (payload) => dataService.createSavedFilter(payload),
    {
      ...options,
      onSuccess: (created, ...args) => {
        queryClient.setQueryData<t.TSavedFiltersResponse>([QueryKeys.savedFilters], (prev) =>
          prev ? { filters: [created, ...prev.filters] } : prev,
        );
        options?.onSuccess?.(created, ...args);
      },
    },
  );
};
```

The import sources match the real hooks (`client/src/data-provider/Auth/mutations.ts` imports
`useMutation`/`useQueryClient` from `@tanstack/react-query` and `MutationKeys`, `QueryKeys`,
`dataService` from `librechat-data-provider`). Run
`npm run sort-imports -- client/src/data-provider/SavedFilters` afterwards.

```tsx
// Illustrative — client/src/components/Conversations/SavedFilters/SaveFilterButton.tsx
// Modelled on StatefulWorkspaceDefault.tsx (toasts, disabled while pending) and the
// HeaderMenu DropdownPopup pattern if it lives in a menu.
export default function SaveFilterButton({ query }: { query: Record<string, string> }) {
  const localize = useLocalize();
  const { showToast } = useToastContext();
  const mutation = useCreateSavedFilterMutation({
    onSuccess: () => showToast({ message: localize('com_ui_saved_filter_created'), status: 'success' }),
    onError: () => showToast({ message: localize('com_ui_saved_filter_failed'), status: 'error' }),
  });
  // Open a dialog with a name field (see §9 for the react-hook-form rules), then:
  // mutation.mutate({ name, query });
}
```

Backend error codes (`saved_filter_limit_reached`, `saved_filters_disabled`) should map to distinct
localized strings in the component, never to the raw response text. Gate the entry point on the
startup/interface config so the button is hidden when `savedFilterLimit` is `0`, while the 403 on
the server remains the real enforcement.

Tests: `__tests__/SaveFilterButton.spec.tsx` — pending (button disabled), success (toast + cache
updated), failure (error toast, input kept), modelled on `StatefulWorkspaceDefault.spec.tsx` and
`Chat/__tests__/Rename.spec.tsx`. Run `cd client && npx jest src/components/Conversations/SavedFilters`.

### Common mistakes

- Inline query-key arrays (`['savedFilters']`) instead of `keys.ts`.
- Invalidating everything after a mutation — patch what is rendered, invalidate only what cannot be
  patched correctly (rename's paginated lists).
- Forgetting `npm run build:data-provider` — `client` Jest maps `librechat-data-provider/react-query`
  to source, but other imports resolve to the build (Verified, `client/jest.config.cjs`).
- Calling `axios`/`fetch` directly from a component — it bypasses the token-refresh interceptors in
  `request.ts`.

---

## 9. Form validation, loading and error states

### Forms (Verified, E7 §7)

The repo uses **`react-hook-form` with inline `register()` rules and no schema resolver**. No
`zodResolver`/`yupResolver` is wired into any `useForm` in `client/src/components`, even though
`zod` is a dependency (used for backend/config schemas). Follow that pattern; do not introduce a
resolver.

```tsx
// verbatim excerpt — client/src/components/Auth/LoginForm.tsx
{...register('email', {
  required: localize('com_auth_email_required'),
  maxLength: { value: 120, message: localize('com_auth_email_max_length') },
  validate: useUsernameLogin
    ? undefined
    : (value) => validateEmail(value, localize('com_auth_email_pattern')),
})}
aria-invalid={!!errors.email}
```

- Messages are localized keys; `aria-invalid` mirrors the error; a `renderError(field)` helper
  renders `errors[field]?.message` in a `role="alert"` element.
- Server-derived limits come from startup config (`minLength: startupConfig?.minPasswordLength || 8`).
- Multi-panel forms: `useForm` + `FormProvider` + `useWatch` (`SidePanel/Agents/AgentPanel.tsx`),
  validation distributed across sub-panels.
- Simple dialogs may use local `useState` instead (`Chat/Rename.tsx`).
- **Client validation is UX only**; the handler re-validates ([§1](#1-adding-a-new-backend-endpoint)).

### Loading

| Situation | Pattern (Verified) |
|---|---|
| Panel/list with known layout | Feature-shaped skeleton composing `Skeleton` from `@librechat/client` — `SidePanel/Agents/AgentPanelSkeleton.tsx` |
| Whole route waiting on queries | `Spinner` with `role="status"` and `aria-live="polite"` — `routes/ChatRoute.tsx` |
| Mutation in flight | Disable the control (`disabled={mutation.isLoading}` in `StatefulWorkspaceDefault.tsx`) |

### Errors

| Surface | Pattern (Verified) |
|---|---|
| In-chat message error codes | `client/src/components/Messages/Content/Error/registry.ts`: `errorCopy` (code → one translation key) or `errorRenderers` (code → component, e.g. `BalanceError`, `ProviderError`). Add the backend code here |
| Non-chat query failure | Inline branch with a retry `Button` (`ChatRoute.tsx` on `initialConvoQuery.isError && !initialConvoQuery.isFetching`) |
| Mutation failure | `useToastContext().showToast(...)` with a localized message; keep user input (Rename) or roll back optimistic local state (StatefulWorkspaceDefault) |
| Route crash / chunk failure | `RouteErrorBoundary` (already attached) |
| 401 on any query | Handled globally (`QueryCache.onError` → `ApiErrorBoundaryContext`; axios refresh-and-retry) — do not handle per component |
| Trace/debug detail | `client/src/components/Chat/Trace/Viewer.tsx` |

Backend side of the same contract: emit a stable, safe code; the UI maps it to localized copy with a
fallback (AGENTS.md). Never render `error.message` from a response.

### Empty / success

No dedicated component convention: render the real content from `data`, and an explicit empty
state when the list is empty (AGENTS.md lists "empty" as a required observable state).

---

## 10. Adding background processing

Read [09 Background processing](./09-background-processing.md) first. Summary of what exists
(Verified, E6): **no** `node-cron`, BullMQ, Agenda or worker process. Background work is
Mongo-lease polling plus one general-purpose durable queue.

| Mechanism | Files | Use it for |
|---|---|---|
| **Agent Trigger Delivery** | `packages/api/src/agents/triggers/{service,host,envelope,dispatch}.ts`, `README.md`; collection `AgentTriggerDelivery` | **The extension point** for new asynchronous agent work. Its README: *"A schedule, webhook, queue consumer, MCP integration, or internal event adapter produces the same versioned envelope and calls `enqueueAgentTrigger`; the adapter does not invoke an agent runtime directly."* At-least-once, idempotent by `deliveryKey`, bounded backoff, dead letters, 90-day TTL on success |
| Schedule engine | `packages/api/src/schedules/{engine,fire,service,cadence}.ts` | A *producer* into trigger delivery; example of a 30 s polling loop with Mongo leases, misfire grace, reconcile, pre-drain shutdown |
| Subagent completion wakeup | `packages/api/src/agents/subagentCompletionWakeup.ts` | Second producer: `event.source = { type: 'internal', id: 'subagent-completion' }` |
| Resumable generation jobs | `packages/api/src/stream/GenerationJobManager.ts` | Live generation state, not a general queue |
| Shutdown tasks | `packages/api/src/app/shutdown.ts` | Register `pre-drain`/`post-drain` tasks; never add your own `process.on('SIGTERM')` |

### Writing a new trigger adapter (follow the README's adapter contract, Verified)

1. Authenticate and authorize the source **before** building an envelope.
2. Strip credentials and transport secrets from `event.payload`.
3. Give each source event a stable `event.id`; keep `deliveryId` stable across retries.
4. Use `continue` mode only with a persisted `conversationId` and exact `parentMessageId`; external
   sources address an event binding by opaque id, never a raw child conversation/agent id.
5. Use `orderingKey` only when ordering must span sources.
6. Call `enqueueAgentTrigger`. Execution then arrives through the self-loopback `POST
   /api/agents/chat` with `req._isAgentTrigger` set — the same controller as a human message.
7. Remote/external sources already have an ingress: `POST /api/agents/v1/events` (API key +
   `Idempotency-Key`, `202` with a poll location).

### Non-agent periodic work

Follow the schedule engine's shape: a `setTimeout` tick with jitter, a Mongo lease/claim (unique or
partial-unique index rather than a count), per-row isolated `try/catch`, a `pre-drain` shutdown
task that awaits the in-flight tick, and a topology check (Redis-backed job store or explicit
single-process assertion) before arming in multi-replica deployments. Leader-only work can gate on
`LeaderElection.isLeader()` (always `true` without Redis). Expose interval/limits via `configSchema`
(polling intervals already live under `endpoints.agents.eventDriven.idlePolling`; schedules under
`interface.schedules`, default disabled).

### Common mistakes

- Introducing a queue library — the architecture deliberately uses Mongo CAS/leases.
- Calling the agent runtime directly from an adapter instead of enqueueing.
- In-process timers that assume one replica.

---

## 11. Writing unit and integration tests

See [11 Testing & debugging](./11-testing-debugging.md) for configs, CI workflows and debugging.

### Mocking policy (AGENTS.md, Verified practice)

- **Real database**: `mongodb-memory-server`. `packages/data-schemas/jest.globalSetup.mjs`
  pre-downloads the binary once so workers do not race (`ENOENT … tgz.downloading`).
- **Real MCP SDK** for MCP behavior (Inferred from the Jest ESM allow-list; AGENTS.md states it).
- **Mock only external HTTP / uncontrollable services**: `jest.mock('axios', …)` in
  `packages/api/src/utils/axios.spec.ts`; better, inject the transport, as
  `packages/api/src/skills/sync/github.spec.ts` injects `fetchFn` (including `TypeError('fetch failed')`).
- Prefer injected `jest.fn()` dependencies over `jest.mock` of internal modules — the DI factories
  make this possible (`preferences.spec.ts`).

### Templates by layer

| Layer | Template | Run |
|---|---|---|
| data-schemas method | `packages/data-schemas/src/methods/user.methods.spec.ts` (`MongoMemoryServer.create()` in `beforeAll`, `dropDatabase()` in `beforeEach`, real factories) | `cd packages/data-schemas && npx jest src/methods/<file>.spec.ts` |
| data-schemas migration | `packages/data-schemas/src/migrations/tenantIndexes.spec.ts` | same workspace |
| packages/api handler | `packages/api/src/user/preferences.spec.ts` (hand-built `req`/`res`, injected deps) | `cd packages/api && npx jest src/user/preferences.spec.ts` |
| packages/api with DB | `packages/api/src/schedules/service.spec.ts`, `agents/transactions.spec.ts` (`MongoMemoryServer`) | same |
| Redis / stream integration | `*.cache_integration.spec.ts`, `*.stream_integration.spec.ts` | `npm run test:cache-integration:core` (or `:cluster`, `:mcp`, `:stream`) in `packages/api` |
| Wider integration | `*.integration.spec.ts` | `cd packages/api && npm run test:integration` |
| api route wiring | `api/server/routes/presets.test.js` (supertest + `jest.mock('~/models')` + stubbed `requireJwtAuth`/`configMiddleware`) | `cd api && npx jest server/routes/presets.test.js` |
| Client component | `client/src/components/Nav/Settings/__tests__/StatefulWorkspaceDefault.spec.tsx`, `Chat/__tests__/Rename.spec.tsx`; helpers in `client/test/` (`layout-test-utils.tsx`, `itemFactories.ts`, `harness.tsx`) | `cd client && npx jest <path>` |
| Shared UI primitive | `packages/client` Jest (jsdom) | `cd packages/client && npx jest <path>` |
| data-provider | `packages/data-provider` Jest | `cd packages/data-provider && npx jest <path>` |
| Config migrations | `config/jest.config.js` | `npm run test:config` |
| E2E | `e2e/playwright.config.mock.ts` | `npm run e2e:mock` |
| Performance gate | `e2e/lighthouse/` | `npm run lighthouse` |

The route-wiring template mocks `~/models` and the middleware barrel (verbatim, `presets.test.js`):

```js
jest.mock('~/models', () => ({
  getPresets: mockGetPresets,
  savePreset: mockSavePreset,
  deletePresets: mockDeletePresets,
}));

jest.mock('~/server/middleware', () => ({
  requireJwtAuth: (req, _res, next) => {
    req.user = { id: 'user-1' };
    next();
  },
  configMiddleware: (req, _res, next) => {
    req.config = mockAppConfig;
    next();
  },
}));
```

That is acceptable for testing **wiring**; behavior belongs in the factory and data-schemas specs
where real logic and a real database run. Likewise, the client component specs mock `~/hooks` and
`@librechat/client` (Verified in `StatefulWorkspaceDefault.spec.tsx`) — keep those mocks at the
hook boundary.

### What to cover (AGENTS.md)

- Absence **versus** failure (`null` vs thrown), the boundary's non-success response, and that
  sensitive diagnostics cannot reach the client (assert the 500 body is the fixed message).
- Loading, success and failure for components.
- Tenant isolation for new entities (`*.tenant.spec.ts` siblings exist).
- Auth user-doc cache invalidation for user mutations (`user.methods.spec.ts` asserts the cache
  `delete` calls).

### Gotchas

- `packages/api`'s default `test`/`test:ci` ignore `*.integration.*`, `*.helper.*`,
  `__tests__/helpers/` and `*.manual.spec.*` — name integration specs accordingly or they will run
  (and fail) in the unit lane.
- `packages/client` excludes specs from `tsc`.
- `api/test/jestSetup.js` sets dummy `MONGO_URI`, `JWT_SECRET`, `CREDS_KEY`/`CREDS_IV`; the logger is
  mocked globally — don't assert on real log transports.

---

## 12. Updating API and developer documentation

- This suite lives in `docs/developer-guide/`. Update the page that owns the subsystem you changed
  (see [§13](#13-documentation-maintenance-checklist)); keep `Verified`/`Inferred`/`Unknown` labels
  honest — re-verify a claim you touch.
- Feature-specific reference docs already sit in `docs/` (`mcp-apps.md`, `tool-approval-modes.md`,
  `skills-management-api.md`, `workspace-checkouts.md`, `run_files.md`, `permissions/`). Extend the
  matching one rather than creating a parallel page.
- Config fields: document in `librechat.example.yaml` next to the schema field you added.
  Environment variables: `.env.example` (but prefer `configSchema`, AGENTS.md).
- Upgrade steps / operator commands: `UPGRADING.md`.
- Domain vocabulary: `CONTEXT.md`.
- OpenAPI: `packages/api` generates `openapi/agents.openapi.json` from `src/openapi/generate.ts`
  (`openapi:generate` writes, `openapi:check` verifies, `openapi:test` smoke-tests the router); CI
  runs check/test in `backend-review.yml` (Verified). The spec covers the **agents API** surface, so
  regenerate it (`npm run -w @librechat/api openapi:generate`) when you change those routes; an
  ordinary app route such as example A is not part of it (Inferred from the file name and scope).
- PR description: follow `.github/pull_request_template.md` and AGENTS.md (what breaks, what triggers
  it, behavior after, one focused view of the mechanism).
- Before pushing doc-adjacent code changes, AGENTS.md's hygiene still applies:
  `npm run sort-imports -- <touched paths>` and `npx tsc --noEmit` in each changed workspace.

---

## 13. Documentation maintenance checklist

Use this when a change lands in a subsystem. It extends AGENTS.md's spirit (describe the code as it
stands) without adding any new tooling.

| If you changed… | Update |
|---|---|
| A route, its auth/limiter chain, or a status/code it returns | [02-backend.md](./02-backend.md); API reference page if present; `packages/api/openapi/agents.openapi.json` via `openapi:generate` for agents-API routes |
| `api/server/index.js` startup order or mounts | [02-backend.md](./02-backend.md), [00-overview.md](./00-overview.md); run `npm run lighthouse` |
| A schema, index, TTL or method contract in `packages/data-schemas` | [05-database.md](./05-database.md) (entity table, indexes) |
| A migration (any mechanism in [§4](#4-adding-a-database-migration)) | [05-database.md](./05-database.md), `UPGRADING.md`, root `package.json` scripts list in the docs |
| A business rule or its enforcement point | [06-business-logic.md](./06-business-logic.md); `CONTEXT.md` if a term changed |
| A Redis key, TTL, script or fallback | The Redis section of the suite (key table) and [09-background-processing.md](./09-background-processing.md) if streams/leases changed |
| Trigger delivery, schedules, subagents, shutdown tasks | [09-background-processing.md](./09-background-processing.md); `packages/api/src/agents/triggers/README.md` |
| `configSchema` field | `librechat.example.yaml`; any doc that lists the lever; [00-overview.md](./00-overview.md) if it is deployment-relevant |
| Env var | `.env.example` |
| Routes, layouts, providers, state ownership, theming tokens | [03-frontend.md](./03-frontend.md) |
| Test scripts, Jest configs, CI workflows, Lighthouse budgets | [11-testing-debugging.md](./11-testing-debugging.md); `e2e/lighthouse/README.md` |
| A pattern this guide recommends (new template, a legacy file fixed) | This file — e.g. if `convos.js` deletion orchestration moves to `packages/api`, update the [§1](#1-adding-a-new-backend-endpoint) "Common mistakes" note |
| Contributor rules themselves | `AGENTS.md` (maintainer decision), then align this guide |

---

## 14. Pre-PR checklist

Condensed from AGENTS.md; every item maps to a section above.

- [ ] Branched from `dev`; `gh pr create --base dev` (or `--base canary` only when the maintainer
      chose canary). Link issues with `Related to #N`; close them by hand after merge.
- [ ] New backend behavior is in `packages/api` / `packages/data-schemas`; `api/` gained only
      wiring ([§1](#1-adding-a-new-backend-endpoint)).
- [ ] No Mongoose types in exported `packages/api` / `data-provider` / `client` signatures.
- [ ] New levers are `configSchema` fields with today's behavior as default (all touchpoints —
      for `interface`, also `loadDefaultInterface`).
- [ ] Failures: plain value / documented `null` / typed result / throw; safe codes at the
      boundary; no `error.message` to clients.
- [ ] User mutations invalidate the auth user-doc cache.
- [ ] Frontend: Jotai for new state, shell state passed in, `useLocalize` + English keys only,
      semantic tokens, a11y, `DropdownPopup` menus, keys in `keys.ts`, related queries
      invalidated/patched.
- [ ] Loading, empty, success, failure, cancellation/retry states shipped and tested.
- [ ] Focused tests in each owning workspace; `npx tsc --noEmit` in each changed workspace;
      `npm run sort-imports -- <paths>`; `npm run static-checks -- --against origin/dev`.
- [ ] `npm run lighthouse` if startup/auth/config/file/message loading changed.
- [ ] Docs updated per [§13](#13-documentation-maintenance-checklist).
- [ ] Report: pushed head, what ran locally, CI state, review result **at that head**, rejected
      findings with reasons, checks you could not run.

---

### Open questions (Unknown)

- Exactly which routes `src/openapi/generate.ts` covers was not traced; check it before assuming a
  route is outside the agents OpenAPI spec.
- Whether `migrate-agent-permissions.js` and siblings call packaged migrations or implement logic
  inline in `config/` was only confirmed for `migrate-tenant-indexes.js`; prefer the packaged shape
  for new work regardless.
