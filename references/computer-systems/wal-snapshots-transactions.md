# WAL, snapshots, and transactions

Status: written 2026-11-03. **Verified** restates sources read in this pass. **Interpretation** is for Agent runtimes.

## 1. Problem statement

A crash can occur after a model has decided to act and before the runtime has a durable record of that decision, or after an external effect and before its receipt is stored. Recovery then either repeats a payment or message, or forgets that the user asked for work. Write-ahead logging and snapshots are how storage engines make that window explicit.

## 2. Core concepts

**Verified.** PostgreSQL's WAL introduction states the central rule: changes to data files must be written only after those changes have been logged, meaning after WAL records describing the changes have been flushed to permanent storage. After a crash, changes not yet in data pages can be redone from the WAL. PostgreSQL calls this roll-forward recovery (REDO). It also states that flushing the sequential WAL is enough to commit, so every changed data page need not be flushed on each commit, and that one WAL flush can commit many small transactions. Archiving WAL plus a prior physical backup supports point-in-time recovery. The backup itself need not be an instantaneous snapshot; replaying WAL across the backup window fixes inconsistencies.

**Verified.** ARIES (Mohan, Haderle, Lindsay, Pirahesh, Schwarz, ACM TODS, March 1992) is a recovery method using write-ahead logging. The ACM abstract and the paper opening read in this pass state that ARIES repeats history: it redoes missing updates before rolling back loser transactions. It uses a log sequence number (LSN) on each page to relate the page to logged updates. Updates performed during rollback are also logged. The paper describes support for fuzzy checkpoints, partial rollback, and operation logging. This note does not re-derive the full ARIES restart algorithm.

**Interpretation.** An agent "transaction" is often not a database transaction. It is a durable intent, an external effect, and a receipt. WAL discipline applies to that trio even when the effect is a git push or a chat message that no database can roll back.

## 3. Design principles

- Persist intent before the external effect.
- Give the effect an idempotency key when the external system supports one.
- Persist the receipt after the effect.
- A crash between effect and receipt is ambiguous. Do not blindly retry non-idempotent effects.
- Snapshots are an optimization for recovery time, not a replacement for the log of intents and receipts.
- Illegal task transitions are rejected with a version check, analogous to not applying a log record that does not match the page LSN.

## 4. Internal mechanisms

PostgreSQL's rule, restated as an agent log:

1. Append a record: task version, transition, effect intent, idempotency key. Flush it.
2. Perform the effect.
3. Append the receipt, including ambiguity if the call timed out. Flush it.
4. On restart, replay from the last snapshot: reapply committed transitions, find intents without receipts, reconcile.

**Verified.** ARIES repeats history before undo so that the database matches the log and then undoes losers. **Interpretation.** Agent recovery should not undo an external message that already happened. Replay rebuilds internal state. External ambiguity is a reconciliation case, not an automatic undo. That is a deliberate difference from database recovery.

Fuzzy checkpoints, as described for ARIES, allow a checkpoint without stopping all writes. **Interpretation.** An agent snapshot of session summaries can be taken while tasks still run, if each record has a version and the log after the snapshot is replayed.

## 5. Architecture and execution flow

```text
Command accepted
  WAL: intent (task, effect, key, policy version)
  fsync
  Operation performs effect
  WAL: receipt | ambiguous
  fsync
Crash
  load snapshot
  redo WAL after snapshot
  intents without receipts -> reconciliation queue
  Watchdog does not mark them completed
```

Recorder persists this trajectory. Telemetry may count ambiguous receipts. Observability reads them and does not become a second writer of the log.

## 6. Benefits

- Restart does not erase task intent.
- Commit cost is a sequential flush, which PostgreSQL cites as cheaper than flushing every data page.
- Point-in-time copies support audit and debugging of how a task reached a state.
- Versioned records reject stale Workers, pairing with fences.

## 7. Trade-offs

- A flush per effect increases latency. Grouping commits, which PostgreSQL notes can happen when many small transactions share one WAL flush, applies only when the effects are allowed to commit together.
- The log grows until it is snapshotted and truncated. Unbounded event history is a storage incident.
- Redo of agent state is not redo of the outside world.
- Putting large transcripts in the log destroys the cost model. Store references, as the skill already requires for task rows.

## 8. Failure modes

- Writing the data file first and the log later. Crash loses the commit or exposes a torn write. This violates the WAL rule PostgreSQL states.
- Snapshot without a log position. Recovery cannot know what happened after the snapshot.
- Replaying effects during redo rather than replaying state transitions.
- Two writers appending "the" task row without a version or LSN.

## 9. Mature implementations

- PostgreSQL WAL, including online backup and point-in-time recovery as documented in the cited chapter.
- ARIES-family database recovery.
- Akka Persistence appends events and can recover from a snapshot plus subsequent events. See [Akka](../frameworks/akka.md).
- Temporal Event History is a durable log of workflow commands and events, with replay into workflow state. See [durable execution](../distributed-systems/durable-execution.md).

## 10. Agent Runtime mapping

| Storage idea | Runtime field |
|---|---|
| WAL record | Task transition, effect intent, receipt |
| Page LSN | Task version / fencing token |
| Fuzzy checkpoint | Session or task snapshot with a log position |
| REDO | Rebuild Router and Worker state after restart |
| Undo of loser | Only internal state that never became an external effect |
| Point-in-time | Audit of Recorder history, not a second mutable store |

Operation does not open its own log. Runtime and Recorder own durability. Plugins store namespaced state with schema versions, not private writes into the host log.

## 11. When to use

Use a WAL or an equivalent append-only table whenever a task can have an external effect or must survive process restart.

## 12. When not to use

Do not put a full database recovery algorithm under a prototype that only stores chat memory in one file. A single append-only intent log plus receipts is enough until concurrent writers or media failure appear. Do not adopt event sourcing for every CRUD setting. See [event-driven systems](../architecture-patterns/event-driven-outbox-cqrs.md).

## 13. Original references

| Title | Author / org | URL | Date | Type | Concepts | Why it matters |
|---|---|---|---|---|---|---|
| Write-Ahead Logging (WAL) | PostgreSQL Global Development Group | https://www.postgresql.org/docs/current/wal-intro.html | Documentation 18; retrieved 2026-11-03 | Official doc | WAL-before-data, REDO, PITR, group commit | Precise statement of the logging rule |
| ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging | C. Mohan, Don Haderle, Bruce Lindsay, Hamid Pirahesh, Peter Schwarz | https://dl.acm.org/doi/10.1145/128765.128770 | ACM TODS 17(1), March 1992, pp. 94–162 | Paper | Repeating history, LSN, fuzzy checkpoints | Canonical recovery method. PDF read in part: https://www.cs.cmu.edu/~15849g/readings/mohan92.pdf |
| Event Sourcing | Akka | https://doc.akka.io/libraries/akka-core/current/typed/persistence.html | page cites Akka 2.10.24; retrieved 2026-11-03 | Official doc | Append events, replay, snapshots, durable state | Shows snapshot-plus-log in an actor runtime |

Reading limit: the ARIES paper is long. This pass used the abstract and opening claims (repeat history, LSN, logged undo, fuzzy checkpoints), not every lock mode or figure.
