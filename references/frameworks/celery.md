# Celery

Status: written 2026-11-03 from the stable tasks guide. Broker differences between RabbitMQ and Redis, canvas workflows, and the prefetch multiplier were not re-read.

## Problem it actually solves

Celery distributes function calls to worker processes through a message broker. It is a task queue for units of work, not a durable conversation runtime and not a workflow history.

## Core abstractions

**Verified** from the tasks page:

- A task message stays on the queue until a worker acknowledges it.
- Workers may reserve multiple messages.
- The documented recommendation is that task functions be idempotent, because the worker cannot always tell whether a previous attempt finished.
- `acks_late`: when true, ack happens after the task returns. A crash in the middle can run the task more than once. The page states the default is to acknowledge before execution (`acks_late` false), overridable by `task_acks_late`.
- If the child process is terminated by `sys.exit()` or by a signal, the worker still acknowledges the message even when `acks_late` is set.
- `Reject` uses AMQP `basic_reject` and the page says it has no effect unless `acks_late` is enabled.
- `Ignore` records no task state but still acknowledges the message.

Retry policy, result backends, and chords are outside this reading.

## Execution model

A client enqueues a message. A worker process reserves it, runs the Python function, and acks according to the settings above. Concurrency is a pool of processes or threads depending on configuration that was not re-read. Do not state a default concurrency number from this note.

## Persistence model

Durability is the broker's durability plus whatever the task writes itself. Celery does not, on the page read, store a replayable deterministic history. Early ack means the broker may forget the message before the function finishes.

## Failure handling

Late ack trades loss for duplicates. Early ack trades duplicates for loss. Signal death can ack even in the late mode, which drops the message relative to the application's try/except. Idempotency is the documented mitigation, not a broker transaction with the user's database.

## Limitations

- No workflow determinism or event history in the material that was read.
- Signal and `sys.exit` interaction is a trap for cancellation: killing the child can look like success to the broker.
- Not a lease or fencing system.
- A result backend, if you add one later, is still not a receipt log unless you write the intent before the call.

## Agent Runtime mapping

| Celery | This skill |
|---|---|
| Task message | An Operation dispatch, after the runtime has already persisted intent |
| Early ack | Unsafe for the only copy of the work |
| Late ack | Requires idempotent Operations and receipts |
| Worker process | Execution plane under the host supervisor |
| Ignore / Reject | Explicit drop or dead-letter, recorded by the runtime |

Conflict: using Celery as the system of record makes the broker the task ledger. The skill's ledger is the durable task state. Celery can carry the dispatch. It should not be the only place that knows the task exists. See [delivery](../distributed-systems/delivery-consistency-idempotency.md) and [durable task case](../design-cases/durable-task-loop.md).

## References

| Title | Author / org | URL | Date | Type |
|---|---|---|---|---|
| Tasks | Celery Project | https://docs.celeryq.dev/en/stable/userguide/tasks.html | stable documentation; retrieved 2026-11-03; page shows no publication date | Official doc |
