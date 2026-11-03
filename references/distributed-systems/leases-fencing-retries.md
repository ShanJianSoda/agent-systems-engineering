# Leases, fencing, and retries

Status: written 2026-11-03. **Verified** restates sources read in this pass. **Interpretation** is for Agent runtimes.

## 1. Problem statement

A Worker can pause, lose its network, or be declared dead while it still completes a write. Timeouts free the task so someone else can continue, but the old Worker may wake up and write too. Retries without a budget turn a dependency failure into a self-inflicted outage. Leases answer "who may act now." Fencing tokens answer "whose write is still legal." Retry budgets answer "when to stop trying."

## 2. Core concepts

**Verified.** Gray and Cheriton, SOSP 1989, define a lease as a contract that gives its holder specified rights over property for a limited time. In their cache protocol, a lease grants control over writes to the covered datum during the term. They argue that non-Byzantine failures affect performance rather than correctness when leases are short, because holders lose rights at a deadline even if a revocation message is lost. The paper is about distributed file cache consistency, not about agent task ownership. This pass read the abstract and the lease definition, not the full V-system evaluation.

**Verified.** Martin Kleppmann, 8 February 2016, separates locks used for efficiency from locks used for correctness. He shows a client that pauses past lease expiry and then writes stale data. Checking the lease locally just before the write does not help, because the pause can occur after the check, and because a packet can be delayed until after expiry. His fix is a fencing token: a number that increases every time the lock is acquired. The storage system must remember the highest token it has accepted and reject a write with a lower token. He states that Redlock, as he analyzed it, does not generate such a token.

**Interpretation.** A Worker lease in this skill is the Gray-style time-bounded right to own a task, plus Kleppmann's storage check. Expiry alone is not enough if the database still accepts the old owner's write.

## 3. Design principles

- Every task assignment has an expiry and a monotonically increasing token or version.
- The store rejects writes whose token is older than the newest accepted token, using compare-and-swap or an equivalent.
- Heartbeats extend the lease only while the Worker still owns that token.
- On expiry, reconcile external resources before another Worker starts.
- Retry only errors that a retry can plausibly fix. Bound the count, use backoff and jitter, and open a circuit when the dependency is failing as a class.
- Do not use a lock service that cannot mint fencing tokens for correctness-critical task ownership.

## 4. Internal mechanisms

Lease lifecycle:

1. Router assigns Task to Worker with `lease_epoch = n` and `expires_at`.
2. Worker writes progress only with `expected_epoch = n`.
3. Store accepts the write if `n` is still current, and may extend `expires_at` on heartbeat.
4. If the Worker is silent past `expires_at`, the epoch becomes free.
5. Router increments to `n+1` and assigns a new Worker only after reconciliation.
6. A late write with `n` fails the compare.

**Verified.** Kleppmann's storage server is an active participant. **Interpretation.** The task table is that storage server. A Redis lock around the Worker, with the task table trusting whoever presents the lock, recreates the bug he describes.

Retry classes:

| Class | Example | Retry? |
|---|---|---|
| Transient transport | connection reset | Yes, with budget |
| Ambiguous timeout | HTTP client timed out after the server maybe acted | Reconcile, do not blindly repeat |
| Deterministic reject | validation, auth | No |
| Resource exhausted | quota, disk full | Back off or shed load, do not hot-loop |
| Poison input | crash on one event | Dead-letter, do not restart forever |

Erlang supervisors add a restart intensity and period for the same reason: a child that crashes in a loop must stop the restart loop. See [supervision](../architecture-patterns/supervision-watchdog-reconciliation.md).

## 5. Architecture and execution flow

```text
assign(epoch=n, deadline)
  Worker heartbeats
  effect(idempotency key, epoch)
  store.compare_and_swap(epoch)
pause longer than lease
  Watchdog reconciles
  assign(epoch=n+1)
late Worker write epoch n
  store rejects
  Worker must stop
```

Cancellation and lease loss are different. Cancel is a user or policy decision that makes the task terminal. Lease loss means this Worker must stop while the task may continue under a new epoch.

## 6. Benefits

- Crashed Workers do not hold tasks forever. Gray and Cheriton's point is that a missed revocation still expires.
- Stale Workers cannot clobber a newer owner if the store checks tokens, which is Kleppmann's point.
- Retry budgets protect dependencies.
- Epochs make audit trails explain which Worker was allowed to act.

## 7. Trade-offs

- Short leases increase heartbeat traffic and false expiry during GC or load pauses. Kleppmann lists GC, page faults, scheduler delay, and network delay as reasons pauses happen.
- Long leases slow failover.
- The store must be in the check. A fence that only the Worker believes is useless.
- Clock skew matters for expiry. Prefer a store-side deadline over comparing two machines' wall clocks in the Worker.

## 8. Failure modes

- Lease without fencing, plus a correctness-critical write.
- Heartbeat from a thread that is alive while the Operation thread is stuck. Liveness is not progress.
- Retry of a non-idempotent message after timeout.
- Restart intensity left unlimited, so a bad plugin crash-loops the host.
- Using a lock for efficiency and then silently promoting it to a correctness lock later.

## 9. Mature implementations

- Gray and Cheriton leases for cache consistency. The mechanism is the time bound, not their file-cache protocol.
- Kleppmann's fencing-token correction to distributed locks. He points at ZooKeeper zxid or znode version as tokens that already increase. This note did not re-test ZooKeeper.
- Erlang supervisor intensity and period.
- Temporal activity heartbeats and retry policies, documented as configuration on activities rather than as application locks. See [Temporal](../frameworks/temporal.md).

## 10. Agent Runtime mapping

- **Task.assignedWorkerId** plus **version** is the lease record. The version is the fence.
- **Worker** may update the task only while it holds the current version.
- **Watchdog** expires leases and separates process heartbeat from task progress.
- **Operation** includes the epoch in any external idempotency key when two epochs must not both succeed.
- **Plugin** code does not grant itself a longer lease or write the task row directly.

Conflict: a virtual actor that "always exists" (Orleans) can hide epoch changes inside activation. If activations move, the persistent store still needs a token check. Do not assume the runtime's single-threaded grain replaces fencing once more than one activation can overlap during failure.

## 11. When to use

Use a lease and a fencing token whenever more than one process might run the same task, including after a crash on a single host that restarts quickly.

## 12. When not to use

Do not use a distributed lock for a local in-memory demo with one Worker. Do not adopt Redlock-style algorithms for correctness based on this note; Kleppmann's objection is that the algorithm he reviewed does not issue fencing tokens. Re-read the current Redis documentation before making a new claim about Redlock, which was not re-read here.

## 13. Original references

| Title | Author / org | URL | Date | Type | Concepts | Why it matters |
|---|---|---|---|---|---|---|
| Leases: An Efficient Fault-Tolerant Mechanism for Distributed File Cache Consistency | Cary G. Gray, David R. Cheriton | https://dl.acm.org/doi/10.1145/74850.74870 | SOSP, November 1989, pp. 202–210 | Paper | Time-bounded rights, failure affects performance if leases are short | Primary definition of leases. PDF excerpt: https://web.eecs.umich.edu/~mosharaf/Readings/Leases.pdf |
| How to do distributed locking | Martin Kleppmann | https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html | 8 February 2016 | Article | Efficiency versus correctness, fencing tokens, pauses | Why lease expiry without store checks is unsafe |
| supervisor | Erlang/OTP | https://www.erlang.org/doc/apps/stdlib/supervisor.html | current stdlib manual; retrieved 2026-11-03 | Official doc | Restart intensity and period | A production bound on retry loops |

Reading limit: the 1989 paper's performance model and V-system measurements were not re-analyzed. Do not quote its speedups.
