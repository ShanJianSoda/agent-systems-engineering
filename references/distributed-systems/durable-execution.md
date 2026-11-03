# Durable execution, checkpoints, and recovery

Status: written 2026-11-03. **Verified** restates sources read in this pass. **Interpretation** is for Agent runtimes.

## 1. Problem statement

An agent task can last longer than the process that started it. After a crash, the runtime must know what was intended, what was sent to the outside world, and where it is legal to continue. Checkpointing stores a position. Durable execution stores a history that can be replayed into that position. They fail differently.

## 2. Core concepts

**Verified.** Temporal's "Understanding Temporal" page defines Durable Execution as the property that a Workflow, once started, runs to completion even if that takes a long time. If the process crashes, the service hands the work to another process, which rebuilds execution state and resumes. The page states that the Temporal Service does not run user code. Workers poll task queues and run the code. The Event History is the ordered record that makes replay possible. Workflow code must be deterministic because it is rerun to rebuild state. Anything that talks to the outside world belongs in an Activity.

**Verified.** The Workflow Definition page states that replay compares commands emitted by the code with the existing Event History, and that a mismatch is a nondeterminism error. Sources of mismatch include code changes on a running execution and intrinsic nondeterminism such as local time or random branching. Activities, which include LLM or API calls in Temporal's own examples of what not to inline, execute outside the replay path. The page also states that Worker process failure is not handled inside the workflow definition; the platform recovers execution state. An execution fails because the code throws, not because a Worker host reboots.

**Verified.** Akka Persistence states that only persisted events are stored in the event-sourced style, not the actor's full state, although snapshots exist. Recovery replays events. Events are applied after they have been persisted. Commands are validated and are not what gets replayed. A separate durable-state behavior persists the latest state instead of events.

**Verified.** Ray's actor fault-tolerance page states that `max_restarts` defaults to 0. Restart reruns the constructor and does not automatically restore application state. The application must checkpoint. By default actor tasks are at-most-once (`max_task_retries=0`). At-least-once retries can execute a method twice, once on the failed actor and once on the restarted actor. For critical state, Ray says the application is responsible for recovery.

## 3. Design principles

- Persist intent, assignment, checkpoint, pending effects, and receipts. The skill already lists this set. Do not shrink it because a framework uses one history log.
- Separate deterministic control state from nondeterministic work. Model calls, tool I/O, and clocks that are not replay-safe do not belong inside a replayed function.
- A checkpoint names a log position. Restoring a checkpoint without the log tail loses later intents.
- Process restart is not task success and not task failure. Read the durable status.
- Version the control code. Temporal documents that reordering activities or timers breaks replay of running executions.

## 4. Internal mechanisms

Three recovery styles, all seen in the sources:

| Style | What is stored | How resume works | Duplicate risk |
|---|---|---|---|
| Event history replay (Temporal workflow, Akka event sourced actor) | Ordered commands or events | Rerun deterministic code against the log until the tail | External calls must sit outside replay |
| Latest-state checkpoint (Akka durable state, Ray manual checkpoint) | Last state blob | Load blob, continue | Work after the blob and before the crash is ambiguous |
| At-most-once task (Ray default) | Not the application state | Report the error | Loss, or uncertainty if the task finished as the actor died |

**Interpretation.** An agent runtime should use history for the task ledger and checkpoints for large context summaries. Replaying an LLM call to "rebuild" a decision will not reproduce the decision. Store the decision receipt and replay that.

Temporal's split maps cleanly onto this skill if names stay local:

- Workflow history ≈ durable task and session transitions that are deterministic given recorded inputs.
- Activity ≈ Operation that performs a tool or model call.
- Worker poll ≈ host execution plane, not the product's Router.

Do not rename the product's Worker to Temporal's Worker in code. They are different roles. Temporal's Worker runs both workflow tasks and activities. This skill's Worker executes a task and does not own global coordination.

## 5. Architecture and execution flow

```text
Task accepted
  append transition
  checkpoint context refs
loop
  deterministic Router step reads state
  schedules Operation (model or tool)
  Operation result appended as receipt
  crash
restart
  replay transitions
  load latest checkpoint
  if receipt missing: reconcile, do not rerun blindly
  continue
```

Code change: new branches in the deterministic step require a version marker, analogous to Temporal's patching or Worker versioning. Shipping a new Router that emits a different command sequence for an in-flight task is a migration, not a restart.

## 6. Benefits

- Long tasks survive deploys and crashes if history is complete.
- Deterministic replay makes control bugs reproducible from the log.
- Checkpoints shorten replay, which Akka documents as the reason snapshots exist.
- The outside world is called from a place that is allowed to retry under an explicit policy.

## 7. Trade-offs

- Determinism is a severe constraint. Temporal lists timers, activity schedules, child workflows, signals, and completion as commands that must not be reordered once an execution exists.
- History storage grows with every model step. Snapshots and continuing-as-new, mentioned by Temporal as an ending mode, are operational necessities. This note did not re-read the continue-as-new page.
- Replay latency delays recovery if the history is huge and the checkpoint is rare.
- Ray-style constructor replay loses memory unless the constructor loads a checkpoint. That is easy to miss because restart "succeeds."

## 8. Failure modes

- Putting `datetime.now()` or an LLM call in the replay path. Temporal documents this class of bug as intrinsic nondeterminism.
- Changing Operation order in a deploy while old tasks are open.
- Checkpointing a summary and deleting the tool receipt that the summary came from.
- Equating "actor restarted" with "task recovered." Ray says state is recreated by the constructor only.
- Two Workers replay the same history and both perform an activity because fencing was omitted.

## 9. Mature implementations

- Temporal Event History, deterministic workflows, and activities. [Framework note](../frameworks/temporal.md).
- Akka event sourcing versus durable state. [Framework note](../frameworks/akka.md).
- Ray actor restarts and application-level checkpoints. [Framework note](../frameworks/ray.md).
- Database WAL redo. [WAL note](../computer-systems/wal-snapshots-transactions.md).

## 10. Agent Runtime mapping

- **Durable state** is the task ledger plus receipts, not the model context window.
- **Session** survives as identity and references. Rebuilding a session does not mean replaying every token into a prompt.
- **Operation** is the non-deterministic unit. It stays thin: inputs, idempotency key, cancel flag, receipt.
- **Recorder** appends the trajectory that replay and audit use.
- **Watchdog** detects a live process with a stalled checkpoint, which history alone will not mark as stuck if the process is still heartbeating.
- **Telemetry** is derived and may be rebuilt. It is not the recovery log.

Conflict: importing Temporal as the runtime would move lifecycle ownership into Temporal's history service and workers. That can be a legitimate substrate, but the product Router, Task, and Plugin contracts must remain the architecture. Do not let Workflow code mutate plugin policy or skip authorization because the replay already passed that line once. Authorization has to be rechecked on effects, or the recorded grant must be an immutable input to replay.

## 11. When to use

Use durable history when tasks must survive restart and the control decisions can be recorded. Use checkpoints when the replay set is large (context summaries, repository indexes) and can be rebuilt or reloaded.

## 12. When not to use

Do not wrap a single synchronous tool call in a workflow engine. Do not replay nondeterministic model code and call the result deterministic. Do not adopt Ray's default actor settings as a durable execution design; the documentation says the opposite.

## 13. Original references

| Title | Author / org | URL | Date | Type | Concepts | Why it matters |
|---|---|---|---|---|---|---|
| Understanding Temporal | Temporal | https://docs.temporal.io/evaluate/understanding-temporal | retrieved 2026-11-03 | Official doc | Durable execution, event history, workers poll, activities | Primary vocabulary for history replay |
| Temporal Workflow Definition | Temporal | https://docs.temporal.io/workflow-definition | retrieved 2026-11-03 | Official doc | Determinism, command match, versioning, worker failure | What actually breaks replay |
| Event Sourcing | Akka | https://doc.akka.io/libraries/akka-core/current/typed/persistence.html | Akka 2.10.24 cited on page; retrieved 2026-11-03 | Official doc | Persist events, replay, snapshots, durable state | Two persistence styles in one actor toolkit |
| Actor Fault Tolerance | Ray | https://docs.ray.io/en/latest/ray-core/fault_tolerance/actors.html | retrieved 2026-11-03 | Official doc | Default no restart, at-most-once, manual checkpoint | A contrast: process actors are not durable by default |
