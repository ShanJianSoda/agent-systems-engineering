# Reference map

This skill synthesizes systems engineering for Agent runtimes. It is not an implementation of any paper or framework. `SKILL.md` remains the constraint set. This file is the retrieval index. Open the smallest set of notes that matches the problem.

Reading rule: **Verified** claims in the notes restate sources read on 2026-11-03. **Interpretation** is local engineering judgment. If a note marks a source as unread, do not invent the missing detail.

## Read this for the problem in front of you

| Problem | Read | Also compare |
|---|---|---|
| Who runs, who is isolated, what the kernel schedules | [Processes, threads, scheduling](computer-systems/processes-threads-scheduling.md) | [Ray](frameworks/ray.md), [Erlang/OTP](frameworks/erlang-otp.md) |
| Cancel, signals, child processes | [IPC and cancellation](computer-systems/ipc-signals-cancellation.md) | [Celery](frameworks/celery.md), [Ray](frameworks/ray.md) |
| Shared state, mailboxes, races | [Actors and mailboxes](computer-systems/actors-mailboxes-synchronization.md) | [Akka](frameworks/akka.md), [Orleans](frameworks/orleans.md) |
| Intent, crash, redo | [WAL and snapshots](computer-systems/wal-snapshots-transactions.md) | [Durable execution](distributed-systems/durable-execution.md) |
| Context window cost | [Locality and working sets](computer-systems/locality-and-working-sets.md) | Skill section on Inline / Spill |
| Duplicates, loss, "exactly once" | [Delivery and idempotency](distributed-systems/delivery-consistency-idempotency.md) | [NATS JetStream](frameworks/nats-jetstream.md), [Celery](frameworks/celery.md) |
| Stale Worker writes, retry storms | [Leases and fencing](distributed-systems/leases-fencing-retries.md) | [Temporal](frameworks/temporal.md) |
| Resume after restart | [Durable execution](distributed-systems/durable-execution.md) | [Durable task case](design-cases/durable-task-loop.md) |
| Publish and state commit together; read models | [Events, outbox, CQRS](architecture-patterns/event-driven-outbox-cqrs.md) | [NATS JetStream](frameworks/nats-jetstream.md) |
| Stuck tasks, crash loops, desired state | [Supervision and reconciliation](architecture-patterns/supervision-watchdog-reconciliation.md) | [Kubernetes controllers](frameworks/kubernetes-controllers.md), [Erlang/OTP](frameworks/erlang-otp.md) |
| Plugin trust boundary | [Plugins and capabilities](architecture-patterns/plugins-and-capabilities.md) | [Plugin case](design-cases/plugin-host-isolation.md) |
| Logs, metrics, traces, paging | [Observability](architecture-patterns/observability.md) | — |
| Queue growth, batching, priority | [Backpressure and scheduling](architecture-patterns/backpressure-batching-scheduling.md) | [Proactive case](design-cases/proactive-trigger-pipeline.md) |
| Whether the choice deserves a record | [Decision and failure discipline](engineering-practices/decision-and-failure-discipline.md) | [templates](../templates/design-decision-record.md) |
| Proving recovery and idempotency | [Testing concurrency and recovery](engineering-practices/testing-concurrent-and-recovery.md) | Skill test matrix |

Templates:

- [Architecture review](../templates/architecture-review.md)
- [Design decision record](../templates/design-decision-record.md)
- [Failure analysis](../templates/failure-analysis.md)

## Framework notes

Each note states the problem that system solves, its abstractions, execution, persistence, failure behavior, and limits. Do not force them into one matrix.

- [Temporal](frameworks/temporal.md) — durable control flow and supervised activities.
- [Kubernetes controllers](frameworks/kubernetes-controllers.md) — level-triggered desired state.
- [Erlang/OTP](frameworks/erlang-otp.md) — supervision trees and behaviours.
- [Akka](frameworks/akka.md) — mailboxes, supervision, event-sourced actors.
- [Orleans](frameworks/orleans.md) — virtual actors and activation.
- [NATS JetStream](frameworks/nats-jetstream.md) — durable at-least-once streams.
- [Ray](frameworks/ray.md) — tasks and process actors for heterogeneous compute.
- [Celery](frameworks/celery.md) — broker task queue and acknowledgement trade-offs.

## Design cases

- [Durable task loop](design-cases/durable-task-loop.md)
- [Proactive trigger pipeline](design-cases/proactive-trigger-pipeline.md)
- [Plugin host isolation](design-cases/plugin-host-isolation.md)

## Prior agent-architecture sources

These motivated the skill. They are not the systems bibliography above.

1. DeepSeek AI, `dsh-superagent-prototype`  
   https://github.com/deepseek-ai/dsh-superagent-prototype  
   Public research prototype. The repository was reachable on 2026-11-03. Its README has said the prototype is not production-ready and does not fully implement every design in the article. Re-read the repository before relying on filenames or APIs.

2. DeepSeek Harness, “Exploring Super-agent Design: Towards a More Autonomous Agent System”  
   Links originally supplied with this skill:
   - https://mp.weixin.qq.com/s/zoS7GlReWPhfCjENGxXCdg — HTTP 200 on 2026-11-03. The article body was not re-read in this pass.
   - https://zhuanlan.zhihu.com/p/2099987320443237060 — automated GET returned 403 on 2026-11-03. The article body was not read.

3. Roger-Lv, “Super-agent 设计探索：从架构理念到 dsh-superagent 原型实现”  
   https://roger-lv.github.io/2026/11/01/2026-11-01-Super-agent%E8%AE%BE%E8%AE%A1%E6%8E%A2%E7%B4%A2%EF%BC%9A%E4%BB%8E%E6%9E%B6%E6%9E%84%E7%90%86%E5%BF%B5%E5%88%B0dsh-superagent%E5%8E%9F%E5%9E%8B%E5%AE%9E%E7%8E%B0/  
   HTTP 200 on 2026-11-03. The article body was not re-read in this pass.

Distinguish article proposal, public prototype, and production recommendation. Never claim a prototype implements a paper's full architecture unless the code was checked.

## Research gaps

Recorded so later work does not pretend these were read:

- Lamport, “Time, Clocks, and the Ordering of Events in a Distributed System” (1978). PDF URL responded on 2026-11-03: https://lamport.azurewebsites.net/pubs/time-clocks.pdf. Body not re-read. Delivery notes do not state its theorems.
- Gilbert and Lynch on CAP were not opened.
- Pat Helland, “Life beyond Distributed Transactions: an Apostate’s Opinion” (CIDR 2007). URL responded: http://www.cidrdb.org/cidr2007/papers/cidr07p15.pdf. Body not re-read.
- Peter Denning, “The Working Set Model for Program Behavior” (CACM 1968). ACM landing page returned 403 to an automated request. No full text was read. Locality notes avoid that paper's theorems.
- Henry Levy, *Capability-Based Computer Systems* (1984). Only https://homes.cs.washington.edu/~levy/capabook/ was confirmed. Chapters were not re-read.
- Mark Miller, *Robust Composition*. The previously used PDF host failed TLS verification. Not cited.
- Joe Armstrong's 2003 thesis PDF responded at https://www.erlang.org/download/armstrong_thesis_2003.pdf and was not re-read. OTP behavior is cited from the current manuals.
- Reactive Streams site https://www.reactive-streams.org/ responded. The specification text was not re-read, so backpressure notes do not describe its API.
- Azure Architecture Center transactional-outbox URL returned 404. Richardson's outbox page is the source used instead.
- USENIX HTML page for the Ray talk returned 403. The PDF https://www.usenix.org/system/files/osdi18-moritz.pdf was read in part.
- ARIES was used from the abstract and the opening of the paper, not from a full re-derivation. A later automated GET of the ACM DOI returned 403. The PDF mirror https://www.cs.cmu.edu/~15849g/readings/mohan92.pdf returned 200, and an earlier fetch of the DOI page did return the abstract.
- Gray and Cheriton leases were used from the abstract and the definition of a lease, not from a re-analysis of the V-system numbers. A later automated GET of the ACM DOI returned 403. The PDF https://web.eecs.umich.edu/~mosharaf/Readings/Leases.pdf returned 200.
- Hewitt, Bishop, and Steiger (IJCAI 1973) was used from the bibliographic record and the OCR text of the actor-and-message claim, not as a line-by-line formal semantics.
- Akka cluster sharding, NATS exactly-once publish options, JetStream KV consistency, Celery prefetch, Temporal continue-as-new limits, Kubernetes leader election, Orleans transactions, and Ray's 2018 lineage implementation versus current checkpoints were not fully re-read. Framework notes say so where it matters.
- No new benchmark was run. Paper performance figures are not targets for an Agent runtime.

## Primary systems sources that were read

Use the note that cites each source. The core set:

- Hewitt, Bishop, Steiger, IJCAI 1973: https://ijcai.org/proceedings/1973/31
- Gray and Cheriton, SOSP 1989: https://dl.acm.org/doi/10.1145/74850.74870
- Mohan et al., ARIES, ACM TODS, March 1992: https://dl.acm.org/doi/10.1145/128765.128770
- Bernstein, Bykov, Geller, Kliot, Thelin, Orleans, MSR-TR-2014-41, March 2014: https://www.microsoft.com/en-us/research/publication/orleans-distributed-virtual-actors-for-programmability-and-scalability/
- Moritz, Nishihara, Wang, et al., Ray, OSDI, October 2018: https://www.usenix.org/system/files/osdi18-moritz.pdf
- Kleppmann, distributed locking, 8 February 2016: https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
- Fowler, Event Sourcing, 12 December 2005: https://martinfowler.com/eaaDev/EventSourcing.html
- Fowler, CQRS, 14 July 2011: https://martinfowler.com/bliki/CQRS.html
- Richardson, transactional outbox: https://microservices.io/patterns/data/transactional-outbox.html
- PostgreSQL WAL: https://www.postgresql.org/docs/current/wal-intro.html
- Linux `sched(7)`, `signal(7)`, `namespaces(7)`, `cgroups(7)`, and cgroup v2 docs
- Temporal understanding and workflow definition docs
- Kubernetes controllers concept
- Erlang/OTP design principles and `supervisor` manual
- Akka typed actors and typed persistence
- Orleans overview
- NATS JetStream concepts
- Ray actors and actor fault tolerance
- Celery tasks guide
- OpenTelemetry signals
- Google SRE, “Monitoring Distributed Systems”: https://sre.google/sre-book/monitoring-distributed-systems/
