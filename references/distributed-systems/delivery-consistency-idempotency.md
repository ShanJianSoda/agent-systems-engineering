# Delivery, consistency, and idempotency

Status: written 2026-11-03. **Verified** restates sources read in this pass. **Interpretation** is for Agent runtimes.

## 1. Problem statement

Events and task assignments are delivered over process restarts, queues, and networks that retry. A retry can run a tool twice. A lost acknowledgement can run it again after it already succeeded. Agent runtimes that promise "exactly once" across chat providers, git hosts, and payment APIs are promising something those systems do not jointly provide.

## 2. Core concepts

**Verified.** NATS documentation states that core NATS delivers only to subscribers connected at publication time, at most once, with no replay. JetStream adds a persistence layer and at-least-once delivery: a consumer tracks progress, and if a message is not acknowledged in time the server redelivers it.

**Verified.** Celery's task documentation says a task message stays on the queue until a worker acknowledges it. It says task functions should ideally be idempotent, because the worker cannot always know whether a previous execution finished. With `acks_late`, the worker acknowledges after the task returns, and a crash in the middle can execute the task more than once. The default is to acknowledge before execution. The same page says that if the child process is terminated by `sys.exit()` or a signal, the worker still acknowledges even when `acks_late` is set.

**Verified.** Chris Richardson's transactional outbox pattern states that the message relay can publish more than once, for example if it crashes after publish and before recording that fact. Consumers must be idempotent, often by remembering message IDs. The pattern also says this is acceptable because brokers already often deliver more than once.

**Verified.** Martin Fowler's CQRS article (14 July 2011) says that separate command and query models raise the question of how hard to keep them consistent, and that this raises the likelihood of eventual consistency. He also says CQRS is a minority tool and risky when applied to the whole system.

**Interpretation.** The skill's rule stands: assume at-least-once delivery unless the transport proves otherwise, and aim for idempotent effects plus reconciliation. Do not describe that combination as exactly-once execution across arbitrary external systems.

## 3. Design principles

- Stable event IDs and idempotency keys are part of the contract, not an optimization.
- Deduplicate inside a defined window. A window is not a proof that a duplicate will never arrive later.
- Ordering is per stream or per aggregate, not global, unless a single log is the source of truth.
- A query projection may lag the command side. User-visible status must say so when the lag matters.
- Consistency choices are local. One task ledger can be linearizable while a dashboard is eventual.

## 4. Internal mechanisms

At-least-once consumer:

1. Read a message with its ID and sequence.
2. If the ID is already in the receipt store, acknowledge and stop.
3. Otherwise persist the intent to process it, with the ID.
4. Perform the effect if it is idempotent or not yet done.
5. Persist the receipt.
6. Acknowledge.

Step 4 is the dangerous one. If the effect cannot be deduplicated by the downstream system, step 3 must record "unknown" after a timeout, and a reconciler must query the downstream system. Retrying immediately is how duplicate messages are sent.

**Verified.** JetStream assigns a stream sequence and a consumer advances only on acknowledgement. **Interpretation.** That sequence is a cursor for one consumer, not a global order across all agent event types. Independent consumers on the same stream, which JetStream allows, each have their own position. That is useful for a Recorder and a Router that both read events. It is harmful if both of them also perform the same side effect.

Celery's early acknowledgement avoids duplicates by risking loss: a crash after ack and before finish drops the task. Late acknowledgement avoids loss by risking duplicates. Those are the only two options the cited page describes. Neither is exactly-once.

## 5. Architecture and execution flow

```text
Source -> durable log or queue
  consumer (Router)
    dedupe by event ID
    policy decision
    Task transition + outbox record in one local commit
  relay publishes task.assigned
  Worker dedupes by task version
  external effect with idempotency key
  receipt
```

See [outbox, event sourcing, and CQRS](../architecture-patterns/event-driven-outbox-cqrs.md) for the atomic publish problem.

## 6. Benefits

- Retries become safe for effects that have keys or natural idempotency.
- At-least-once logs survive a consumer crash, which core NATS without JetStream does not.
- Separate read models can serve status without contending with the task ledger.
- Ambiguous receipts are visible instead of being guessed.

## 7. Trade-offs

- Idempotency storage has to live at least as long as the retry horizon.
- Exactly-once *inside* one log (a broker publishing to itself) does not extend to email, chat, or shell commands.
- Eventual consistency means the UI can show a stale "running" after completion. The projection needs a version.
- Stronger consistency, such as a single-writer task row with compare-and-swap, reduces concurrency.

## 8. Failure modes

- Deduping only in process memory. Restart forgets IDs and repeats effects.
- Acknowledging before the receipt is durable. Celery's default ack is this shape.
- Two consumers in the same group both performing effects because membership changed.
- Treating HTTP 200 from a proxy as proof the downstream action committed.
- Using a global lock to "get consistency" without a fencing token. See [leases](leases-fencing-retries.md).

## 9. Mature implementations

- NATS JetStream streams and explicitly acknowledged consumers.
- Celery's acknowledge-early versus `acks_late` choice.
- Transactional outbox as described by Richardson.
- Temporal activity retries, which are at-least-once execution of activities supervised by the service. See [Temporal](../frameworks/temporal.md).

## 10. Agent Runtime mapping

- **Event** delivery is at-least-once. Handlers must tolerate duplicates.
- **Trigger** evaluation must be idempotent so a redelivered message does not open two tasks. Use the event ID and a dedupe record.
- **Task** creation uses a unique key derived from the source event when one task per event is the policy.
- **Effect** and **Receipt** are separate records. The receipt is the consistency point for "we know what happened."
- **Session** ordering is the order of that session's log, not arrival order at the network interface.
- **Observability** may show lag between the task ledger and a status projection. It must not write either one.

Conflict: a framework advertisement of exactly-once processing does not relax the skill's rule. Map the claim to a specific boundary (one database, one log) and keep external effects idempotent.

## 11. When to use

Use durable at-least-once delivery plus idempotency keys as soon as a task can outlive one HTTP request or one process.

## 12. When not to use

Do not add a broker, an outbox, and a projection for a single-user tool that can rerun safely from a local task file. Do not block the interaction path on a strongly consistent read of a distant projection when a local task row already has the authoritative status.

## 13. Original references

| Title | Author / org | URL | Date | Type | Concepts | Why it matters |
|---|---|---|---|---|---|---|
| JetStream | NATS | https://docs.nats.io/nats-concepts/jetstream | retrieved 2026-11-03 | Official doc | At-most-once core NATS, at-least-once JetStream, ack, redelivery | Concrete delivery semantics |
| Tasks | Celery | https://docs.celeryq.dev/en/stable/userguide/tasks.html | stable docs; retrieved 2026-11-03 | Official doc | Ack timing, idempotency, signal termination | Shows loss versus duplicate, not exactly-once |
| Pattern: Transactional outbox | Chris Richardson, microservices.io | https://microservices.io/patterns/data/transactional-outbox.html | retrieved 2026-11-03; page does not show a publication date | Pattern catalog | Atomic state change and publish, duplicate relay | Why consumers stay idempotent |
| CQRS | Martin Fowler | https://martinfowler.com/bliki/CQRS.html | 14 July 2011 | Article | Separate models, eventual consistency, caution | Author's recommendation against using CQRS everywhere |

Reading limit: Lamport's "Time, Clocks, and the Ordering of Events" (1978) and the Gilbert & Lynch CAP paper were not re-read. This note does not state their theorems. Helland's "Life beyond Distributed Transactions" (CIDR 2007) URL responded, but the body was not re-read; it is listed only as an unread primary source in the reference map's gaps.
