# Design case: proactive work without a runaway loop

Status: written 2026-11-03. Engineering comparison for the skill's trigger pipeline.

## Problem

Repository and chat events should be able to start work, but the default deployment must not execute inferred actions, and a noisy source must not fill the queue.

## Alternatives actually compared

1. **Edge-triggered handler only.** Each event starts a task immediately. A dropped event is lost. A burst has no bound. This fights both backpressure and the observe-only default.
2. **Kubernetes-style reconciler only.** A loop compares desired subscriptions to observed files or issues. It heals missed events, which the controller document describes as continuous movement toward desired state. It is a poor fit for low-latency chat unless the period is short, and a short period is a polling load.
3. **Hybrid used by this skill.** Deterministic prefilter and rate limit, observe versus auto policy, durable event ID dedupe, bounded queue, and a slow reconciler that repairs missed source cursors.

## Recommendation

Use alternative 3. Keep `auto` off until policy and authorization are tested. The reconciler repairs cursors and stalled tasks. It does not replace the event path.

## What must be true

- Default mode is observe or off, not auto.
- Duplicate event IDs create one task.
- Queue saturation rejects or coalesces and records why.
- Subscription to a thread does not grant tools.
- Semantic model assessment runs only after the deterministic gate, and only when the gate cannot decide.

## Conflict with framework habits

A controller that writes fixes on every loop will repeat side effects. The agent reconciler may only enqueue a trigger or repair a cursor. Creating user-visible actions stays behind the policy gate. See [supervision](../architecture-patterns/supervision-watchdog-reconciliation.md) and [backpressure](../architecture-patterns/backpressure-batching-scheduling.md).

## Sources

- https://kubernetes.io/docs/concepts/architecture/controller/
- https://docs.nats.io/nats-concepts/jetstream
- https://docs.temporal.io/evaluate/understanding-temporal
