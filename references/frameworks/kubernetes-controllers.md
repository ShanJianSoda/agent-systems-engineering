# Kubernetes controllers

Status: written 2026-11-03 from the official controllers concept page. Leader election, informers, and work-queue implementation pages were not re-read.

## Problem it actually solves

Kubernetes needs many independent loops that keep cluster objects moving toward the spec users declared, even though objects change continuously and controllers themselves crash. The Job controller's problem is "run this work to completion by ensuring the right Pods exist," not "execute the user's process inside the controller."

## Core abstractions

**Verified.**

- **Spec**: desired state on an object.
- **Controller**: a non-terminating loop for at least one resource type.
- **API server**: the place controllers read and write. The Job controller creates or deletes Pods through the API. It does not run containers.
- **Current state**: what the controller observes, including status it writes back, such as marking a Job finished.
- **Small controllers**: one loop should not be a monolith. Controllers can fail and another part of the control plane takes over built-in controllers. The page states that several controllers may touch the same kind of object and must distinguish ownership, for example with labels, so a Job controller does not delete a Deployment's Pods.

Some controllers act outside the cluster and then report status back. The page's example is scaling nodes.

## Execution model

Read objects, compare with desired state, make one change or request one change, update status, repeat. The page says the cluster may never reach a stable state, and that this is acceptable while controllers can still make useful changes. This is level-triggered reconciliation.

## Persistence model

Desired and observed state live in the API server's objects. This note does not describe etcd or watch bookmarks; those pages were not read.

## Failure handling

A controller crash is tolerated if another instance continues and actions are safe to repeat. Built-in controllers run in a resilient control plane. The page does not claim that external side effects are exactly-once. Direct controllers that call cloud APIs must handle their own ambiguity. That limit is an interpretation of "communicate directly with an external system," not a quoted retry algorithm.

## Limitations

- Kubernetes is a cluster control plane. Adopting it to supervise one local agent process is an operational mismatch.
- Reconciliation period and event delivery latency are not specified on this page. Do not invent them.
- The page does not document leasing. Do not cite a lease API from this note.
- Desired state is declarative. Chatty edge events and user conversation are not the model.

## Agent Runtime mapping

| Kubernetes | This skill |
|---|---|
| Spec | Desired task status, policy, cancel |
| Status | Observed Worker, receipt, progress |
| Controller | Watchdog or reconciler, outside the Worker |
| API server | Task store, the single writer of record |
| Labels / owner | Task owner and plugin ID, so loops do not fight |

Conflict: a reconciler that sends a user message every loop violates idempotency. The agent loop may reconcile process and lease state continuously. User-visible effects stay behind receipts and policy. See [supervision](../architecture-patterns/supervision-watchdog-reconciliation.md) and [proactive case](../design-cases/proactive-trigger-pipeline.md).

## References

| Title | Author / org | URL | Date | Type |
|---|---|---|---|---|
| Controllers | Kubernetes | https://kubernetes.io/docs/concepts/architecture/controller/ | retrieved 2026-11-03 | Official doc |
