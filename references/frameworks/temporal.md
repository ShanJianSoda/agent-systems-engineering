# Temporal

Status: written 2026-11-03 from the two official pages cited below. Not a full reading of Temporal's architecture manual.

## Problem it actually solves

Temporal keeps a coded control flow running to completion across Worker process loss, for durations from seconds to a long time, and supervises external calls with retries, timeouts, and heartbeats. The understanding page presents this as a replacement for hand-rolled state tables, checkpoint columns, reconciliation jobs, and per-call retry logic.

## Core abstractions

**Verified.**

- **Workflow**: business process as ordinary control flow. The definition must be deterministic because it is rerun to rebuild state.
- **Activity**: work that touches the outside world. Retries, timeouts, and optional heartbeats come from configuration. A failed activity retries with backoff. If its Worker dies, it is rescheduled.
- **Worker**: user process that polls task queues and runs workflow and activity code. The Temporal Service does not run that code.
- **Task queue**: named queue. Workers pull. A down or saturated Worker stops polling and tasks wait.
- **Event History**: ordered durable log. Workflow API calls become commands; the service appends events.
- **Temporal Service**: Frontend, History, Matching, and an internal Worker Service, with Cassandra, MySQL, PostgreSQL, or SQLite persistence. Elasticsearch is optional for advanced visibility.

The page also describes standalone activities, signals, queries, updates, child workflows, schedules, pause, and worker versioning. Those were not each re-read on their own pages.

## Execution model

Workers poll. The service matches tasks to pollers. Replay fetches history, reruns workflow code, and continues. A command that does not match the next history event is a nondeterminism error. The workflow definition page lists commands that must not be reordered, including timers, activities, child workflows, signals, completion, and patch markers. Local time and random branches inside the workflow are intrinsic nondeterminism. SDK APIs exist to record time and randomness into history; this note did not re-read each SDK page.

Worker failure is not handled inside the workflow function. The platform resumes the execution if a Worker is available. The execution fails if the code throws, not because a host rebooted.

## Persistence model

History is the durable log, stored by the History Service. Visibility storage is separate. This note did not re-read shard or multi-region behavior.

## Failure handling

Activities have retry policies and heartbeats. Workflow code is replayed. Cancellation and termination exist as CLI operations on the understanding page. Pause holds an execution without dropping state. Versioning is required so running executions stay on a compatible command sequence. Two strategies are named: Worker Versioning and patching. Operational details of those strategies were not re-read.

## Limitations

- Determinism rules out inline LLM calls, clocks, and I/O. Temporal's own tip says to put LLM and API calls in activities.
- History grows with every step. Continue-as-new is named on the workflow page as an ending mode; its operational limits were not re-read.
- You operate a service and a database, or pay for Temporal Cloud. That is a real operational cost for a single-user desktop agent.
- Activity execution is supervised retry, not end-to-end exactly-once across arbitrary APIs. Do not import that phrase.
- The service schedules work; it does not understand agent policy or capability grants.

## Agent Runtime mapping

| Temporal | This skill | Conflict if copied blindly |
|---|---|---|
| Workflow | Deterministic task control state | Must not contain the model call |
| Activity | Operation | Keep it a thin wrapper with a receipt |
| Worker | Execution plane | Not the Router |
| Task queue | Bounded work pull | Still needs a local quota and priority policy |
| History | Recorder plus task log | Not a substitute for authorization checks on effects |
| Signal | Control or input event | Cancel remains a first-class persisted control event |

Use Temporal as a substrate when the ADR shows multi-host durable control flow. Keep Router, Session, Plugin, and Capability as product concepts.

## References

| Title | Author / org | URL | Date | Type |
|---|---|---|---|---|
| Understanding Temporal | Temporal | https://docs.temporal.io/evaluate/understanding-temporal | retrieved 2026-11-03 | Official doc |
| Temporal Workflow Definition | Temporal | https://docs.temporal.io/workflow-definition | retrieved 2026-11-03 | Official doc |
