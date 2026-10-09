# 14 — Glossary

Eleven-Chat is a heavily customized LibreChat fork. Much of its vocabulary is invented here and
does not appear in upstream LibreChat docs. Examples are "Subagent completion wakeup", "Agent
trigger capability shield" and "Event actor head". The canonical source for those terms is the
repository's [`CONTEXT.md`](../../CONTEXT.md). That file is dense and precise, but it does not say
where each concept lives in the code or whether anyone has checked the definition against that
code recently.

This glossary covers both. For each fork-specific term it gives a plain-language explanation first.
It then describes the mechanism, cites the implementing code, links to the guide chapter that covers
it in depth, and labels how much of the definition was confirmed against source.

## How to read the verification labels

Each entry uses one of three labels.

| Label | Meaning |
|---|---|
| **Verified** | The behavior was traced to implementing code with `file:line` evidence during the documentation research pass. You can rely on the citation as a starting point. |
| **Inferred** (partially verified) | The concept clearly exists in code: its storage shape, type, or entry-point function was located and its doc comment matches. However, the detailed behavior that `CONTEXT.md` describes was **not** traced statement by statement. The entry states exactly which part was confirmed. Any claim beyond that is quoted or paraphrased from `CONTEXT.md` and marked *per CONTEXT.md, not independently re-verified in this pass*. |
| **Unknown** | No implementing code was located. The definition is reproduced from `CONTEXT.md` and attributed to it. Treat it as the intended design, not as confirmed behavior. |

Some entries cite code that was located only while this glossary was being written, and not by the
earlier research reports. Those entries say "located in the glossary pass". In every such case the
identifier and its doc comment were read, but the logic behind them was not.

Line numbers are the last known locations as of 2026-10-09, and they drift. Search for the cited
identifier if a line has moved.

**Related chapters in this guide**

- [00 — Overview](./00-overview.md) and [01 — Architecture](./01-architecture.md): the big picture.
- [02 — Backend](./02-backend.md) and [03 — Frontend](./03-frontend.md): request path, layers and
  client state.
- [04 — Redis](./04-redis.md): Lua scripts, job-store keys, leader election.
- [05 — Database](./05-database.md): Mongoose, TTL indexes, tenant isolation.
- [06 — Business Logic](./06-business-logic.md): code-environment sealing, enforced model specs,
  balance, tool approval and resume security.
- [07 — API Reference](./07-api-reference.md): steer, queued-turn and subagent routes.
- [08 — Auth & Security](./08-auth-security.md): JWT, token retirement, RBAC/ACL, encryption
  generations, SSRF.
- [09 — Background Processing](./09-background-processing.md): schedules, Agent Trigger delivery,
  subagents, MCP, the resumable stream and shutdown.
- [10 — Feature Development](./10-feature-development.md) and
  [11 — Testing & Debugging](./11-testing-debugging.md): working in the code.
- [12 — Configuration & Deployment](./12-configuration-deployment.md): `librechat.yaml`, env vars,
  shutdown.
- [13 — Architecture Decisions & Limitations](./13-architecture-decisions-and-limitations.md): why
  things are shaped this way.

The deepest original sources are [`CONTEXT.md`](../../CONTEXT.md),
[`CRITICAL_FLOWS.md`](../../CRITICAL_FLOWS.md) and [`PROJECT_MAP.md`](../../PROJECT_MAP.md).

### At-a-glance status of the `CONTEXT.md` terms

| Term | Status | Primary code location |
|---|---|---|
| Agent continuation preparation | Inferred | `packages/api/src/agents/triggers/continuation.ts` |
| Agent event actor mailbox | Inferred | `AgentTriggerDelivery` ordering-lane index |
| Agent event expected action | Inferred | `packages/api/src/agents/triggers/expectedAction.ts` |
| Agent event handling outcome | Inferred | `packages/api/src/agents/triggers/outcome.ts` |
| Agent execution context | Inferred | `packages/api/src/agents/runtime.ts` |
| Agent execution enrollment | Inferred | `packages/api/src/agents/remote/lifecycle.ts` |
| Agent execution host | Inferred | `packages/api/src/agents/remote/host.ts` |
| Agent queued turn | Inferred | `packages/api/src/agents/queuedTurns.ts` |
| Agent run envelope | Inferred | `packages/api/src/agents/envelope.ts` |
| Agent trigger capability shield | Inferred | `packages/data-schemas/src/schema/triggerDelivery.ts` |
| Agent turn execution plan | Inferred | `packages/api/src/agents/plan.ts` |
| Attached code environment | Inferred | `packages/api/src/code/` |
| Caller Capability Projection | Inferred | `packages/api/src/agents/callerCapabilities.ts` |
| Conversation code-environment decision | **Verified** | `packages/api/src/code/decision.ts` |
| Effective agent selection | **Verified** | `canAccessAgentFromBody.js`, `buildEndpointOption.js` |
| Event actor head | Inferred | `Conversation.agentEventActor*` fields |
| Event actor invocation fork | Inferred | `packages/api/src/agents/triggers/actor.ts` |
| Event actor receipt | Inferred | `packages/data-schemas/src/types/triggerDelivery.ts` |
| Live subagent task owner | **Verified** | `packages/api/src/agents/subagentThreads.ts` |
| MCP direct OpenID bearer | **Verified** (core) | `packages/api/src/mcp/openid.ts` |
| MCP OAuth prompt projection | **Verified** | `packages/api/src/mcp/oauth/resume.ts` |
| MCP runtime request body | Inferred | `packages/api/src/mcp/request.ts` |
| Scheduled run admission | **Verified** | `packages/api/src/schedules/fire.ts`, `engine.ts` |
| Subagent activity stream | **Verified** | `packages/api/src/agents/subagentThreads.ts` |
| Subagent completion wakeup | **Verified** | `packages/api/src/agents/subagentCompletionWakeup.ts` |
| Subagent thread | **Verified** | `packages/api/src/agents/subagentThreads.ts` |
| Theme definition | Inferred | `packages/client/src/theme/` |
| Turn delivery routing | Unknown | not located |
| Warm terminal steer continuation | Inferred | `packages/api/src/agents/steering/runtime.ts`, `RedisJobStore` steer keys |

---

## Domain & Business Concepts

This section holds the fork-specific terms. Terms taken from `CONTEXT.md` are listed with
everything else. Terms marked *(supplementary)* do not appear in `CONTEXT.md`, but they come up
constantly in the code and in this guide.

### Agent continuation preparation

**Status: Inferred.** The entry-point function was located in the glossary pass. The adapter-dispatch
logic was not traced.

**In plain terms:** some background work, such as an event or a finished subagent, needs an agent
to pick up a conversation where it left off. Just before that resumed turn is admitted, one function
decides which source-specific adapter is allowed to prepare it.

**Mechanism:** `createAgentContinuationResolver` is in
`packages/api/src/agents/triggers/continuation.ts:10-11`, a 24-line module. Its doc comment reads
*"Selects the single continuation adapter authorized for this delivery."* The following details are
per CONTEXT.md and were not independently re-verified in this pass:

- Bound Event Actor work selects its binding adapter.
- Internal completion work selects an adapter by stable source identity.
- Preparation may resolve input and branch state, or settle work that was already consumed.
- It does not own result truth, delivery ordering or generation execution.

**Deep dive:** [09 — Background Processing](./09-background-processing.md) (Agent Trigger delivery).
Original definition: [`CONTEXT.md`](../../CONTEXT.md).

### Agent event actor mailbox

**Status: Inferred.** The storage for an ordering lane is verified. The mailbox serialization logic
was not traced.

**In plain terms:** an external source, such as a webhook bound to an agent, can fire many events in
a row. The mailbox makes sure the bound agent handles them one at a time and in order. The next event
waits until the current turn has a final recorded outcome.

**Mechanism:** the `AgentTriggerDelivery` collection
(`packages/data-schemas/src/schema/triggerDelivery.ts`) has an ordering-lane index on
`{orderingKey, status, laneSequence}`. That index is verified by the background-processing research,
and it is the storage a per-binding ordering lane would need. The identifier "mailbox" also appears
in `packages/data-schemas/src/methods/triggerDelivery.ts` and
`packages/api/src/agents/triggers/outcome.ts`; that is a name match only, and the code was not read.

Per CONTEXT.md, not independently re-verified: the mailbox keeps later deliveries queued after
transport admission until the current child turn records an authoritative terminal
[handling outcome](#agent-event-handling-outcome). It serializes coalesced batches and individual
events without becoming a second execution controller or checkpoint store.

**Deep dive:** [09 — Background Processing](./09-background-processing.md),
[`triggers/README.md`](../../packages/api/src/agents/triggers/README.md).

### Agent event expected action

**Status: Inferred.** The matcher function was located in the glossary pass. The surrounding policy
was not traced.

**In plain terms:** an event source can say what it expects the agent to do, for example "call tool
X with argument Y". The host then checks whether that actually happened, based on the tool steps the
run really completed. It does not rely on the model's claim that it succeeded.

**Mechanism:** `matchesExpectedAction` is in
`packages/api/src/agents/triggers/expectedAction.ts:39`, alongside `CompletedToolEvidence` at `:3`.
The recorder doc comment in `packages/api/src/agents/triggers/outcome.ts:210-215` describes the
fences that apply:

- the exact tool name, including the MCP suffix form
- the declared argument subset
- an error-free result
- exclusion of background non-execution receipts

Per CONTEXT.md, the expected action is *evidence policy, not authorization and not a model-authored
success claim*.

**Deep dive:** [09 — Background Processing](./09-background-processing.md).

### Agent event handling outcome

**Status: Inferred.** The lifecycle states and the classifier function are located. Generation
fencing was not traced.

**In plain terms:** "we received your event" is not the same as "the agent did the work". Each
accepted event delivery therefore gets a separate, durable record of what the agent actually did
with it.

**Mechanism:** the Agent Trigger README documents the public `handling` lifecycle as
`started → applied | completed_no_action | failed | cancelled`. That lifecycle is kept separate from
the delivery `status` (`pending | leased | succeeded | dead`) on `POST /api/agents/v1/events`. This
was verified from the README by the background-processing research. The classifier
`classifyAgentEventRunOutcome` (`packages/api/src/agents/triggers/outcome.ts:157`) has the doc
comment *"Classifies terminal run evidence once for both checkpoint commit and public receipt"*; it
was located in the glossary pass. Per CONTEXT.md, the record is generation-fenced, and `started`
proves generation admission.

**Deep dive:** [09 — Background Processing](./09-background-processing.md),
[`triggers/README.md`](../../packages/api/src/agents/triggers/README.md).

### Agent execution context

**Status: Inferred.** The interface and its doc comment were read in the glossary pass. Its
consumers were not traced.

**In plain terms:** this is the runtime information an agent run needs, such as who the user is,
the app configuration and the conversation facts. It deliberately excludes the HTTP
request/response objects, so the same run can be started from HTTP, a background trigger or a
future host.

**Mechanism:** `AgentExecutionContext` is in `packages/api/src/agents/runtime.ts:7-13`. Its doc
comment reads *"Runtime-only state required to initialize and execute an Agent run. This context
deliberately contains no transport objects."* A factory at `:35` creates it at the existing HTTP
adapter seam. Per CONTEXT.md, it is rehydrated beside an [Agent run envelope](#agent-run-envelope)
and never contains serialized credentials.

**Deep dive:** [02 — Backend](./02-backend.md),
[`CRITICAL_FLOWS.md` Flow 1](../../CRITICAL_FLOWS.md#flow-1-message-lifecycle-most-important).

### Agent execution enrollment

**Status: Inferred.** The class and several matching doc comments were read in the glossary pass.
The full drain and deletion semantics were not traced.

**In plain terms:** this is the bookkeeping that follows one agent run from admission to final
cleanup. It makes sure the run is marked finished only after its usage, artifacts and stored
response have all been written. It also makes sure that deleting a user or a conversation cannot
race a run that is still writing data.

**Mechanism:** `AgentExecutionEnrollment` is in `packages/api/src/agents/remote/lifecycle.ts:40`,
and `enrollAgentExecution` is at `:200`. The doc comments at `:96`, `:135` and `:163` cover three
cases:

- a provider-start CAS whose response may have been lost
- a failed terminal write being retried instead of becoming a "drained running job"
- the decision not to publish a drain marker when terminalization gives up

All three match CONTEXT.md's description. The full delete-all and exact-conversation deletion
semantics in CONTEXT.md were **not** traced:

- owner fence
- repeated drain sweeps
- idempotent cleanup over the deleted-ID set

**Deep dive:** [09 — Background Processing](./09-background-processing.md) (shutdown and drain).
Original definition: [`CONTEXT.md`](../../CONTEXT.md).

### Agent execution host

**Status: Inferred.** The entry function and its doc comment were read in the glossary pass. Its
callers were not traced.

**In plain terms:** this is the one place that runs an agent. It does not care whether the request
came in through the chat UI, the OpenAI-compatible API or the Responses API. It handles starting
the run, cancelling it when the client disconnects, and recording how it finished. The protocol
adapters only validate input and render output.

**Mechanism:** `executeAgentRun` is in `packages/api/src/agents/remote/host.ts:26-31`, an 85-line
module. Its doc comment reads *"Owns admission, cancellation, provider-start fencing, and settlement
for a validated Agent run. Protocol implementations own only execution semantics; ingress adapters
own transport validation and final rendering."* That closely matches CONTEXT.md.

**Deep dive:** [01 — Architecture](./01-architecture.md), [02 — Backend](./02-backend.md),
[`CRITICAL_FLOWS.md` Flow 1](../../CRITICAL_FLOWS.md#flow-1-message-lifecycle-most-important).

### Agent queued turn

**Status: Inferred.** The collection, lifecycle factory, constants and routes are verified. The
admission state machine was not traced.

**In plain terms:** you can send another message while the agent is still answering. That message
waits in a durable server-side queue and runs after the current answer finishes. If a server crashes,
the system shows that the message is in an uncertain state. It does not quietly drop the message or
send it twice.

**Mechanism:** verified parts, from the background-processing and business-logic research:

- **Storage:** the `QueuedTurn` collection in `packages/data-schemas/src/schema/queuedTurn.ts`, with
  `queuedTurnSequence.ts` alongside it.
- **Lifecycle factory:** `createAgentQueuedTurnLifecycle` in
  `packages/api/src/agents/queuedTurns.ts:1072`, which is a 1,118-line file.
- **Constants:** `CLAIM_LEASE_MS = 2 * 60_000`, reconciliation backoff from 5 s up to 5 min, and a
  process-fenced `PROCESS_CLAIM_OWNER`.
- **Routes:** `GET /api/agents/chat/queued-turns` and `DELETE /api/agents/chat/queued-turns/:queuedTurnId`.

Per CONTEXT.md, not independently re-verified in this pass:

- The Mongo row is the sole FIFO, payload and lifecycle authority.
- A deterministic delivery identity is reserved before publication.
- Admission waits for a clean durable predecessor outcome.
- A process death before the generation receipt commits leaves "admission-indeterminate" evidence.
- Conversation deletion cancels the rows and retires their deliveries.

**Deep dive:** [06 — Business Logic](./06-business-logic.md) (queued turns),
[09 — Background Processing](./09-background-processing.md) and
[07 — API Reference](./07-api-reference.md) (routes).

### Agent run envelope

**Status: Inferred.** The version constant and factory doc comment were read in the glossary pass.
The rehydration path was not traced.

**In plain terms:** this is a small, versioned, JSON-safe "work order" for one agent run. It is
created after the request has been authenticated and validated, and it carries only the validated
payload and the user's identifiers. Everything else is reloaded from those identifiers on the side
that executes the run.

**Mechanism:** `AGENT_RUN_ENVELOPE_VERSION = 1` is in `packages/api/src/agents/envelope.ts:7`, and
the envelope type's `version` field is at `:38`. The factory doc comment at `:110` reads *"Creates
the versioned, transport-safe request that crosses the agent execution seam."* A separate envelope
type serves background work: `createAgentTriggerEnvelope` in `packages/api/src/agents/triggers/`,
which is verified. Do not confuse the two.

**Deep dive:** [02 — Backend](./02-backend.md),
[09 — Background Processing](./09-background-processing.md).

### Agent trigger capability shield

**Status: Inferred.** The storage shape exactly matches the definition (verified). The worker
capability advertisement and claim-arbitration logic were not traced.

**In plain terms:** during a rolling deploy, old and new server versions run at the same time. Some
queued background work must only be run by the newer servers. The shield stores that work so older
servers can see it but never claim it.

**Mechanism:** the `AgentTriggerDelivery.status` enum in
`packages/data-schemas/src/schema/triggerDelivery.ts:126-142` contains `staging`,
`capability_staging`, `batched`, `pending`, `capability_pending`, `leased`, `capability_leased`,
`succeeded`, `capability_dead` and `dead`. The private lease fields at `:143-148` are only written by
capability-aware workers: `requiredWorkerCapability`, `capabilityStatus`, `capabilityLeaseBy`,
`capabilityLeaseUntil` and `capabilityLeaseClaimToken`. Versioned capability constants such as
`AGENT_TRIGGER_WORKER_CAPABILITY_QUEUED_TURN_V1/V2` exist and are imported by `queuedTurns.ts`. The
Redis side, a "versioned fail-closed terminal status and recovery index", is per CONTEXT.md. A
versioned Redis recovery set, `stream:agent_event_detached:terminal_host_action:v1`, was observed by
the Redis research. Per CONTEXT.md, this is an implementation detail at the storage seam and never a
user-facing mode.

**Deep dive:** [09 — Background Processing](./09-background-processing.md) (Agent Trigger delivery),
[04 — Redis](./04-redis.md) (recovery sets), [05 — Database](./05-database.md) (schema).

### Agent trigger delivery *(supplementary)*

**Status: Verified.**

**In plain terms:** this is the fork's general-purpose durable queue for "wake an agent up later".
Schedules, webhooks, internal events and subagent completions all produce the same envelope and drop
it here.

**Mechanism:**

- **Storage:** Mongo collection `AgentTriggerDelivery`. A unique index on `deliveryKey` provides
  idempotency.
- **Delivery semantics:** at-least-once delivery, exponential backoff that honors `Retry-After`, and
  durable dead letters.
- **Retention:** succeeded records expire after 90 days through a TTL index.
- **Polling:** idle-recovery polling runs from 30 s up to 2 min, configurable under
  `endpoints.agents.eventDriven.idlePolling`.
- **Execution:** `AgentTriggerExecutionHost` dispatches by a real HTTP
  [self-loopback](#self-loopback-dispatch-supplementary) to `/api/agents/chat`.

Code: `packages/api/src/agents/triggers/{service,host,envelope,dispatch}.ts` and
`packages/api/src/agents/triggers/README.md`.

**Deep dive:** [09 — Background Processing](./09-background-processing.md).

### Agent turn execution plan

**Status: Inferred.** The plan type and resolver were read in the glossary pass. Its callers were not
traced.

**In plain terms:** after the system knows who is asking, which agent answers and which tools load,
it decides once how this turn will load state. There are three options: resume from a saved
checkpoint, rebuild from message history, or start fresh. Nothing later in the turn re-decides that.

**Mechanism:** `packages/api/src/agents/plan.ts` defines the following:

- `AgentTurnOrigin`, which is `'user' | 'subagent' | 'completion' | 'schedule' | 'event' | 'resume'` (`:4`)
- `AgentTurnContinuationStrategy`, which is `'checkpoint' | 'history' | 'fresh'` (`:6`)
- the `AgentTurnExecutionPlan` interface (`:13`)
- `resolveAgentTurnExecutionPlan` (`:64-65`), whose doc comment reads *"Compiles already-loaded turn
  facts into one immutable state-loading decision"*

`api/server/controllers/agents/request.js` uses it. Per CONTEXT.md, a checkpoint failure falls back
to durable history within the same Agents lifecycle; that fallback was not traced.

**Deep dive:** [02 — Backend](./02-backend.md),
[`CRITICAL_FLOWS.md` Flow 1](../../CRITICAL_FLOWS.md#flow-1-message-lifecycle-most-important).

### Attached code environment

**Status: Inferred.** The conversation-level decision that gates it is verified (see the next
entries). The worker, Code API and sandbox side runs outside this repository and was not traced.

**In plain terms:** a user can connect a real machine or VM, running a `librechat-code` worker, as a
persistent workspace where an agent can run code and edit files. LibreChat chooses which workspace a
chat uses and enforces approval rules. The worker's own sandbox is the final limit on what can
execute.

**Mechanism:** the following parts are verified.

- **User preference:** `PATCH /api/user/preferences { statefulCodeEnvironment }`. It is validated
  against `endpoints.agents.statefulCodeSessions.allowedEnvironments` (`preferences.ts:40-45`).
- **Approval baseline:** when an attached environment is present and the endpoint has not set
  `enabled` either way, `resolveToolApprovalPolicy` (`packages/api/src/agents/hitl/policy.ts:70`)
  synthesizes `{enabled: true, mode: 'bypass'}`.
- **Workspace binding:** binding to a conversation goes through the
  [Conversation code-environment decision](#conversation-code-environment-decision).

Per CONTEXT.md, the interface is runtime-neutral (native SRT, WSL2, Docker/NsJail) and never leaks
host paths into agent tools. That claim was not independently re-verified in this pass.

**Deep dive:** [06 — Business Logic](./06-business-logic.md),
[`docs/workspace-checkouts.md`](../workspace-checkouts.md),
[`docs/tool-approval-modes.md`](../tool-approval-modes.md).

### Balance reservation *(supplementary)*

**Status: Verified.**

**In plain terms:** credits are held when a request is admitted, not just deducted afterwards. This
stops several simultaneous requests from all spending the same remaining balance.

**Mechanism:** `checkBalance` (`packages/api/src/middleware/checkBalance.ts:199`) reserves
`amount × multiplier` through an injected `reserveBalance`. The hold is renewed at half its TTL,
measured from the *stored* expiry (`:112-162`). Release is idempotent and waits for pending
admissions. Insufficient credit throws a JSON-encoded `TOKEN_BALANCE` violation, which is also
logged with score 0. Schedules auto-disable after `BALANCE_SKIP_DISABLE_THRESHOLD = 5` consecutive
balance skips (`schedules/fire.ts:17`). The exact disable call site is Inferred.

**Deep dive:** [06 — Business Logic](./06-business-logic.md).

### Caller Capability Projection

**Status: Inferred.** The host-side consumer was read in the glossary pass. The classification
itself is produced by the external `@librechat/agents` SDK, which is not in this repository.

**In plain terms:** the SDK reports which tools are currently callable, and by whom: directly by the
model, or programmatically from code. LibreChat receives that report as data and cross-checks it
against its own trusted tool registry. It never treats the report as permission.

**Mechanism:** `resolveCallerCapabilityProjectionSnapshot` is in
`packages/api/src/agents/callerCapabilities.ts:4-8`. Its doc comment reads *"Accepts only complete
snapshots for the version this host understands. Missing or future versions intentionally fall back
to the legacy registry projection during a rolling SDK/host deployment."* It is referenced from
`agents/handlers.ts` and the OpenAI-compatible and Responses controllers. Per CONTEXT.md, LibreChat
never recomputes deferred-tool discovery policy; that claim was not independently re-verified in this
pass.

**Deep dive:** [`CONTEXT.md`](../../CONTEXT.md) (original definition).

### Conversation code-environment decision

**Status: Verified.** The pure-function contract is verified. The "refused while a generation is
running" rule is Inferred.

**In plain terms:** the first message in a chat fixes whether that chat has an attached workspace and
which one. Later turns, retries, resumes and other entry points cannot quietly add a workspace or
swap it. The owner can explicitly move the chat to a different live workspace, and only when the
deployment enables moves.

**Mechanism:** `resolveConversationCodeEnvironmentDecision` is in
`packages/api/src/code/decision.ts:73`, with the sealing check at `:84-109`. A request that
conflicts with a sealed decision throws `CodeWorkspaceSelectionError('locked')`; the other error
codes are `'invalid'` and `'required'`. The rule that no run overwrites a stored decision lives in
`resolvePersistableCodeEnvironmentDecision` (`:181`). A conversation that never involved a
code-capable agent holds *no* decision, so its first coding turn still gets to choose.

The owner move is `resolveConversationCodeEnvironmentMove` (`:134`), gated by
`statefulCodeSessions.conversationMoves.enabled`. Its `from` argument must exactly match the sealed
selections, so a stale client cannot replace a decision it has not seen. Moving to the same set, or
detaching a chat that was never attached, is refused. The refusal while a generation is running,
awaiting approval or saving is per CONTEXT.md; it is probably enforced by the caller, and that was
not traced. Tests are in `packages/api/src/code/decision.spec.ts`.

**Deep dive:** [06 — Business Logic](./06-business-logic.md).

### Conversation fork *(supplementary — not the same as "Event actor invocation fork")*

**Status: Verified.**

**In plain terms:** the user branches a conversation into a brand-new, independent chat. The original
chat is never modified.

**Mechanism:** `POST /api/convos/fork` calls `forkConversation`
(`api/server/utils/import/fork.js:71`). It supports three modes: `DIRECT_PATH`, `INCLUDE_BRANCHES`
and the default `TARGET_LEVEL`, which does a breadth-first level walk that keeps sibling branches.
It mints fresh `messageId`s and nudges `createdAt` forward by 1 ms where needed to keep ordering
stable.

**Deep dive:** [06 — Business Logic](./06-business-logic.md),
[`CRITICAL_FLOWS.md` Flow 4](../../CRITICAL_FLOWS.md#flow-4-conversation--context-management).

### Effective agent selection

**Status: Verified.** The research also found a duplication risk, described below.

**In plain terms:** when an admin forces users onto a preset model spec, the agent bound to that spec
must be the one that is authorized *and* the one that runs. Whatever `agent_id` the browser sent is
ignored.

**Mechanism:** when `modelSpecs.enforce` is true, `resolveEnforcedAgentId`
(`api/server/middleware/accessResources/canAccessAgentFromBody.js:18`, used at `:177`) picks the
spec's agent for the ACL check. `buildEndpointOption.js:88-135` applies the same spec preset to the
request. Failures return 400: `'No model spec selected'`, `'Invalid model spec'`,
`'Model spec mismatch'` or `'agent_id is required in request body'`.
[Ephemeral agents](#ephemeral-agent-supplementary) skip the ACL resource check.

**Caveat:** the two call sites each re-derive the enforced identity independently instead of sharing
one value computed once. Today they agree because they use the same resolver, but this is exactly the
divergence CONTEXT.md says must not happen.

**Deep dive:** [06 — Business Logic](./06-business-logic.md),
[08 — Auth & Security](./08-auth-security.md).

### Ephemeral agent *(supplementary)*

**Status: Verified.** This is narrow: only the ACL bypass was confirmed.

**In plain terms:** this is an ad hoc agent configuration assembled for a single chat and never saved
as an Agent document. Because it is not a stored resource, it has no ACL entries.

**Mechanism:** `isEphemeralAgentId` short-circuits the per-resource ACL check
(`canAccessAgentFromBody.js:192`). On resume, `modelLabel` is replayed for ephemeral agents because
their LangGraph node name derives from it. This was a regression fix for issue #14253
(`packages/api/src/agents/hitl/policy.ts`, `RESUME_CONTEXT_KEYS`).

**Deep dive:** [06 — Business Logic](./06-business-logic.md).

### Event actor head

**Status: Inferred.** The storage fields and the commit/snapshot method names are confirmed. The CAS
and reconciliation-journal logic was not traced.

**In plain terms:** an agent that is bound to an external event source keeps a private "where I left
off" pointer to its latest saved LangGraph checkpoint, plus one previous checkpoint for safe cleanup.
The pointer only moves forward when the agent actually performed the expected action.

**Mechanism:** confirmed by the database and business-logic research:

- **Storage:** the `Conversation` schema (`packages/data-schemas/src/schema/convo.ts`) carries
  `agentEventBinding` and `agentEventActor*` fields. They are `select:false`, so they are never
  projected by default, and they hold opaque LangGraph checkpoint and suspension state.
- **Methods:** `getAgentEventBinding`, `getAgentEventActorSnapshot` and `commitAgentEventActorState`
  exist in `packages/data-schemas/src/methods/conversation.ts`.
- **Write protection:** `saveConvo` strips actor checkpoint fields from caller updates, so a generic
  save cannot overwrite them.

Per CONTEXT.md, not independently re-verified in this pass:

- The head advances only through compare-and-swap, and only on a qualifying applied action.
- A legacy-path event cold-marks the head for rebuild from history.
- Commit conflicts and post-commit failures go into a private reconciliation journal that blocks
  later actor turns until an exact marker is cleared.

**Deep dive:** [09 — Background Processing](./09-background-processing.md),
[05 — Database](./05-database.md) (Conversation schema).
Original definition: [`CONTEXT.md`](../../CONTEXT.md).

### Event actor invocation fork

**Status: Inferred.** The concept is located: a resume function and a matching strategy enum were
read in the glossary pass. Almost none of the long `CONTEXT.md` definition was traced.

**In plain terms:** each time an event-bound agent handles one delivery, it works on a scratch copy
(a "fork") of its saved checkpoint. If the expected action happens, the fork becomes the new head.
Otherwise the fork is thrown away. A paused fork, for example one waiting for tool approval or for an
Ask User answer, can be resumed safely on any server.

**Mechanism:** located in the glossary pass:

- `packages/api/src/agents/triggers/actor.ts:904` has the doc comment *"Resumes one signed suspended
  fork on any replica using the Conversation as authority."* The file is 1,368 lines.
- The `fresh | history | checkpoint` adapters named in CONTEXT.md match
  `AgentTurnContinuationStrategy` in `packages/api/src/agents/plan.ts:6`.

Everything else is per CONTEXT.md and not independently re-verified in this pass:

- signed, versioned suspension evidence
- generation protocol v2 selection
- the provider-start CAS written immediately before the continuation gate opens
- an approval projection exposed before the persistence barrier
- precedence of a pending interrupt over expected-action evidence
- terminal no-action retirement

Not to be confused with a [Conversation fork](#conversation-fork-supplementary--not-the-same-as-event-actor-invocation-fork).

**Deep dive:** [`CONTEXT.md`](../../CONTEXT.md) holds the full definition.
[09 — Background Processing](./09-background-processing.md) covers the surrounding delivery system.

### Event actor receipt

**Status: Inferred.** The receipt type and its host location are confirmed. Replay and recovery use
was not traced.

**In plain terms:** after an event-bound agent finishes handling one delivery, a small private proof
of the result is stored on that delivery's record. It holds which checkpoint resulted and which
action was taken, but none of the prompt, event, tool arguments or output. It is used for replay and
crash recovery.

**Mechanism:** the `AgentEventActorReceipt` interface is in
`packages/data-schemas/src/types/triggerDelivery.ts:42`. Its host collection, `AgentTriggerDelivery`,
is verified. `packages/api/src/agents/triggers/actor.ts:46` documents an *"Authenticated binding that
owns the delivery receipt"*. Per CONTEXT.md, the receipt does not own the actor checkpoint, and the
conversation keeps only the [head](#event-actor-head) and any unresolved reconciliation until the
receipt is durable. That claim was not independently re-verified in this pass.

**Deep dive:** [09 — Background Processing](./09-background-processing.md).

### Generation job *(supplementary)*

**Status: Verified.**

**In plain terms:** in this fork an AI response is a server-side job, not a long-lived HTTP request.
Closing the tab does not stop the job, and reopening it re-attaches to the same response.

**Mechanism:** `GenerationJobManager` (`packages/api/src/stream/GenerationJobManager.ts`) runs the
sequence `claimGeneration → createJob → subscribe/emitChunk → completeJob/abortJob`. The job is
claimed under an [idempotency key](#idempotency-key) of `(userId, clientRequestId, streamId)`
(`:3322-3409`). It is backed by an in-memory store or `RedisJobStore`, chosen once at startup by
`createStreamServices.ts:72-137` (`USE_REDIS_STREAMS`, which defaults to `USE_REDIS`). Mid-stream
errors are persisted to Mongo *before* the terminal SSE `error` event is sent, through the
`beforeErrorPublication` hook.

**Deep dive:** [09 — Background Processing](./09-background-processing.md),
[04 — Redis](./04-redis.md) (a generation job through Redis),
[`CRITICAL_FLOWS.md` Flow 1](../../CRITICAL_FLOWS.md#flow-1-message-lifecycle-most-important).

### Live subagent task owner

**Status: Verified.**

**In plain terms:** while a subagent is running, exactly one API server process "holds" it. That
process owns the executor, the abort switch and the control queue. Other replicas can forward
"steer" or "cancel" commands to it through Redis, but the running subagent never moves between
servers.

**Mechanism:** `TaskThreadLease` (`packages/api/src/agents/subagentThreads.ts:206-234`) holds the
per-process abort and execution promise and a bounded control queue. It can also hold a Redis lease
with `DEFAULT_LEASE_TTL_MS = 30s`, renewed every `DEFAULT_LEASE_HEARTBEAT_MS = 10s` (`:66-67`).
Cross-replica routing (`RedisSubagentTaskControlTransport`) is wired only when `USE_REDIS` is on;
without Redis, subagents stay in the process that started them
(`api/server/services/Endpoints/agents/subagentThreadStore.js:110-160`). Control commands (`steer`,
`queue`, `interrupt` and `cancel`) are recorded in a durable receipt ledger, so a retried command
replays its prior outcome (`replayDurableControl`, `:910-1005`). On shutdown, owned children are
cancelled and drained for up to 45 s.

**Deep dive:** [09 — Background Processing](./09-background-processing.md).

### MCP direct OpenID bearer

**Status: Verified (core mechanism).** The retry rule is per CONTEXT.md.

**In plain terms:** an operator can configure a remote MCP server so that LibreChat forwards the
logged-in user's own OpenID access token as the `Authorization` header. The MCP server then sees the
real user.

**Mechanism:** `packages/api/src/mcp/openid.ts:10-11` resolves the
`{{LIBRECHAT_OPENID_ACCESS_TOKEN}}` and `{{LIBRECHAT_OPENID_TOKEN}}` placeholders in a server's
configured `Authorization` header. It uses the user's live OpenID token through an injected
`UpstreamTokenProvider`. Per CONTEXT.md, not independently re-verified in this pass: after a forced
session refresh it may replace one rejected connection, but it never automatically replays the
rejected tool call.

**Deep dive:** [09 — Background Processing](./09-background-processing.md) (MCP),
[08 — Auth & Security](./08-auth-security.md) (OpenID).

### MCP OAuth prompt projection

**Status: Verified.**

**In plain terms:** an agent may be waiting for the user to authorize an MCP server through an OAuth
login. A browser that reconnects to the resumable stream gets a clean, token-free list of the
authorization prompts that are still pending, so it can redraw them without replaying the whole event
log.

**Mechanism:** `projectPendingMCPOAuthPrompts(replayEvents)` is in
`packages/api/src/mcp/oauth/resume.ts:50`. It derives the projection from a resumable generation's
`ResumeState.replayEvents`. The OAuth flow state itself sits in Lua-guarded `FLOWS` keys
(`packages/api/src/flow/manager.ts`).

**Deep dive:** [09 — Background Processing](./09-background-processing.md),
[04 — Redis](./04-redis.md) (flow keys),
[`CRITICAL_FLOWS.md` Flow 6](../../CRITICAL_FLOWS.md#flow-6-frontend-state--data-fetching)
(SSE reconnection).

### MCP runtime request body

**Status: Inferred.** The factory and its doc comment were read in the glossary pass. The placeholder
substitution sites were not traced.

**In plain terms:** some MCP server configurations use placeholders such as "current conversation
ID" in their headers. Those values are supplied only while that one agent request is running. They
are never written into the shared server definition, which other users also use.

**Mechanism:** `createMCPRuntimeRequestBody` is in `packages/api/src/mcp/request.ts:9-14`, which
re-exports `MCPRuntimeRequestBody` at `:6`. Its doc comment explains that an explicit `null` parent
becomes a root sentinel. An omitted parent stays omitted, so protocols without parent-message
identity *fail closed* for configurations that need that placeholder. `AGENTS.md` cites
`MCPRequestContext.js` as the model of thin `/api` wiring around this module.

**Deep dive:** [09 — Background Processing](./09-background-processing.md) (MCP).

### Message tree *(supplementary)*

**Status: Verified.**

**In plain terms:** a conversation is a tree of messages linked by `parentMessageId`, not a list.
Editing or regenerating creates a sibling branch instead of overwriting anything. The model only ever
sees one path through the tree.

**Mechanism:** `BaseClient.getMessagesForConversation` (`api/app/clients/BaseClient.js:1499`) walks
from the leaf to the root, with a cycle guard. Message saves are upserts keyed by `messageId`, and an
authoritative re-save happens before the `final` SSE event (`request.js:3048`). The active branch is
client-only view state (`siblingIdxFamily`).

**Deep dive:** [06 — Business Logic](./06-business-logic.md),
[`CRITICAL_FLOWS.md` Flow 4](../../CRITICAL_FLOWS.md#flow-4-conversation--context-management).

### Resume context and request fingerprint *(supplementary)*

**Status: Verified.**

**In plain terms:** a run can pause for tool approval or an Ask User question. When it resumes, the
server rebuilds exactly the agent, model and tools that were paused. It does not trust what the
resume request claims, so a crafted resume cannot swap in a different setup.

**Mechanism:** `RESUME_CONTEXT_KEYS` (`packages/api/src/agents/hitl/policy.ts:455`) lists every field
that determines the graph. `applyResumeContext` (`:773`) overwrites the client's values and
**deletes** fields that the paused turn never carried. `computeAgentRequestFingerprint` (`:848`)
hashes the request with SHA-256, and a legacy variant at `:875` covers mixed-version deploys.
Credential-shaped model parameters are stripped before persistence (`:572-671`).

**Deep dive:** [06 — Business Logic](./06-business-logic.md),
[`docs/tool-approval-modes.md`](../tool-approval-modes.md).

### Schedule and ScheduleRun *(supplementary)*

**Status: Verified.**

**In plain terms:** a Schedule is a saved "run this agent with this prompt on this cadence" rule. Each
time it fires, a ScheduleRun row is written.

**Mechanism:** both live in `packages/data-schemas/src/schema/{schedule,scheduleRun}.ts`.

- **Cadence:** `hourly | daily | weekdays | weekly`, or raw cron. `croner` is used *only* to compute
  the next occurrence, with DST handling and deterministic jitter (`schedules/cadence.ts`).
- **Engine:** a custom 30 s polling engine (`schedules/engine.ts`) claims due schedules under a
  5-minute [lease](#lease-and-claim-token). Occurrences more than 15 min overdue are skipped forward
  (`MISFIRE_GRACE_MS`).
- **Run limits:** at most one `started` run per schedule, and the global run capacity, are both
  enforced by [partial unique indexes](#partial-unique-index).
- **Availability:** disabled by default (`interface.schedules`, plus the `SCHEDULES_DISABLED` kill
  switch). The engine refuses to arm across multiple replicas unless the job store is Redis-backed or
  `SCHEDULES_SINGLE_PROCESS=true` is set.

**Deep dive:** [09 — Background Processing](./09-background-processing.md),
[12 — Configuration & Deployment](./12-configuration-deployment.md) (enabling schedules).

### Scheduled run admission

**Status: Verified.**

**In plain terms:** before a scheduled occurrence takes one of the limited "generation slots", it
re-checks everything as of right now:

- Does the owner still exist?
- Is scheduling still allowed for them?
- Can they still reach the agent?
- Are the files and MCP servers ready?

A slow or failing check therefore never ties up a slot.

**Mechanism:** `fireSchedule` (`packages/api/src/schedules/fire.ts:126`) runs these steps in order:

1. Resolve `nextRunAt`. If it is null, the schedule is disabled with reason `'invalid_schedule'`.
2. Rehydrate the owner with `getUserContext`. If the owner is missing, the schedule is disabled with
   reason `'permission_revoked'`.
3. Re-resolve owner-scoped limits.
4. Resolve files.
5. Run the MCP preflight.
6. Only then, call `withCapacitySlot` (`schedules/capacity.ts:23`).

The capacity slot is a DB-unique partial index on `capacitySlot` scoped to `status:'started'`
(`scheduleRun.ts:180-186`). It is not a race-prone "count then insert". Every write is fenced on a
rotating `claimToken`. A fire superseded by an owner edit steps aside without advancing the schedule
(`stepAsideSuperseded`, `fire.ts:214`).

**Deep dive:** [06 — Business Logic](./06-business-logic.md),
[09 — Background Processing](./09-background-processing.md).

### Self-loopback dispatch *(supplementary)*

**Status: Verified.**

**In plain terms:** background work does not call the agent code directly. The server sends an
authenticated HTTP POST to its own chat endpoint, so a scheduled or event-driven turn goes through
exactly the same path as a human message.

**Mechanism:** `packages/api/src/agents/triggers/host.ts:628-654` POSTs to `/api/agents/chat` with a
minted bearer token and the `x-lc-agent-trigger: 1` header. The base URL comes from
`AGENT_TRIGGERS_SELF_URL` or the bound listener address. The route sets `req._isAgentTrigger`
(`api/server/routes/agents/index.js:158`), and `ResumableAgentController` branches on it
(`api/server/controllers/agents/request.js:983-1107`).

**Deep dive:** [09 — Background Processing](./09-background-processing.md).

### Steer *(supplementary)*

**Status: Verified** (routes and storage).

**In plain terms:** a steer is a message sent *while* the agent is working. It gets injected at the
next tool boundary instead of waiting for the agent to finish. A message that should wait until the
agent finishes is a [queued turn](#agent-queued-turn) instead.

**Mechanism:**

- **Routes:** `POST /api/agents/chat/steer`, plus `/steer/deliver`, `/steer/cancel` and
  `/steer/arm`. Arming escalates a steer to an interrupt.
- **Storage:** with Redis, steers live in `stream:{id}:steers`, a FIFO list, and
  `stream:{id}:steers-claimed`. Parked steers have their own 300 s TTL. Idempotency receipts are
  capped at 100 per stream.
- **Concurrency:** all of these are mutated by Lua scripts (`enqueueSteer`, `drainSteers`,
  `closeAndDrainSteers`, `armSteer`).
- **SDK requirement:** steering requires an SDK that can both inject and replay steer parts
  (`packages/api/src/agents/steering/runtime.ts:27-38`). Otherwise the route returns 501.

**Deep dive:** [07 — API Reference](./07-api-reference.md) (steer routes),
[04 — Redis](./04-redis.md) (steer keys),
[09 — Background Processing](./09-background-processing.md).

### Subagent activity stream

**Status: Verified.**

**In plain terms:** this is the live progress feed you see in a subagent's side panel. It is
best-effort and for display only. It never contains hidden reasoning and never controls the subagent.
If live events are missed, the panel falls back to polling the saved child thread.

**Mechanism:** `SubagentActivityStream` (`packages/api/src/agents/subagentThreads.ts:684`) is backed
by `InMemoryEventTransport`, or by `RedisEventTransport` when Redis is configured. Size is bounded by
`SUBAGENT_ACTIVITY_LIMITS`, publishing retries with backoff (`retryActivity`, `:1375-1400`), and a
`pre-drain` shutdown task drains it. It is served as SSE on
`GET /api/convos/:parentConversationId/subagents/:threadId/tasks/:taskId/activity`.

**Deep dive:** [09 — Background Processing](./09-background-processing.md).

### Subagent completion wakeup

**Status: Verified.**

**In plain terms:** when a parent agent hands work to a subagent, a durable reminder is filed
*before* the subagent starts: "wake the parent when this finishes". A server crash mid-task therefore
cannot lose the hand-back. The parent resumes only after its own current turn has settled.

**Mechanism:** `packages/api/src/agents/subagentCompletionWakeup.ts:22-23,92-121` enqueues a
`continue`-mode [Agent Trigger delivery](#agent-trigger-delivery-supplementary). The event source is
`{type:'internal', id:'subagent-completion'}`, the event type is `subagent.completion`, and the
payload is `{taskId, threadId, subagentType}`. It carries task metadata, not child output. Delivery
is deferred while the parent is active (`isParentActive` and `isParentWorking`, `:140-152`). It is
then dispatched through the [self-loopback](#self-loopback-dispatch-supplementary). The parent turn
collects the result through `claimSubagentTaskResult`. The wiring is `onTaskPrepared` and
`onTaskSettled` in `subagentThreadStore.js:55-85`.

**Deep dive:** [09 — Background Processing](./09-background-processing.md).

### Subagent thread

**Status: Verified.**

**In plain terms:** a subagent's work is saved as a hidden, read-only child conversation under the
parent chat. The parent agent can continue the same thread later by its `threadId`. Humans can view
it but cannot type into it.

**Mechanism:** it is a regular Conversation plus message tree. `subagentThread.parentConversationId`
and `rootConversationId` link it to its parent, and the generic single-conversation read returns 404
for it. It is persisted by `SubagentThreadTaskStore`
(`packages/api/src/agents/subagentThreads.ts:631`), which extends the SDK's
`InMemorySubagentTaskStore`. The transcript lives in `Message.subagentTranscript`, capped at 12 MB
(`:104,455-481`). Writes to a child thread fail with 409 `CHILD_THREAD_READ_ONLY_ERROR`. The
`/api/convos/:parentConversationId/subagents/...` routes expose list, view and control.

**Deep dive:** [09 — Background Processing](./09-background-processing.md),
[07 — API Reference](./07-api-reference.md) (subagent routes),
[`docs/run_files.md`](../run_files.md) (files shared with subagents).

### Theme definition

**Status: Inferred.** The interface and the module README were read in the glossary pass, and the
token system is verified by the frontend research. The validator and resolver code was not traced.

**In plain terms:** a theme is pure data. It names colors for LibreChat's semantic roles, such as
"surface" or "text-primary", optionally with separate light and dark values. It cannot carry custom
CSS, behavior or layout changes.

**Mechanism:** `ThemeDefinition { version: 1; ... }` and `ResolvedThemeDefinition` are in
`packages/client/src/theme/types/index.ts:756-765`. The doc comment reads *"Versioned, data-only
theme input. Missing values resolve against LibreChat defaults."* The token list is shared with
`librechat-data-provider` so that the server validates against the same set (`theme/registry.ts:53`).
Semantic tokens live in `packages/client/src/theme/tokens.css` inside `@theme inline`, so a runtime
switch through `ThemeProvider`/`applyTheme` re-colors Tailwind utilities. `DeploymentTheme` resolves
the theme in this order of precedence:

1. high-contrast
2. `librechat.yaml`
3. the build-time environment
4. the user's stored choice

**Deep dive:** [03 — Frontend](./03-frontend.md),
[`packages/client/src/theme/README.md`](../../packages/client/src/theme/README.md),
[`AGENTS.md`](../../AGENTS.md) (theming rules),
[`PROJECT_MAP.md`](../../PROJECT_MAP.md) §9.

### Token retirement *(supplementary)*

**Status: Verified.**

**In plain terms:** after a password reset or a 2FA change, every token issued earlier stops working
on the very next request. The user does not have to wait for the token to expire.

**Mechanism:** JWTs carry no roles. They carry a custom millisecond `issuedAtMs` claim
(`packages/data-schemas/src/methods/user.ts:781-801`), because the standard one-second `iat` cannot
order a token mint against a reset that lands in the same second. `jwtStrategy.js:28-82` reloads the
user on every request. It rejects a token that predates `credentialsChangedAt` (`isTokenRetired`,
`packages/api/src/auth/twoFactor.ts:237-245`), and rejects requests while account deletion is in
progress with `ACCOUNT_DELETION_IN_PROGRESS`. Because the user document is cached for burst
requests, every user-document mutation must invalidate that cache; see `AGENTS.md`, "Backend auth
cache".

**Deep dive:** [08 — Auth & Security](./08-auth-security.md),
[`CRITICAL_FLOWS.md` Flow 2](../../CRITICAL_FLOWS.md#flow-2-authentication--authorization).

### Tool approval policy (HITL) *(supplementary)*

**Status: Verified.**

**In plain terms:** this is "human in the loop": rules for which agent tool calls must pause for a
person to approve. It is off unless an admin turns it on. The exception is an attached code
environment, which gets a safe default.

**Mechanism:** `resolveToolApprovalPolicy` (`packages/api/src/agents/hitl/policy.ts:70`) layers the
endpoint, agent, skill and attached-environment policies. Only the endpoint config owns `enabled`,
and `isHITLEnabled` (`:102`) requires `enabled === true`. A `deny` always wins (`:173-194`).
`ask_user_question` is exempt from approval unless an admin names it (`:984`).

**Deep dive:** [06 — Business Logic](./06-business-logic.md),
[`docs/tool-approval-modes.md`](../tool-approval-modes.md).

### Turn delivery routing

**Status: Unknown.** No implementing code was located in this pass. Searches for `turnRoute`,
`deliveryRoute` and `TurnDeliveryRoute` returned nothing.

**Definition (per CONTEXT.md, not independently re-verified in this pass):** this is the per-agent,
request-local value that decides how each attachment reaches the model on one turn: `provider`,
`text` or `none`. Initialization settles it once, after the provider swap and the Responses API
decision, under the endpoint's own name and the media dialect its config declares. Every reader of a
turn route consumes that one value instead of deriving it from the agent. A stored route is an
upload-time inference that this value resolves again for the turn; a destination the user chose
stands.

**Related verified context:** provider-specific attachment limits live in
`packages/api/src/files/validation.ts`, covered in [06 — Business Logic](./06-business-logic.md).
[`docs/run_files.md`](../run_files.md) describes how provider and extracted-text delivery consume a
model's attachment budget.

### Usage cost computation *(supplementary)*

**Status: Verified.**

**In plain terms:** the cost shown live while a response streams and the amount actually billed come
from the same single function, so the two cannot drift apart.

**Mechanism:** `computeUsageCostUSD` (`packages/api/src/agents/usage.ts:215`) normalizes provider
cache-token quirks. Bedrock counts cache tokens additively, and Vertex AI omits thoughts from
`output_tokens`. Rates resolve through `getMultiplier` (`packages/data-schemas/src/methods/tx.ts:627`)
in this order, falling back to `defaultRate = 6`:

1. the admin `endpointTokenConfig`
2. the premium tier
3. the static `tokenValues` table

**Deep dive:** [06 — Business Logic](./06-business-logic.md).

### Warm terminal steer continuation

**Status: Inferred.** The steer storage, Lua scripts and SDK hook types are verified or located. The
StopFinalize decision and FIFO batch claim were not traced.

**In plain terms:** a [steer](#steer-supplementary) can arrive just as the agent is about to finish.
If it arrives before the cut-off, the agent keeps going in the *same* run instead of starting a new
response. Anything arriving after the cut-off becomes an ordinary follow-up message.

**Mechanism:** the following is confirmed.

- **Redis storage (Redis research):**
  - the steer FIFO `stream:{id}:steers`
  - the claimed-but-uncommitted list `stream:{id}:steers-claimed`
  - parked steers, which survive job-hash deletion
  - steer receipts
- **Lua script (Redis research):** `closeAndDrainSteers`.
- **SDK hook types (located in the glossary pass):** `TerminalSteerHookInput`, derived from the SDK's
  `Stop` hook input, and `TerminalSteerHook`, at
  `packages/api/src/agents/steering/runtime.ts:15,21`. `StopFinalize`-related identifiers also appear
  in `agents/run.ts` and `agents/hooks/`; that is a name match only.

Per CONTEXT.md, not independently re-verified in this pass:

- After parallel Stop hooks fold, a serialized StopFinalize phase tells the job store whether a
  continuation is already planned.
- The store atomically chooses between three outcomes:
  - claim the current protocol-v2 FIFO batch
  - keep empty admission open
  - seal admission
- Claimed steer receipts are the crash-recovery authority.
- Protocol-v1 generations always seal.

**Deep dive:** [09 — Background Processing](./09-background-processing.md),
[04 — Redis](./04-redis.md) (steer keys and scripts).

---

## Technical & Infrastructure Terms

These are standard industry terms. Each one is explained as this application uses it.

### `@librechat/agents` SDK

This external npm package does the actual LangGraph-style agent execution, provider calls and tool
loops. It is *not* in this repository. The repo plugs in fork-specific implementations of the SDK's
interfaces, for example `SubagentThreadTaskStore extends InMemorySubagentTaskStore`. Behavior that
lives inside the SDK, such as the Stop-continuation loop or the Caller Capability classification,
cannot be verified from this repository. See [09](./09-background-processing.md) and
[`CRITICAL_FLOWS.md` Flow 3](../../CRITICAL_FLOWS.md#flow-3-multi-provider-llm-abstraction).

### AES encryption generations (v1, v2, v3)

Three secret-encryption schemes coexist in `packages/data-schemas/src/crypto/index.ts`:

| Version | Functions | Cipher | IV | Location |
|---|---|---|---|---|
| v1 | `encrypt` | AES-CBC | fixed, from env `CREDS_IV` | `:27-65` |
| v2 | `encryptV2` | AES-CBC | random | `:74-112` |
| v3 | `encryptV3` | `aes-256-ctr` | random, `v3:` prefix | `:123-148` |

None of the three authenticates its ciphertext. New 2FA and admin secrets use v3. Per-user provider
keys (`methods/key.ts:121`) still use v1. See [08](./08-auth-security.md).

### AsyncLocalStorage (tenant context)

This Node API carries per-request context through async calls without passing it explicitly. The fork
stores the current tenant in it (`packages/data-schemas/src/config/tenantContext.ts`), and Mongoose
middleware reads it to scope every query. See [Multi-tenancy](#multi-tenancy-tenant-isolation) and
[05 — Database](./05-database.md).

### CAS (compare-and-swap)

A CAS write succeeds only if the current value still matches what the writer expects. It is the
fork's main concurrency primitive. Examples:

- Redis Lua `transitionStatus`, with `expectActionId`/`expectCreatedAt` guards
- Mongo lease `claimToken` fencing
- the event-actor head, which is CAS per CONTEXT.md

See [Lua script](#lua-script), [Lease](#lease-and-claim-token) and [04 — Redis](./04-redis.md).

### Dead letter

A dead letter is a queued work item that has permanently failed and stopped retrying. In
`AgentTriggerDelivery`, rows end up with status `dead`, or `capability_dead` for capability-gated
work. Dead letters persist until someone explicitly requeues them, unlike succeeded rows, which
expire after 90 days. See [09](./09-background-processing.md).

### Dependency injection (method factories)

Modules receive their dependencies instead of importing singletons. All database access goes through
`create<Domain>Methods(mongoose, deps)` factories in `packages/data-schemas/src/methods/`, composed
by `createMethods` and exposed to legacy code as `~/models`. `AGENTS.md` requires the same for new
`packages/api` code. The MCP static singletons (`MCPManager.getInstance()`) are the explicit
counter-example. See [05 — Database](./05-database.md) and [02 — Backend](./02-backend.md).

### Graceful shutdown (pre-drain / post-drain)

`packages/api/src/app/shutdown.ts` handles `SIGTERM`, `SIGINT`, `SIGQUIT` and `SIGHUP` in one
coordinated sequence:

1. `pre-drain` tasks run concurrently with `httpServer.close()`. They stop admitting work, close SSE
   streams and stop the schedule timer.
2. `post-drain` tasks wait, within a time budget, for detached generations and subagents to persist.
3. A 60 s force-exit timer is the final backstop.

Work still running at force-exit is reconciled as orphaned by the next process. See
[09](./09-background-processing.md) and [12](./12-configuration-deployment.md).

### Hash tag (Redis Cluster)

A hash tag is the `{...}` part of a Redis key. In Cluster mode it forces related keys onto the same
slot, so a Lua script can update them atomically. Every per-generation key, such as
`stream:{streamId}:job`, is hash-tagged this way. Global sets like `stream:running` cannot share that
slot, so on Cluster the status guard is documented as best-effort (`IJobStore.ts:1169-1172`). See
[04 — Redis](./04-redis.md).

### Idempotency key

An idempotency key is a client-supplied identifier that makes a retried request attach to the
original result instead of repeating the work. Examples in this app:

- **Chat:** `clientRequestId` keys `claimGeneration`, so a retried POST never double-bills.
- **Remote events:** the `Idempotency-Key` header on `POST /api/agents/v1/events`.
- **Background work:** the unique `deliveryKey` on `AgentTriggerDelivery`.
- **Steers:** `clientSteerId` receipts.

### JWT access and refresh tokens

- **Access token:** a short-lived bearer JWT (`JWT_SECRET`) that holds only identity. Roles are
  re-read from Mongo on each request.
- **Refresh token:** a separate JWT (`JWT_REFRESH_SECRET`) sent only as an `httpOnly`,
  `SameSite=Strict` cookie and tracked in the `Session` collection.

See [Token retirement](#token-retirement-supplementary) and [08](./08-auth-security.md).

### Jotai and Recoil

These are two React state libraries. The client is migrating from Recoil to Jotai: new state must use
Jotai, and app-global preferences are passed into features rather than read from `~/store`.
Persisted atoms use `client/src/store/jotai-utils.ts`. See `AGENTS.md`, "Client state ownership",
[03 — Frontend](./03-frontend.md) and [`CRITICAL_FLOWS.md` Flow 6](../../CRITICAL_FLOWS.md#flow-6-frontend-state--data-fetching).

### LangGraph checkpoint

A checkpoint is a saved snapshot of an agent graph's state, which allows a paused or event-bound run
to resume exactly where it stopped. Checkpoints back tool-approval pauses and the
[Event actor head](#event-actor-head). Conversation deletion must also delete them
(`openCheckpointDeletion`, checkpoint pruning in `packages/api/src/conversations/save.ts`).

### Lease and claim token

A lease is a time-limited claim on a piece of work, held by one worker and renewed while that worker
is alive. If the worker dies, the lease expires and another worker can take over. The fork uses
leases throughout:

- schedules, with `leaseUntil`/`leaseBy`/`claimToken` and a 5 min lease
- queued turns, with a 2 min lease
- trigger deliveries
- subagent owners, with a 30 s lease and 10 s heartbeat

A rotating **claim token** fences writes, so a worker whose lease was taken over cannot overwrite
newer state.

### Leader election

Leader election picks exactly one replica to run cluster-wide singleton tasks.
`packages/api/src/cluster/LeaderElection.ts` uses `SET LeadingServerUUID <uuid> NX EX ~25s`, renews
every 10 s, and resigns with a Lua check-and-delete. Without Redis, `isLeader()` always returns
`true`, because every single-process instance is its own leader. See [04 — Redis](./04-redis.md).

### `librechat.yaml` and `configSchema`

`librechat.yaml` is the main operator configuration file. It is validated by the Zod `configSchema`
in `packages/data-provider/src/config.ts`, which is more than 5,500 lines. Loading happens in
`packages/api/src/app/loader.ts`. Per-tenant and per-principal overrides are cached for 60 s
(`packages/api/src/app/service.ts:203`) and attached to each request as `req.config`. `AGENTS.md`
requires every new limit, timeout or toggle to get a field here. See
[12 — Configuration & Deployment](./12-configuration-deployment.md) and [`CRITICAL_FLOWS.md` Flow 5](../../CRITICAL_FLOWS.md#flow-5-configuration-system).

### Lua script

A Lua script is a small program that Redis runs atomically on the server. No other command can
interleave with it. The fork relies on Lua scripts heavily:

- job status CAS
- idempotency claims
- the steer queue
- terminal-event publication
- MCP OAuth flow transitions (`flow/manager.ts`)
- the per-user concurrency limiter
- leader resignation

TTL extensions inside these scripts are written as "extend-only", so a later write can never shorten
a key's remaining life. See [04 — Redis](./04-redis.md).

### MCP (Model Context Protocol)

MCP is an open protocol for connecting LLM agents to external tool and data servers. LibreChat acts
as an MCP *client*:

- **Connection management:** `MCPManager`, `MCPServersRegistry`.
- **Tool wrapping:** discovered tools become LangChain tools in `mcp/tools.ts`.
- **Authentication:** per-user OAuth (`mcp/oauth/`), [direct OpenID bearer](#mcp-direct-openid-bearer)
  credentials, and [runtime request body](#mcp-runtime-request-body) placeholders.

Outbound connections are SSRF-hardened. See [09](./09-background-processing.md).

### MCP Apps

MCP Apps are interactive UIs (`ui://` resources) that an MCP server provides. They render in a
sandboxed cross-origin iframe, and every tool call or chat message an App initiates needs host
confirmation. See [`docs/mcp-apps.md`](../mcp-apps.md).

### Meilisearch

Meilisearch is an optional full-text search index, holding a derived copy of `Conversation` and
`Message` data. It is synced by the `mongoMeili` Mongoose plugin only when `MEILI_HOST` and
`MEILI_MASTER_KEY` are set. Mongo remains the source of truth. See [05 — Database](./05-database.md).

### Mixed-version (rolling-deploy) compatibility

During a rolling deploy, old and new server versions share the same Mongo and Redis. Many fork
features explicitly handle that overlap:

- legacy request fingerprints
- legacy generation-claim keys
- protocol-v1 and protocol-v2 generations
- the [capability shield](#agent-trigger-capability-shield)
- the version fallback in [Caller Capability Projection](#caller-capability-projection)

`AGENTS.md` lists mixed-version behavior as one of the invariants to review. See
[13 — Architecture Decisions & Limitations](./13-architecture-decisions-and-limitations.md).

### Mongoose

Mongoose is the MongoDB object modeling library. In this fork, schemas live in
`packages/data-schemas/src/schema/` (about 58 files), and models are built by
`create<Name>Model(mongoose)` factories that attach the tenant-isolation and Meilisearch plugins.
`AGENTS.md` forbids exposing Mongoose types (`FilterQuery`, `Types.ObjectId`, `Document`) in
exported signatures outside `data-schemas`. See [05 — Database](./05-database.md).

### Mongoose-backed RBAC and ACL

Access control is enforced by two independent systems, both stored in Mongo:

1. **Capabilities.** These are category-level permissions such as `SystemCapabilities.ACCESS_ADMIN`.
   They come from `Role.permissions` and `SystemGrant` rows, and are checked by
   `hasCapability`/`requireCapability`. This is the real gate on `/api/admin/*`; the legacy
   `checkAdmin` middleware is unused.
2. **Per-resource ACL.** `AclEntry` rows grant a principal a permission bitmask on one resource:
   `VIEW=1`, `EDIT=2`, `DELETE=4`, `SHARE=8` and `VIEW_INSIGHTS=16`. Named templates such as
   `AGENT_VIEWER`, `AGENT_EDITOR` and `AGENT_OWNER` live in `AccessRole`. Checks go through
   `canAccessResource`.

The capability bypass is checked first; a failure there falls through to the ACL check and is not a
denial by itself. See [08](./08-auth-security.md) and
[`docs/permissions/acl-write-compatibility.md`](../permissions/acl-write-compatibility.md).

### `mongodb-memory-server`

This package runs a real in-memory MongoDB for tests. `AGENTS.md` prefers it over mocking the
database. Most `packages/data-schemas/src/methods/*.spec.ts` suites use it. See
[11 — Testing & Debugging](./11-testing-debugging.md).

### Multi-tenancy (tenant isolation)

The fork has its own multi-tenancy layer. `applyTenantIsolation(schema)` adds Mongoose middleware to
every model that scopes queries by `tenantId`, which it reads from
[AsyncLocalStorage](#asynclocalstorage-tenant-context). Indexes are tenant-qualified, for example
`{email, tenantId}`. On unauthenticated routes the tenant comes from an `X-Tenant-Id` header, which
is trusted only when `TRUST_TENANT_HEADER=true`. The reserved value `__SYSTEM__` is rejected
(`packages/api/src/middleware/preAuthTenant.ts:38-65`). See [05](./05-database.md) and
[06](./06-business-logic.md).

### Partial unique index

A partial unique index is a MongoDB unique index that applies only to documents matching a filter.
The fork uses it to make the database the arbiter of concurrency instead of running a racy count:

- one `started` ScheduleRun per schedule
- one global `capacitySlot` per running ScheduleRun
- per-user schedule `slot` enforcement of `maxPerUser`

See [05 — Database](./05-database.md).

### Provider drain

Provider drain is the point at which an LLM provider call has provably stopped and can no longer
write user data. `abortJob({awaitProviderDrain: true})` waits up to 30 s for it
(`PROVIDER_DRAIN_TIMEOUT_MS`, `IJobStore.ts:93`). Deletion and shutdown logic waits for drain before
removing data.

### React Query

React Query (TanStack Query) is the client's server-state cache. `AGENTS.md` requires it for all API
interactions, with query and mutation keys defined in `packages/data-provider/src/keys.ts` and
related queries invalidated after each mutation. See [03 — Frontend](./03-frontend.md).

### Redis pub/sub and Redis Streams

The fork uses these two Redis features differently:

- **Streams** (`XADD`/`XRANGE` on `stream:{id}:chunks`) hold the durable, replayable event log of a
  generation.
- **Pub/sub** (`stream:{id}:events`) pushes live chunks to subscribers across replicas. It needs a
  second, dedicated `ioredis` connection.

Sequence numbers are assigned atomically in Lua (`publishWithSequence`). See
[04 — Redis](./04-redis.md).

### Resumable stream

A resumable stream is an SSE stream that a client can disconnect from and later reconnect to, picking
up where it left off. Reconnection replays from `durableEventSequence` with Redis, or from an
in-process snapshot without it. Reliable cross-replica resume therefore requires Redis. See
[Generation job](#generation-job-supplementary) and
[`CRITICAL_FLOWS.md` Flow 6](../../CRITICAL_FLOWS.md#flow-6-frontend-state--data-fetching).

### SSE (Server-Sent Events)

SSE is a one-way HTTP streaming format from server to browser. It is the only real-time channel for
chat; the app does not use WebSockets. Agent chat uses two calls:

1. `POST /api/agents/chat/:endpoint` returns a `streamId` almost immediately.
2. `GET /api/agents/chat/stream/:streamId` is the actual SSE connection.

The `ws` package is only a transitive dependency. See [07 — API Reference](./07-api-reference.md)
and [01 — Architecture](./01-architecture.md).

### SSRF (server-side request forgery)

SSRF tricks a server into calling internal addresses. The fork defends against it in two layers:

1. **Preflight checks:** `isPrivateIP` and `isSSRFTarget` reject private ranges, tunneled IPv6
   addresses and Docker/Kubernetes service names (`packages/api/src/auth/{ip,domain}.ts`).
2. **Connect-time re-check:** `createSSRFSafeUndiciConnect` validates the IP again at connect time,
   which closes the DNS-rebinding gap. It is used for MCP, LLM clients and OAuth discovery.

Operator exemptions must be listed as exact `host:port` pairs in `allowedAddresses`. See
[08](./08-auth-security.md).

### `tsdown` (build without typecheck)

`tsdown` is the bundler for `packages/api`, `packages/client` and `packages/data-schemas`. It emits
JavaScript **without checking types**, so a green build does not prove the types are correct. Run
`npx tsc --noEmit` in each workspace you changed, as `AGENTS.md` requires. See
[11 — Testing & Debugging](./11-testing-debugging.md) and
[12 — Configuration & Deployment](./12-configuration-deployment.md).

### TTL index

A TTL index is a MongoDB index with `expireAfterSeconds` that automatically deletes a document once a
date field passes. It is the fork's main retention mechanism, and nothing in the fork uses
`isDeleted` soft-delete flags. TTL indexes cover:

- `Session`, `Token` and `RefreshTokenBridge`
- `AgentApiKey`, `File` and `SharedLink`
- `Message` and `Conversation` (`expiredAt`, for temporary chats)
- `ScheduleRun` (`settledAt`, 90 days)
- succeeded `AgentTriggerDelivery` rows (90 days)

This is distinct from Redis key TTLs, such as job hashes, which expire 300 s after completion. See
[05 — Database](./05-database.md) and [04 — Redis](./04-redis.md).

### Zod

Zod is the TypeScript schema-validation library behind `configSchema` and request validators such as
`agentCreateSchema` (`packages/api/src/agents/validation.ts:615`). Configuration is validated with
`configSchema.strict().safeParse(...)`.
