# Backpressure, batching, and scheduling

Status: written 2026-11-03. **Verified** restates sources read in this pass. **Interpretation** is for Agent runtimes. The Reactive Streams specification site was reachable, but the specification body was not re-read, so this note does not describe `request(n)` semantics.

## 1. Problem statement

Proactive agents create work from the environment. Without a bound, a busy repository or chat channel creates tasks faster than Workers finish them. Latency collapses for the user who is actually waiting, and retries make the pile larger. Batching and scheduling are how to spend fixed capacity. Backpressure is how to refuse the excess.

## 2. Core concepts

**Verified.** Temporal states that Workers do not receive pushed work. They poll task queues. A Worker that is down or saturated stops polling, and tasks wait. The same page says task queues also isolate work by service, fleet, or hardware, and that priority and fairness features can run urgent work ahead of bulk work so one tenant does not monopolize a queue. This note does not re-read the priority feature page; only the existence of that routing role is taken from the understanding document.

**Verified.** NATS JetStream pull consumers, in the documentation's own examples, fetch a bounded batch (`--count`, `max_messages`, `Fetch(3)`). The application asks for N messages. That is a pull bound, not an unbounded push.

**Verified.** The SRE chapter defines saturation as how full a service is, notes that systems degrade before 100% utilization, and says latency increases are often a leading indicator. It warns that means hide tails.

**Verified.** Ray's OSDI paper describes `ray.wait` as waiting for the first k results rather than all, specifically to handle tasks with different durations. That is a scheduling primitive for stragglers, not a license to launch unlimited work. The paper's million-task experimental claims were not re-used as targets for an agent runtime.

## 3. Design principles

- Every queue has a maximum length. Every user and source has a quota.
- Overload behavior is chosen: queue, reject, coalesce, or degrade. Unbounded creation is not an option.
- Interaction work outranks background and proactive work.
- Batch only cold or identical work. Do not batch extra context into the user-facing model call.
- Retries count against the same budget. A retry is new load.
- Pull or credit-based intake is easier to bound than a push bus that must be slowed from downstream.

## 4. Internal mechanisms

Backpressure points:

1. Ingress: drop or coalesce events before they become tasks. Record the drop reason.
2. Task queue: reject when depth or age exceeds the limit.
3. Worker pool: do not start an Operation without a slot.
4. Tool process: cgroup or a local semaphore.
5. Downstream API: stop calling when the circuit is open.

Batching that helps:

- Coalesce many file-change events into one task per path per window.
- Fetch N log messages at a time, as JetStream's examples do, and ack each.
- Group WAL flushes when several local commits can share a disk flush. PostgreSQL documents that one WAL flush may commit many small transactions. Do not group external side effects that have different idempotency keys just to save a flush.

Scheduling:

- Priority classes: user interaction, already-running task progress, proactive suggestions.
- Aging so low priority cannot starve forever.
- Separate pools (bulkheads) so a stuck tool pool does not consume interaction slots.

**Interpretation.** `ray.wait` style "first k" is useful when several retrievals are speculative and the agent can proceed with a subset. It is harmful if the ignored stragglers keep consuming quota. Cancel them.

## 5. Architecture and execution flow

```text
Event source
  prefilter and coalesce
  quota check
    reject -> drop metric + audit
    accept -> bounded queue
Worker slots
  pull next task by priority
  one Operation
  completion event
Watchdog
  if wait time or saturation high
    shed proactive work first
```

## 6. Benefits

- The interaction path stays available when background load spikes.
- Pull keeps a dead Worker from receiving a push it cannot store.
- Coalescing cuts duplicate reasoning on noisy sources.
- Separate pools contain failure.

## 7. Trade-offs

- Rejecting work loses events unless they remain in the source log for a later pull. JetStream retention makes that possible. An in-memory bus does not.
- Priority can starve. Aging or a fairness queue is required if proactive work matters.
- Batching adds delay. A window that is fine for indexing is wrong for a chat reply.
- Global quotas are harder than per-session quotas and are unnecessary on a single-user device.

## 8. Failure modes

- Unbounded mailbox on the Router.
- `acks_late` consumers prefetch a huge batch and then crash, redelivering all of it. Celery's late ack increases redelivery; this note did not re-read Celery's prefetch multiplier, so no numeric prefetch default is stated.
- Retry storms on a failing model API.
- Batching distinct users' events into one model call and crossing a trust boundary.
- Using mean queue delay as the only saturation signal.

## 9. Mature implementations

- Temporal worker polling and queue isolation.
- JetStream pull fetch bounds.
- Ray `ray.wait` for heterogeneous completion.
- PostgreSQL group commit as a batching example that does not apply blindly to external effects.
- SRE saturation and tail latency as the signals that should trip shedding.

## 10. Agent Runtime mapping

- **Trigger** pipeline already has a deterministic gate and rate limits. Those are the first backpressure stage.
- **Task** queue bounds are host policy, not plugin policy.
- **Worker** count is a bulkhead.
- **Operation** concurrency is the slot. A plugin ASYNC handler must not bypass the slot by running work inline.
- **Watchdog** sheds load and cancels stale work. It does not only restart.
- **Telemetry** exports queue depth, wait, reject reason, and token rate.

Conflict: a framework that pushes messages as fast as the publisher can send will violate bounded intake unless the host adds a pull or a reject path in front of it. Do not disable that path to match the framework's benchmark.

## 11. When to use

Add explicit bounds before the second concurrent task source exists. Add priority when proactive work shares a pool with interactive work. Add coalescing when one source is noisy.

## 12. When not to use

Do not add a distributed fair-queuing system for a local prototype. Do not batch user-visible replies to improve a throughput chart. Do not copy Reactive Streams API details from this note; the spec was not re-read.

## 13. Original references

| Title | Author / org | URL | Date | Type | Concepts | Why it matters |
|---|---|---|---|---|---|---|
| Understanding Temporal | Temporal | https://docs.temporal.io/evaluate/understanding-temporal | retrieved 2026-11-03 | Official doc | Workers poll, saturated workers stop pulling, queue isolation | Pull as backpressure |
| JetStream | NATS | https://docs.nats.io/nats-concepts/jetstream | retrieved 2026-11-03 | Official doc | Bounded pull fetch, explicit ack | A concrete batch bound |
| Ray: A Distributed Framework for Emerging AI Applications | Moritz et al. | https://www.usenix.org/system/files/osdi18-moritz.pdf | OSDI, October 2018 | Paper | `ray.wait` for first k of heterogeneous tasks | Straggler-aware waiting. Evaluation numbers not reused |
| Monitoring Distributed Systems | Google SRE | https://sre.google/sre-book/monitoring-distributed-systems/ | retrieved 2026-11-03 | Book chapter | Saturation, tail latency | Signals that should trigger shedding |
| Write-Ahead Logging (WAL) | PostgreSQL | https://www.postgresql.org/docs/current/wal-intro.html | Documentation 18; retrieved 2026-11-03 | Official doc | One flush, many commits | Batching that is safe only for local commits |

Unresolved: https://www.reactive-streams.org/ returned HTTP 200. The specification text was not re-read.
