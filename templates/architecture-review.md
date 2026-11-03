# Architecture review

Use this before implementing a new runtime surface or a plugin that runs background work. Answer in the host repository. Do not treat unchecked boxes as approval.

Problem and non-goals:

Host or repository facts that constrain the design:

## Boundaries

- [ ] Host, runtime, and plugin boundaries are explicit.
- [ ] State ownership is named.
- [ ] Trust boundaries are named, including untrusted event content.
- [ ] Event, Trigger, Task, Command, Effect, and Receipt stay distinct.
- [ ] Session is an interaction boundary. Operation is a thin wrapper.

## Concurrency and intake

- [ ] A slow tool cannot block the interaction path.
- [ ] Workers are isolated.
- [ ] Queue length, worker slots, and per-source quotas are bounded.
- [ ] Overload behavior is reject, queue, coalesce, or degrade.
- [ ] Cancellation is persisted and propagated to tools and child processes.

## Durability

- [ ] Intent is durable before external effects.
- [ ] Missing receipts are reconciled, not blindly retried when the effect is not idempotent.
- [ ] A Worker can write only with the current lease version or fencing token.
- [ ] Process exit is not treated as task completion.

## Proactivity and authorization

- [ ] A deterministic gate runs before any model assessment.
- [ ] Observe-only mode exists, and auto is not the default.
- [ ] Subscription is not authorization.
- [ ] Untrusted content cannot change policy, capabilities, or credentials.

## Plugins

- [ ] Integration is through Registry, Event, Command, and Capability.
- [ ] Plugins cannot mutate private runtime state.
- [ ] INTERCEPT handlers are synchronous and bounded.
- [ ] ASYNC handlers do not block producers.
- [ ] Third-party frameworks stay inside the plugin boundary.

## Operations

- [ ] Liveness is checked from outside the Worker.
- [ ] Progress is monitored separately from process health.
- [ ] Errors are classified by whether retry can help.
- [ ] Logs, metrics, and traces share IDs.
- [ ] Observability does not write the task ledger.

## Evidence

References read (paths or URLs):

Alternatives compared:

Conflicts with skill constraints:

Tests that must exist before calling this done:

Known prototype limits:
