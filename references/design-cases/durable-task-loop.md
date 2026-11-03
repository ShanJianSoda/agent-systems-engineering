# Design case: a task that survives restart

Status: written 2026-11-03. This is an engineering comparison, not a vendor design and not a benchmark.

## Problem

A coding agent starts a test run, the host process restarts, and the user still needs the real outcome: running, finished, failed, or unknown. The run may already have written files.

## Alternatives actually compared

1. **In-memory task plus Celery-style early ack.** Celery's default acknowledges before the task returns. A crash after ack loses the message unless something else stored the intent. This fails the requirement.
2. **Ray actor with default fault tolerance.** Ray documents `max_restarts=0` and at-most-once actor tasks unless options change. Restart does not reload application state. This fails unless the application adds checkpoints and accepts duplicate execution when retries are enabled.
3. **Local task ledger plus WAL order plus lease.** Persist intent, run the Operation, persist the receipt, fence the Worker. This matches the skill and stays inside one host.
4. **Temporal workflow history plus activities.** History replay restores control state. The test command is an activity with a retry policy and heartbeats. This is justified only when executions outlive one host and operators will run the Temporal service or Temporal Cloud.

## Recommendation

For a single-host agent, choose alternative 3. Add alternative 4 only when the ADR shows multi-host recovery, long timers, or an existing Temporal deployment. Do not start with Ray defaults and call them durable.

## What must be true

- Task row or log contains status, version, idempotency key, and receipt.
- The test Operation is cancelable and process-group scoped.
- Restart test crashes after intent and before receipt, and the runtime does not launch a second destructive command blindly.
- A paused Worker with an old version cannot write the result.

## Conflict with framework habits

Temporal workflow code must be deterministic. The decision to run tests can be replayed only if the model choice was recorded. The shell command stays an Operation, not inline workflow I/O. See [durable execution](../distributed-systems/durable-execution.md) and [Temporal](../frameworks/temporal.md).

## Sources

Verified behavior used above comes from:

- https://docs.celeryq.dev/en/stable/userguide/tasks.html
- https://docs.ray.io/en/latest/ray-core/fault_tolerance/actors.html
- https://docs.temporal.io/evaluate/understanding-temporal
- https://docs.temporal.io/workflow-definition
