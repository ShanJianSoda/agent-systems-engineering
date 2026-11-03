# IPC, signals, and cancellation

Status: written 2026-11-03. **Verified** restates sources read in this pass. **Interpretation** is for Agent runtimes.

## 1. Problem statement

Long-running agent work must be stopped on purpose: the user cancels, a deadline fires, a lease expires, or a shutdown begins. Stopping a function in-process does not stop a child process, a remote job, or a side effect already sent to another system. The runtime needs a control message that is distinct from ordinary work, plus an operating-system mechanism that can interrupt a process which is not polling.

## 2. Core concepts

**Verified.** Linux `signal(7)` describes signals as a mechanism for notifying a process of an event. Some signals can be caught or ignored. `SIGKILL` and `SIGSTOP` cannot be caught or ignored. A signal may terminate the process, stop it, or be handled by application code.

**Verified.** The Celery tasks documentation states that a worker acknowledges a message even when `acks_late` is set if the child process executing the task is terminated by `sys.exit()` or by a signal. Termination is therefore not the same thing as a clean application-level cancel.

**Verified.** Temporal's understanding document lists cancel as an operation available through its CLI, and describes Workflows that can be held with Workflow Pause without terminating them or losing state. Cancellation in that system is an API on an execution, not only a Unix signal.

**Interpretation.** This skill already requires cancellation to be a first-class control event. Unix signals are one transport for that event at the process boundary. They are not the domain model.

## 3. Design principles

- Persist the cancellation decision before relying on it. A process can die before it observes the signal.
- Cooperative cancel first: set a flag, abort the in-flight Operation, ask the tool to stop. Then a bounded hard stop.
- Hard stop is `SIGKILL` or an equivalent destroy API. It does not run cleanup. The runtime must reconcile external resources afterward.
- IPC that carries work (events, task assignments) must not be the only channel that can carry cancel. A saturated work queue must not delay a cancel behind a thousand ordinary messages, unless the mailbox is documented to prioritize control messages.

## 4. Internal mechanisms

Typical local IPC choices:

| Mechanism | What it is good for | What it does not do |
|---|---|---|
| In-process queue or mailbox | Assignments and results inside one trust domain | Does not stop a blocked syscall by itself |
| Pipe or socket to a child | Structured protocol with a child tool | Dies with the child; needs a reader |
| Unix signal | Interrupt or kill a process that is not reading | `SIGKILL` skips cleanup; `SIGTERM` can be ignored |
| Remote API cancel | Temporal-style execution cancel, HTTP abort | Only works if the remote system implements it |

**Verified.** `signal(7)` is the source for the catchable versus uncatchable distinction. The exact default action of each signal is in that manual; this note does not reproduce the full table.

**Interpretation.** A Worker should handle `SIGTERM` as "begin shutdown": stop accepting Operations, cancel children, flush receipts, then exit. The Watchdog may send `SIGKILL` only after a documented grace period. The Task record must already say `cancelled` or `expired` so a restart does not treat the dead process as success.

## 5. Architecture and execution flow

```text
Cancel command or deadline
  -> Router persists Task status and reason
  -> control message to Worker mailbox (high priority)
  -> Worker aborts current Operation cooperatively
  -> Worker signals tool process group (SIGTERM)
  -> grace timer
  -> SIGKILL process group if still alive
  -> persist receipt: cancelled, killed, or ambiguous
  -> Watchdog reconciles environment (locks, browsers, worktrees)
```

If the Worker itself is gone, the Watchdog performs the process-group kill and reconciliation. Do not wait for the dead Worker to acknowledge.

## 6. Benefits

- Users can stop work without restarting the runtime.
- Control messages stay distinct from Events that merely report facts.
- Hard kill is available when cooperative cancel is ignored.
- Persisting the decision makes restart idempotent: the task is already terminal.

## 7. Trade-offs

- Cooperative cancel requires every tool and plugin to poll or to use interruptible I/O.
- `SIGKILL` leaks temporary files, locks, and browser sessions unless reconciliation is real.
- Prioritizing cancel inside a single mailbox is easy to forget when the queue is FIFO.
- Remote cancel is best-effort. Ray's actor documentation explicitly says cancellation is not always guaranteed if the request does not reach the executor. See [Ray](../frameworks/ray.md).

## 8. Failure modes

- Cancel recorded only in a prompt, not in Task state. Restart resumes the work.
- `SIGTERM` sent to the parent shell but not the process group. Grandchild tools keep running.
- Treating signal death as task failure and retrying a non-idempotent external message.
- Plugin starts its own background loop and never subscribes to host shutdown.

## 9. Mature implementations

- POSIX/Linux signals, `signal(7)`.
- Temporal cancel and pause on Workflow executions.
- Ray `ray.cancel` on actor task object refs, with best-effort semantics and cooperative checks for sync actor tasks.
- Celery's interaction between signals and acknowledgements, which can drop or finish a message differently than the application expects.

## 10. Agent Runtime mapping

- **Event** may record `task.cancel_requested` as a fact.
- **Task** stores the terminal status and reason. Cancellation is not a suggestion inside Session context.
- **Operation** must observe the cancel flag or an abort exception. It stays a thin wrapper; it does not invent a second workflow engine.
- **Worker** owns the process group for tools it spawned.
- **Plugin** background work is tracked by the host and cancelled on `stop`. An ASYNC handler must not block the producer while it cancels.
- **Watchdog** escalates from observe, to request status, to cancel, to hard stop, matching the existing graduated responses.

## 11. When to use

Use this path for any Operation that can exceed the interaction deadline, spawn a process, or call a remote job API.

## 12. When not to use

Do not use `SIGKILL` as the first response to a slow model call that can still be aborted at the HTTP layer. Do not build a distributed signal bus. Cross-host cancel is a persisted control record plus the remote system's cancel API.

## 13. Original references

| Title | Author / org | URL | Date | Type | Concepts | Why it matters |
|---|---|---|---|---|---|---|
| signal(7) | Linux man-pages | https://man7.org/linux/man-pages/man7/signal.7.html | retrieved 2026-11-03 | Manual | Catchable signals, SIGKILL, SIGSTOP | Hard stop versus cooperative handling |
| Tasks | Celery | https://docs.celeryq.dev/en/stable/userguide/tasks.html | stable docs; retrieved 2026-11-03 | Official doc | Ack on signal termination, idempotency | Shows process death and queue ack are different events |
| Understanding Temporal | Temporal | https://docs.temporal.io/evaluate/understanding-temporal | retrieved 2026-11-03 | Official doc | Cancel, pause, durable timers | Cancel as an execution API that keeps history |
| Actors | Ray | https://docs.ray.io/en/latest/ray-core/actors.html | retrieved 2026-11-03 | Official doc | `ray.cancel`, best-effort, `is_canceled()` | Cooperative cancel inside a worker process |

Reading limit: this note did not re-read the POSIX standard text line by line. Signal behavior cited here is from the Linux man-page retrieved above.
