---
name: agent-systems-engineering
description: Design, implement, review, and extend persistent, event-driven, concurrent Agent runtimes and plugins. Use when building Agent frameworks, Harnesses, Orchestrators, event systems, task/session management, tool runtimes, plugin systems, durable execution, or proactive agents.
---

# Agent Systems Engineering

## Mission

Use this skill to design Agent systems as reliable software systems, not as a prompt wrapped around a ReAct loop.

The target system may be:
- a long-running personal assistant or AI companion;
- a coding agent or team-level Super-agent;
- an Agent runtime / Harness;
- a plugin or extension for an existing Agent host;
- an event-driven automation system;
- a multi-agent execution platform.

The design goal is to balance four properties:

1. **Continuous** — interactions and task context survive across messages and channels.
2. **Productive** — independent work can execute concurrently without blocking the interaction path.
3. **Proactive** — environmental events can initiate work under explicit policy.
4. **Persistent** — tasks and state survive process restarts, worker loss, and temporary environment disconnection.

Do not assume all four properties are required for every plugin. First identify the requested product boundary and apply only the necessary mechanisms.

## Core engineering stance

- Treat the LLM as a probabilistic decision component inside a deterministic runtime.
- Keep authorization, state transitions, scheduling, retries, cancellation, idempotency, and resource limits in code.
- Use the model for semantic interpretation, planning, ambiguous decisions, and synthesis.
- Use deterministic code for stable rules, routing, validation, and repeatable operations.
- Persist intent and state before performing external side effects.
- Make every long-running operation observable, cancellable, and recoverable where feasible.
- Separate facts, decisions, permissions, and execution results.
- Prefer explicit state machines and typed contracts over implicit prompt conventions.
- Never claim a prototype implements a paper's full architecture unless verified in code.

## Workflow

For every framework or plugin task, follow this sequence.

### 1. Discover the host and the actual requirements

Inspect before designing:
- host framework and extension API;
- language/runtime and supported versions;
- lifecycle hooks and event model;
- session/context ownership;
- tool registration and execution semantics;
- persistence facilities;
- concurrency model;
- process model and deployment environment;
- security/capability model;
- tests, conventions, and existing abstractions.

Resolve the requested deliverable:
- new runtime;
- plugin / extension;
- adapter / connector;
- tool;
- event source;
- worker;
- UI/control panel;
- operational service.

Do not introduce a second runtime when the task only requires a plugin. Do not force a plugin abstraction onto a standalone runtime.

### 2. Write a one-page design before coding

Record:
- problem statement and non-goals;
- actors and trust boundaries;
- event flow;
- state ownership;
- lifecycle and state transitions;
- concurrency and resource limits;
- failure and recovery behavior;
- public interfaces;
- test strategy;
- acceptance criteria.

Draw a sequence diagram or state diagram for any asynchronous or multi-step workflow.

### 3. Map requirements to system mechanisms

| Requirement | Candidate mechanism |
|---|---|
| Cross-message continuity | Session identity, durable context, correlation IDs |
| Multiple independent tasks | Router/Worker, task queue, isolated worker context |
| Proactive behavior | Event Sources, Trigger pipeline, policy gates |
| Non-blocking interaction | Async jobs, background execution, completion events |
| Restart recovery | Durable state, checkpoints, leases, reconciliation |
| Avoid repeated reasoning | Fast/slow path, cached decisions, guarded reuse |
| Large context | Inline/Spill, locality-aware retrieval, references |
| Safe plugin extension | Registry, typed events, commands, capabilities |
| Operational reliability | Health checks, watchdog, bounded retries, telemetry |

### 4. Implement the smallest vertical slice

Start with one end-to-end path:
1. receive one event or request;
2. validate and authorize it;
3. create or correlate a task;
4. execute one bounded operation;
5. persist the outcome;
6. publish a result event;
7. expose status and logs.

Then add concurrency, more event sources, recovery, and optimization incrementally.

### 5. Verify behavior, not just compilation

Run:
- unit tests for pure logic and state transitions;
- contract tests for plugin interfaces;
- integration tests for event → task → worker → result;
- concurrency tests for duplicate and simultaneous events;
- restart/recovery tests;
- authorization and prompt-injection tests;
- load tests for queue growth and backpressure;
- regression tests for token, latency, and task quality.

Report what was tested, what was not, and what remains a prototype.

---

# Architecture reference

## 1. Router / Worker: control plane and execution plane

Use a two-level design when the system must remain responsive while multiple long-running tasks execute.

### Router responsibilities

- receive human messages and external triggers;
- maintain the global interaction/session view;
- classify, correlate, prioritize, and route events;
- create, update, cancel, or delegate tasks;
- enforce policy and authorization;
- summarize worker progress to the user;
- avoid expensive or blocking task execution.

### Worker responsibilities

- execute one task or a tightly bounded task group;
- own isolated task context and execution environment;
- use the tools required for the task;
- report progress, blockers, and results through typed messages;
- never independently speak to the user unless explicitly designed as a user-facing worker.

### Invariants

- Worker context is isolated by default.
- Router owns global coordination, not every implementation detail.
- Worker cannot expand its authorization merely by receiving more context.
- A task has one authoritative owner at a time, or an explicit coordination protocol.
- Worker output is a proposal/result; the runtime validates state transitions.
- Worker lifecycle is managed by the host runtime, not by prompt instructions alone.

### Suggested interfaces

```ts
type TaskStatus =
  | "queued"
  | "assigned"
  | "running"
  | "waiting"
  | "blocked"
  | "completed"
  | "failed"
  | "cancelled"
  | "expired";

interface Task {
  id: string;
  type: string;
  status: TaskStatus;
  sessionId: string;
  correlationId: string;
  createdAt: number;
  updatedAt: number;
  deadlineAt?: number;
  assignedWorkerId?: string;
  inputRef?: string;
  resultRef?: string;
}
```

Do not store large transcripts or binary payloads directly in task rows. Store references and bounded summaries.

## 2. Event-driven architecture

Separate these concepts:

- **Event**: an immutable fact that something happened.
- **Trigger**: an event selected for evaluation or routing.
- **Task**: work the system intends to perform.
- **Command**: an instruction to cause a state change or side effect.
- **Effect**: an external side effect such as sending a message or modifying a file.
- **Receipt**: evidence of the result of an attempted effect.

Suggested event envelope:

```ts
interface AgentEvent<T = unknown> {
  id: string;
  type: string;
  source: string;
  occurredAt: number;
  receivedAt: number;
  correlationId: string;
  causationId?: string;
  tenantId?: string;
  payload: T;
  metadata?: Record<string, unknown>;
}
```

### Trigger pipeline

Use a staged pipeline:

1. **Source prefilter** — discard irrelevant scope, self-generated messages, malformed events, and events outside the subscription.
2. **Deterministic gate / Fast Path** — apply allowlists, identity, source, scope, rate limits, deduplication, and explicit authorization.
3. **Semantic assessment / Slow Path** — ask an LLM only when interpretation is genuinely required.
4. **Policy decision** — accept, ignore, defer, ask a human, or create/update a task.
5. **Routing** — associate the trigger with a task/thread/session.
6. **Audit** — record the decision and its basis.

Default proactive mode should be observe-only:
- `off`: do not evaluate inferred triggers;
- `observe`: evaluate and record what would have been accepted, but do not execute;
- `auto`: execute only after policy and authorization checks.

Do not make `auto` the initial deployment default.

### Delivery semantics

Assume event delivery is at-least-once unless the transport proves otherwise. Design consumers to be idempotent.

Use:
- stable event IDs;
- idempotency keys for side effects;
- deduplication windows;
- source cursors / read positions;
- transactional outbox when publishing must match a database commit;
- explicit retry and dead-letter policies.

Do not promise exactly-once execution across arbitrary external systems. Prefer “at-least-once delivery + idempotent effects + reconciliation”.

### Subscription and context are not authorization

A subscription means “deliver this context”. It does not mean “grant permission to act”.

Keep these separate:
- Policy: versioned rules governing what may be accepted.
- Mandate / Grant: immutable authorization captured when a task is accepted.
- Subscription: which event threads or resources are visible to a task.
- Capability: what a plugin/worker is technically allowed to invoke.

A worker following a related thread must not gain new authority from that thread.

## 3. Asynchronous jobs and full-duplex interaction

Do not await a long-running job inside the interaction request path.

Prefer:

```ts
const job = await jobs.start({
  taskId,
  operation: "run_tests",
  input: { command: "npm test" }
});

return { jobId: job.id, status: "running" };
```

Completion becomes a new event:

```ts
await events.publish({
  type: "job.completed",
  source: "job-runtime",
  correlationId,
  payload: { jobId, resultRef }
});
```

Provide explicit operations:
- `start`
- `status`
- `cancel`
- `resume`
- `wait` (bounded)
- `tail` / `output` (bounded)

### Backpressure

Define:
- max concurrent workers;
- max queued tasks;
- per-user / per-source quotas;
- priority classes;
- queue timeout;
- overload behavior;
- cancellation policy for stale work.

When saturated, queue, reject, coalesce, or degrade. Never allow unbounded task creation.

### Cancellation

Cancellation is a control-plane operation, not just a prompt message.

Propagate cancellation through:
- task;
- worker;
- tool call;
- child process;
- subagent;
- external job where supported.

Use cooperative cancellation first, then bounded hard termination. Persist terminal state and cancellation reason.

## 4. Durable state and recovery

Separate three planes:

- **Compute**: ephemeral process executing the agent loop.
- **State**: durable task/session/checkpoint/decision data.
- **Environment**: filesystem, browser, local device, credentials, GPU, repositories, and other external resources.

A runtime restart must not erase task intent.

Persist at least:
- task state and version;
- session identity and context references;
- current assignment / worker lease;
- checkpoint or last safe continuation point;
- pending effects;
- effect receipts;
- retry count and next retry time;
- cancellation/deadline;
- source cursors;
- policy version and authorization grant.

### State machine discipline

- Define legal transitions.
- Reject illegal transitions in code.
- Use optimistic version checks or compare-and-swap for concurrent updates.
- Record transition reason and actor.
- Distinguish `failed`, `blocked`, `waiting`, `cancelled`, and `expired`.
- Do not infer “completed” from a worker process exiting.

### Crash consistency

For external side effects:
1. persist intent;
2. execute effect with idempotency key if supported;
3. persist receipt;
4. reconcile ambiguous outcomes after restart.

A crash between steps 2 and 3 is an ambiguous outcome. Do not blindly repeat non-idempotent effects such as payments, messages, or destructive operations.

### Leases and stale workers

Use worker leases with expiry and heartbeat. A worker may update a task only while it owns the current lease/version. On lease expiry, reconcile the environment before reassigning work.

## 5. Watchdog and operational control loop

Run health supervision outside the process it supervises.

Monitor two dimensions:

### Infrastructure health
- process liveness;
- dependency connectivity;
- event source freshness;
- database availability;
- queue depth;
- worker heartbeat;
- resource saturation.

### Task progress health
- time since last meaningful progress;
- repeated identical tool calls;
- repeated failure signatures;
- task deadline;
- progress state unchanged across N checks;
- goal drift or contradictory worker reports.

Use graduated responses:
1. observe and record;
2. request a worker status report;
3. nudge or replan;
4. cancel a stuck operation;
5. restart/reassign after reconciliation;
6. escalate to a human.

Avoid restart loops. Classify errors by whether restart can plausibly fix them. Apply retry budgets, exponential backoff, jitter, and circuit breakers.

Health status should have a single authoritative writer per component or metric family. Avoid multiple modules overwriting each other's health state.

## 6. Context as a cache: Inline / Spill / Locality

Treat the model context window as a scarce working set, not as the database.

- **Inline**: include high-probability, high-value, immediately relevant information.
- **Spill**: keep large or low-probability content outside context and provide stable references.
- **Locality**: retrieve nearby information first — same task, same thread, parent message, adjacent messages, related PR, recent tool result.
- **Chain of Recall**: use IDs and references to traverse related objects on demand.
- **Bounded expansion**: every retrieval step has a token/character budget and a maximum hop count.

Start with minimal context, then expand based on references. Do not dump all memory or all task history into every model call.

For every summarized or truncated object, preserve:
- stable ID;
- source;
- timestamp;
- summary;
- original-content reference;
- truncation metadata;
- related object IDs.

## 7. Fast Path / Slow Path and guarded decision reuse

Use deterministic code for decisions whose conditions are explicit and stable. Use the LLM for semantic uncertainty.

Candidate Fast Path:
- duplicate event;
- known source identity;
- explicit allow/deny policy;
- task already completed;
- known status transition;
- deterministic routing;
- validated cached decision whose guards still hold.

Candidate Slow Path:
- ambiguous intent;
- conflicting human instructions;
- unclear task scope;
- semantic relevance;
- novel failure;
- changed environment;
- invalid or unexpected tool receipt.

A cached decision is reusable only if:
- its input fingerprint matches;
- policy version matches;
- required dependencies succeeded;
- all guards still hold;
- the action is still authorized;
- no cancellation or newer conflicting instruction exists.

Record a decision receipt:
- decision ID;
- input and policy version;
- selected action;
- guard outcomes;
- reused or recomputed;
- execution result;
- reason for fallback.

Never skip model reasoning merely because a previous action looked similar. Reuse must be guarded and auditable.

## 8. Security and authorization

Assume external event content is untrusted data, including:
- chat messages;
- GitHub issues and PR comments;
- web pages;
- repository files;
- tool output;
- memory retrieved from external sources.

Never let untrusted content:
- alter system policy;
- grant itself capabilities;
- change configured commands or credentials;
- override user or administrator authorization;
- directly mutate internal runtime state.

Enforce authorization in code, not only in prompts.

Use:
- least privilege;
- per-plugin capabilities;
- per-task scopes;
- explicit human approval for high-impact actions;
- isolated workspaces / OS users / containers where needed;
- secret separation from model-visible context;
- audit trails for decisions and side effects.

A wrapper, blocked-command list, or working directory is not automatically a security boundary. State the actual boundary and its limitations.

## 9. Plugin and extension engineering

A plugin should extend the host through declared contracts. It should not become a second hidden runtime.

### Plugin manifest

Define:
- unique plugin ID and version;
- host API compatibility range;
- entry point;
- declared capabilities;
- configuration schema;
- provided tools / commands / event handlers;
- required dependencies;
- startup and shutdown hooks.

### Extension surfaces

Prefer explicit registries:
- `ToolRegistry`
- `CommandRegistry`
- `EventHandlerRegistry`
- `CapabilityRegistry`
- `AdapterRegistry`
- `ServiceRegistry` (only where lifecycle is clear)

A plugin may:
- register tools and commands;
- subscribe to documented events;
- publish documented events;
- expose services through interfaces;
- store its own namespaced state;
- request capabilities declared in its manifest.

A plugin must not:
- mutate private host state;
- patch internal classes at runtime without a documented compatibility contract;
- create unmanaged background loops;
- bypass host authorization;
- write directly into another plugin's storage;
- assume undocumented event ordering.

### Lifecycle

Support:
- `register` / `activate`;
- `start`;
- `stop`;
- `dispose`;
- health reporting;
- config validation;
- version compatibility checks.

Every background task created by a plugin must be tracked and cancelled on shutdown.

### Event handler classes

Use two explicit categories:
- **INTERCEPT**: synchronous, bounded, may reject/transform/annotate an event before normal processing. Must have strict latency limits.
- **ASYNC**: non-blocking observer/consumer. Must not hold up the producer or interaction path.

Document ordering, error propagation, retry behavior, and whether handlers may mutate the event.

### Plugin storage

Namespace all plugin-owned state by plugin ID and schema version. Provide migrations. Do not rely on undocumented host database tables.

### Compatibility

- Pin or declare host API ranges.
- Test against the minimum and maximum supported host versions.
- Fail closed when a required capability is missing.
- Avoid importing host internals when a public API exists.

## 10. Computer-systems concepts to apply

| Computer-systems concept | Agent-system application |
|---|---|
| Control Plane / Data Plane | Router coordination vs Worker execution |
| Actor Model | Isolated workers with mailboxes and message passing |
| Event Loop | Non-blocking orchestration and event processing |
| Message Queue | Buffering, ordering, retry, and backpressure |
| CQRS | Separate commands that change state from queries that read projections |
| Event Sourcing | Immutable event history where audit/replay is valuable |
| Transactional Outbox | Atomic persistence of state change and outbound event intent |
| Idempotency | Safe handling of duplicate triggers and retries |
| Lease / Fencing Token | Prevent stale workers from writing after reassignment |
| Checkpoint / Recovery | Resume work after process failure |
| Write-Ahead Log | Persist intent before effects |
| Cache Locality | Inline hot context, spill cold context |
| Fast Path / Slow Path | Deterministic checks before expensive model reasoning |
| Speculative Execution | Start reversible low-risk work before all information arrives |
| Circuit Breaker | Stop hammering failing dependencies |
| Bulkhead | Isolate worker pools and failure domains |
| Backpressure | Bound event and task intake |
| Watchdog / Supervisor | External liveness and progress control loop |
| Capability Security | Limit what a plugin, worker, or task can do |

Do not adopt a pattern by name alone. State the failure mode it prevents and the cost it introduces.

## 11. Observability

Use structured logs, metrics, and traces.

Every operation should be correlatable through:
- event ID;
- correlation ID;
- task ID;
- session ID;
- worker ID;
- plugin ID;
- tool/effect ID.

Track:
- event intake and drop reasons;
- trigger acceptance rate;
- queue depth and wait time;
- active workers;
- task completion/failure/cancellation;
- time since last progress;
- retries and dead letters;
- model calls and token usage;
- tool latency and error rate;
- decision reuse and fallback rate;
- user notification volume.

Keep raw events, derived health snapshots, and user-facing projections distinct. Observability should read runtime state; it should not become a second competing event bus.

## 12. Testing strategy

Minimum test matrix:

| Area | Test |
|---|---|
| State machine | legal and illegal transitions |
| Idempotency | duplicate event delivered twice |
| Ordering | out-of-order source events |
| Concurrency | two workers update related tasks |
| Cancellation | cancel during tool execution |
| Recovery | crash after intent but before receipt |
| Lease | stale worker writes after lease expiry |
| Backpressure | queue full / worker pool saturated |
| Security | prompt injection in event payload |
| Authorization | unauthorized user asks for a permitted-looking action |
| Plugin lifecycle | start failure, stop, dispose, reload |
| Compatibility | minimum supported host API |
| Watchdog | dead process vs live-but-stuck task |
| Context | truncation, recall chain, token budget |
| Performance | latency, throughput, token cost |

Use deterministic fake models and fake tools for runtime tests. Keep a smaller set of end-to-end tests with real models.

## 13. Design review checklist

Before approving a design, answer:

### Boundaries
- [ ] Is the host/runtime/plugin boundary explicit?
- [ ] Is state ownership clear?
- [ ] Are trust boundaries identified?
- [ ] Are Event, Trigger, Task, Command, Effect, and Receipt distinct?

### Concurrency
- [ ] Can a slow tool block the interaction path?
- [ ] Are workers isolated?
- [ ] Are queue and concurrency limits bounded?
- [ ] Is cancellation propagated?

### Durability
- [ ] What survives process restart?
- [ ] Are pending side effects reconciled?
- [ ] Are retries idempotent?
- [ ] Can stale workers be fenced off?

### Proactivity and safety
- [ ] Is there a deterministic gate before LLM assessment?
- [ ] Is observe-only mode available?
- [ ] Is authorization separate from context subscription?
- [ ] Can untrusted content change policy or credentials?

### Operations
- [ ] Is there an external liveness check?
- [ ] Is task progress monitored separately from process health?
- [ ] Are errors classified by recoverability?
- [ ] Are logs, metrics, and traces correlated?

### Quality
- [ ] Are acceptance criteria measurable?
- [ ] Are failure-path tests present?
- [ ] Are claims about paper/repository behavior verified?
- [ ] Are prototype limitations documented?

---

# Project-specific default conventions

When applying this skill to a modular AI companion or an existing runtime, preserve these boundaries unless the repository explicitly establishes a different architecture:

- Runtime owns lifecycle and scheduling.
- Event is a fact; Task is work; Session is an interaction/context boundary; Operation is a thin execution wrapper.
- Cancellation is a first-class control event.
- Plugins integrate through Registry, Event, Command, and Capability contracts.
- Plugins do not mutate private runtime state.
- INTERCEPT handlers are bounded and synchronous; ASYNC handlers are non-blocking.
- Recorder persists trajectories; Telemetry is a derived aggregate; Observability is a read-only projection.
- Keep the first-party runtime separate from third-party reference implementations.
- Do not copy entire upstream projects into the core merely to reuse a few ideas.

For an existing monorepo, inspect its current architecture baseline before applying these conventions. Treat previous design documents as constraints to reconcile, not as permission to invent missing implementation.

# Output format for architecture work

When asked to design or implement a framework/plugin, produce:

1. **Problem and scope**
2. **Existing host/repository findings**
3. **Architecture and data flow**
4. **Core types and state machines**
5. **Public plugin/runtime interfaces**
6. **Failure, retry, cancellation, and recovery**
7. **Security and authorization**
8. **Implementation plan**
9. **Tests and acceptance criteria**
10. **Known limitations and deferred work**

Prefer small, testable increments. Avoid building a distributed platform before the use case requires one.
