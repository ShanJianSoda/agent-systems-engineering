# Testing concurrency, delivery, and recovery

Status: written 2026-11-03. Practice note. It points at the skill's existing test matrix and says how to make those tests meaningful.

## 1. Problem statement

Agent runtimes pass unit tests that call a fake model and still lose a task on restart, run an effect twice, or let a stale Worker write. The failures are about interleavings and crashes, which a happy-path test never hits.

## 2. Core concepts

The skill's test matrix already names the cases: illegal transitions, duplicate delivery, out-of-order events, concurrent writers, cancel during a tool call, crash after intent and before receipt, stale lease, saturated queue, prompt injection, unauthorized action, plugin lifecycle, watchdog dead-versus-stuck, and context budget.

**Interpretation.** Each row needs a fault injection point, an expected task status, and an expected external effect count. "Did not throw" is not a recovery test.

## 3. Design principles

- Use deterministic fakes for the model and tools in runtime tests.
- Fail a test if an external effect happens more times than the idempotency rule allows.
- Crash between persist-intent and persist-receipt. Assert the restarted runtime reconciles instead of repeating a non-idempotent fake.
- Advance a fake clock past the lease. Assert the old Worker’s write is rejected.
- Fill the queue. Assert reject or coalesce, not unbounded growth.
- Keep one smaller end-to-end suite with a real model. Do not make it the only place recovery is tested.

## 4. Internal mechanisms

Minimum assertions:

| Injected fault | Expected durable state | Expected effect |
|---|---|---|
| Same event ID twice | One task | One effect |
| Crash after intent, before call | Intent present, receipt absent | No call yet, or one call then ambiguous |
| Crash after call, before receipt | Ambiguous or reconciled receipt | Not a blind second call |
| Worker pause past lease | Epoch incremented | Old epoch write rejected |
| Cancel during tool | Task cancelled | Process group stopped or tool abort recorded |
| Queue at limit | New work rejected or coalesced | Interaction slot still available |
| Plugin stop | Background loop cancelled | No further events from that plugin |

Record what was not tested. The skill's output format already requires known limitations.

## 5. Architecture and execution flow

Run state-machine and idempotency tests in process. Run crash tests in a subprocess or by truncating the log at a chosen flush. Run lease tests with a clock interface. Do not sleep on wall clocks in unit tests.

## 6. Benefits

- Regressions show up as duplicate effects or illegal states, which are the production failures.
- Fakes keep the suite fast enough to run on every change.

## 7. Trade-offs

- Fakes can encode the bug. A fake that cannot time out will never produce an ambiguous receipt. Give fakes a timeout and a "succeed after the client gave up" mode.
- End-to-end model tests are noisy. Do not use them to assert exact prose.

## 8. Failure modes

- Testing replay by calling the model again and expecting the same text.
- Asserting process exit equals task completion.
- Concurrency test with a lock so wide that the race cannot happen.

## 9. Mature implementations

Temporal's determinism rules are a reminder that replay tests should compare recorded commands, not re-execute external calls: https://docs.temporal.io/workflow-definition (retrieved 2026-11-03). Akka ships a persistence testkit, named on the persistence page (Akka 2.10.24). This note did not re-read the testkit manual.

## 10. Agent Runtime mapping

Tests name Event, Task, Session, Operation, lease epoch, Plugin, Watchdog, Recorder, and Observability explicitly. A test that only checks a generic "agent loop" return string does not cover this skill.

## 11. When to use

Use this matrix before claiming restart safety, idempotency, or cancellation.

## 12. When not to use

Do not block a one-day prototype on a cluster chaos suite. Do block a claim of durability on the crash-between-intent-and-receipt case.

## 13. Original references

| Title | Author / org | URL | Date | Type | Concepts | Why it matters |
|---|---|---|---|---|---|---|
| Temporal Workflow Definition | Temporal | https://docs.temporal.io/workflow-definition | retrieved 2026-11-03 | Official doc | Replay compares commands | What a replay test can assert |
| Event Sourcing | Akka | https://doc.akka.io/libraries/akka-core/current/typed/persistence.html | retrieved 2026-11-03 | Official doc | Persistence testkit exists | A mature project ships recovery tests. Manual not re-read |
| How to do distributed locking | Martin Kleppmann | https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html | 8 February 2016 | Article | Pause past lease | The stale-writer case to inject |
