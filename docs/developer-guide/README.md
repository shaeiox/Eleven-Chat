# Eleven-Chat Developer Guide

A comprehensive, evidence-based documentation suite for developers working in this repository —
a heavily customized fork of [LibreChat](https://github.com/LibreChat-AI/LibreChat). Every claim in
this suite is grounded in the actual implementation, labeled **Verified**, **Inferred**, or
**Unknown**, and cited by file path (and, where stable, line number). Nothing here was invented;
where something couldn't be confirmed, the relevant chapter says so explicitly.

This suite complements, rather than replaces, three existing deep documents at the repository root:

- **[`../../PROJECT_MAP.md`](../../PROJECT_MAP.md)** — the original architecture map.
- **[`../../CRITICAL_FLOWS.md`](../../CRITICAL_FLOWS.md)** — six execution traces through real code
  (message lifecycle, auth, multi-provider LLM abstraction, conversation/context management,
  configuration system, frontend state & data fetching).
- **[`../../CONTEXT.md`](../../CONTEXT.md)** — a dense domain-language glossary of fork-specific
  concepts.

This suite verifies, corrects, and extends those documents — filling gaps they didn't cover (Redis,
full database schema, full API inventory, security review, testing, configuration/deployment,
business-rule catalog, feature-development workflows) — rather than duplicating what they already
do well. Where this suite found a stale claim in those root documents, it says so explicitly (see
[`13-architecture-decisions-and-limitations.md`](./13-architecture-decisions-and-limitations.md) §5).

## Reading order

**New to this repository?** Read in this order:

1. [`00-overview.md`](./00-overview.md) — what this app is, in ten minutes
2. [`01-architecture.md`](./01-architecture.md) — the full system architecture
3. [`../../CRITICAL_FLOWS.md`](../../CRITICAL_FLOWS.md) — how a chat message actually flows (root-level doc)
4. [`02-backend.md`](./02-backend.md) and [`03-frontend.md`](./03-frontend.md) — the two halves of the app
5. [`05-database.md`](./05-database.md) and [`04-redis.md`](./04-redis.md) — where state lives
6. [`06-business-logic.md`](./06-business-logic.md) — the rules that make this app *this* app
7. [`10-feature-development.md`](./10-feature-development.md) — how to actually build something

**Already familiar, need a specific answer?** Jump straight to the relevant chapter below.

## Full index

| # | Document | Covers |
|---|---|---|
| 00 | [Overview](./00-overview.md) | What the app does, tech stack, major modules, quick orientation |
| 01 | [Architecture](./01-architecture.md) | System architecture, module boundaries, diagrams, startup/shutdown |
| 02 | [Backend Internals](./02-backend.md) | Express app, middleware, routing, error handling, execution traces |
| 03 | [Frontend Internals](./03-frontend.md) | React app, routing, state (Recoil/Jotai), API client, forms |
| 04 | [Redis](./04-redis.md) | Every real Redis use case, key patterns, TTLs, fallback behavior |
| 05 | [Database](./05-database.md) | MongoDB/Mongoose, entity inventory, ERD, migrations, lifecycle |
| 06 | [Business Logic](./06-business-logic.md) | Domain rules by feature, with cross-cutting findings |
| 07 | [API Reference](./07-api-reference.md) | Every discovered REST/SSE endpoint, by domain |
| 08 | [Auth & Security](./08-auth-security.md) | Auth flows, protections verified, gaps found |
| 09 | [Background Processing](./09-background-processing.md) | Schedules, triggers, subagents, MCP, the SSE job system |
| 10 | [Feature Development](./10-feature-development.md) | Practical, repo-specific workflows with worked examples |
| 11 | [Testing & Debugging](./11-testing-debugging.md) | Test setup, how to run tests, debugging guidance |
| 12 | [Configuration & Deployment](./12-configuration-deployment.md) | Env vars, Docker, Helm, CI/CD |
| 13 | [Decisions & Limitations](./13-architecture-decisions-and-limitations.md) | Documented decisions, inferred rationale, risk register |
| 14 | [Glossary](./14-glossary.md) | Every domain term, verification status, cross-links |

## Quick start (verified commands)

```bash
# install dependencies (repo root)
npm install

# build the typed packages, in dependency order
npm run build:data-provider
npm run build:data-schemas
npm run build:api
npm run build:client

# run the backend
npm run backend:dev          # development
npm run backend               # production mode

# run the frontend dev server
cd client && npm run dev

# run focused tests from the owning workspace (never the whole monorepo)
cd packages/api && npm run test:ci
cd client && npm run test:ci

# reproduce this PR's static checks
npm run static-checks -- --against origin/dev
```

See [`12-configuration-deployment.md`](./12-configuration-deployment.md) for the full environment
variable reference and [`11-testing-debugging.md`](./11-testing-debugging.md) for every test command
per workspace.

## Most important entry points

| File | Why it matters |
|---|---|
| `api/server/index.js` | Backend entry point and startup sequence |
| `client/src/main.jsx` | Frontend entry point and provider tree |
| `packages/api/src/stream/GenerationJobManager.ts` | The durable, resumable chat-generation job system |
| `packages/data-schemas/src/schema/*.ts` | Every database entity |
| `packages/data-provider/src/config.ts` | The `librechat.yaml` config schema |
| `packages/data-provider/src/api-endpoints.ts` | Canonical frontend-known API URL shapes |
| `AGENTS.md` (repo root) | Contributor and coding-convention guidance — read before changing code |

## A map of which document answers which question

| Question | Document |
|---|---|
| "What does this app do?" | [00](./00-overview.md) |
| "How is this thing structured?" | [01](./01-architecture.md) |
| "Where does this request go after `app.use()`?" | [02](./02-backend.md) |
| "Is this state Recoil or Jotai, and why?" | [03](./03-frontend.md) §6 |
| "Do I need Redis for this?" | [04](./04-redis.md) |
| "What does the `User` document look like?" | [05](./05-database.md) |
| "Why does the app refuse to do X?" | [06](./06-business-logic.md) |
| "What does this endpoint expect/return?" | [07](./07-api-reference.md) |
| "Is this protected? How?" | [08](./08-auth-security.md) |
| "How do scheduled runs/subagents/MCP actually work?" | [09](./09-background-processing.md) |
| "How do I add X, following the existing pattern?" | [10](./10-feature-development.md) |
| "How do I run/debug the tests?" | [11](./11-testing-debugging.md) |
| "What env var controls this, and how do I deploy it?" | [12](./12-configuration-deployment.md) |
| "Is this a known limitation or a bug I just found?" | [13](./13-architecture-decisions-and-limitations.md) |
| "What does this fork-specific term mean?" | [14](./14-glossary.md) |

## Suggested learning path for a new developer

1. Skim [00](./00-overview.md) and [01](./01-architecture.md) — don't memorize, just get the shape.
2. Read [`../../CRITICAL_FLOWS.md`](../../CRITICAL_FLOWS.md) Flow 1 in full — it's the single most
   load-bearing design decision in the codebase (durable, resumable generation jobs).
3. Pick one real feature (e.g. renaming a conversation — traced in [03](./03-frontend.md) §18) and
   follow it through every layer using [02](./02-backend.md), [03](./03-frontend.md),
   [05](./05-database.md).
4. Read [06](./06-business-logic.md) for *why* the app behaves the way it does, not just *how*.
5. Before writing any code, read [10](./10-feature-development.md) for the workflow that matches
   your task, and [`../../AGENTS.md`](../../AGENTS.md) for the contribution conventions.
6. Before shipping, check [13](./13-architecture-decisions-and-limitations.md) — you may be about to
   touch a documented limitation or a real finding worth knowing about first.

## How this suite was produced

This suite was generated by a multi-phase investigation: ten parallel research passes read the
actual source (not just existing docs or filenames) across backend, frontend, database, Redis,
background processing, API surface, security, business logic, and testing; their findings were then
synthesized into the fourteen numbered chapters plus this index, with every chapter's author
independently re-checking a sample of the research's own citations against current source and
correcting several (documented inline in each chapter, typically in a closing "Findings and
corrections" section). It reflects the repository at commit `f28809c` on branch
`claude/gallant-einstein-wfmdag`. Code moves; citations can drift — if you find one that no longer
matches, treat the surrounding explanation as still generally reliable and the specific line number
as the part to re-verify.
