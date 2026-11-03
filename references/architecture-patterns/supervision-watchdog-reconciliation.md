# Supervision, watchdogs, and reconciliation

Status: written 2026-11-03. **Verified** restates sources read in this pass. **Interpretation** is for Agent runtimes.

## 1. Problem statement

A Worker process can be alive and still make no progress. A process can be dead while its external effect succeeded. A controller can crash and must resume from desired state rather than from a lost in-memory plan. Supervision, watchdogs, and reconciliation cover these three different failures.

## 2. Core concepts

**Verified.** OTP design principles define a supervision tree: workers do the work, supervisors monitor workers and restart them. The supervisor manual documents restart strategies: `one_for_one` (only the terminated child), `one_for_all` (all children), `rest_for_one` (the terminated child and those started after it), and `simple_one_for_one` (dynamic instances of one child type). It documents restart intensity and period to stop infinite restart loops. Child restart types include `permanent` (always restarted), `temporary` (never restarted), and the manual's other documented types. This pass relies on that manual's strategy list rather than memory of older OTP books.

**Verified.** The Kubernetes controller document describes controllers as non-terminating loops that move current state toward desired state. A controller tracks a resource whose spec is desired state. The Job controller does not run Pods itself; it asks the API server to create or remove Pods. Controllers can fail, so Kubernetes prefers many small controllers. The cluster may never be stable. Desired state is reconciled continuously, not only when an edge event arrives.

**Verified.** The SRE monitoring chapter distinguishes symptoms ("what is broken") from causes ("why"), and black-box checks from white-box metrics. It names four golden signals: latency, traffic, errors, and saturation. It says paging a human should be rare, actionable, and about a user-visible or imminent problem, not "something seems weird."

## 3. Design principles

- Supervise from outside the process being supervised. A thread that checks itself cannot report its own deadlock.
- Separate infrastructure health from task-progress health, as the skill already requires.
- Restart only when restart can plausibly help. Intensity limits exist because restart loops are a failure mode OTP documents explicitly.
- Reconciliation compares desired task status with observed environment. It does not assume the last event was applied exactly once.
- One authoritative writer per health family. Multiple plugins must not overwrite the same health field.
- Pages and automatic restarts are different. SRE's point is that human alerts must stay rare. A watchdog can repair quietly and page only when repair is exhausted or unsafe.

## 4. Internal mechanisms

Supervision reacts to process termination. It sees an exit. It does not see "the model called the same tool fifty times."

A progress watchdog records a heartbeat that means "meaningful progress," not "process scheduled." Examples already in the skill: time since last progress, repeated identical tool calls, unchanged state across N checks, deadlines.

Reconciliation loop, in the Kubernetes sense:

1. Read desired state (task should be running, cancelled, or completed).
2. Read observed state (process alive, lease epoch, receipt present, worktree lock held).
3. If they differ, take one small action: start a Worker, kill a process group, query an external API, or mark ambiguous.
4. Write the new observed status back through the task store, not around it.
5. Sleep or watch. Repeat. Crash of the reconciler is safe if steps are idempotent.

**Interpretation.** Level-triggered reconciliation tolerates lost events. Edge-triggered handlers are still useful for low latency, but the watchdog should be able to rebuild from the task table if the bus drops a message. That pairs with at-least-once delivery rather than replacing it.

## 5. Architecture and execution flow

```text
Supervisor: process down?
  classify exit
  restart Worker if policy allows and intensity remains
  else mark task failed or hand to reconciler

Watchdog: progress stale?
  observe
  ask Worker for status
  nudge or replan
  cancel Operation
  after grace, hard stop and reconcile
  escalate to a human

Reconciler: desired vs observed
  lease expired -> fence, inspect effects, reassign
  cancel desired but process alive -> cancel path
  receipt missing -> query downstream, do not guess
```

## 6. Benefits

- Process death becomes a policy decision instead of a silent stall.
- Stuck-but-alive tasks get a different response from crashed tasks.
- Lost events heal on the next reconcile.
- Restart storms stop when intensity is exceeded.
- Small controllers can fail independently, which is the Kubernetes design tenet cited above.

## 7. Trade-offs

- Reconciliation is slower than an event that arrives immediately. Use both when latency matters.
- `one_for_all` restart amplifies one child crash into many. Use it only when those children share a fate that is real.
- Aggressive progress timeouts kill slow but healthy work.
- Symptom paging on every retry creates the fatigue SRE describes, and operators start ignoring the watchdog.

## 8. Failure modes

- In-process "watchdog" thread that deadlocks with the worker it monitors.
- Restart on deterministic bugs, forever, until intensity happens to trip.
- Inferring task completion from process exit.
- Two controllers writing the same task because ownership labels were omitted. Kubernetes notes that multiple controllers can create the same kind of object and must distinguish them.
- Reconciler that performs a non-idempotent message every pass because desired and observed stay unequal.

## 9. Mature implementations

- Erlang/OTP supervisors. [Note](../frameworks/erlang-otp.md).
- Kubernetes controllers, especially the Job controller as documented. [Note](../frameworks/kubernetes-controllers.md).
- Temporal activity retry and workflow recovery, which supervise executions rather than OS processes alone. [Note](../frameworks/temporal.md).
- Google SRE monitoring guidance for what is worth an alert.

## 10. Agent Runtime mapping

- **Watchdog** is outside the Worker process. It owns progress policy and escalation.
- **Runtime** owns restart. A Plugin does not fork an unmanaged supervisor.
- **Task** desired status is the spec. Observed process and receipt are the status. This is a reconciliation vocabulary, not a requirement to use Kubernetes APIs.
- **Event** loss is tolerated because the next loop rereads the task.
- **Operation** exit codes are inputs to classification, not the task status by themselves.
- **Telemetry** holds derived health. The task row remains the authoritative status writer.

Conflict: OTP "let it crash" restarts the compute process. It must not replay a non-idempotent effect. Kubernetes reconciliation will retry a fix forever while the gap remains, so the action has to be safe to repeat. Neither slogan overrides the skill's crash-consistency steps.

## 11. When to use

Use a supervisor for process lifecycle, a progress watchdog for agent loops, and a reconciler for anything that touches external resources or more than one process.

## 12. When not to use

Do not deploy a Kubernetes control plane to supervise one local assistant process. Do not page a human because a single tool retry happened. Do not use `one_for_all` because it sounds thorough.

## 13. Original references

| Title | Author / org | URL | Date | Type | Concepts | Why it matters |
|---|---|---|---|---|---|---|
| OTP Design Principles | Ericsson / Erlang | https://www.erlang.org/doc/system/design_principles.html | docs v29.1.2 at retrieval | Official doc | Supervision tree, workers versus supervisors | Lifecycle structure |
| supervisor | Erlang/OTP | https://www.erlang.org/doc/apps/stdlib/supervisor.html | retrieved 2026-11-03 | Official doc | Restart strategies, intensity, child types | Concrete restart bounds |
| Controllers | Kubernetes | https://kubernetes.io/docs/concepts/architecture/controller/ | retrieved 2026-11-03 | Official doc | Desired versus current state, small controllers | Level-triggered repair |
| Monitoring Distributed Systems | Rob Ewaschuk, edited by Betsy Beyer, Google SRE | https://sre.google/sre-book/monitoring-distributed-systems/ | SRE book chapter; retrieved 2026-11-03 | Book chapter | Symptoms versus causes, four golden signals, paging discipline | What a watchdog should alert on |

Reading limit: Joe Armstrong's thesis PDF was reachable at https://www.erlang.org/download/armstrong_thesis_2003.pdf but was not re-read in this pass. Restart strategy details come from the supervisor manual, not from the thesis.
