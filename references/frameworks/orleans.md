# Microsoft Orleans

Status: written 2026-11-03 from the Orleans overview and the opening of MSR-TR-2014-41. Grain persistence, transactions, and the directory protocol were not re-read beyond what those two sources state.

## Problem it actually solves

The 2014 report states that high-scale interactive services need throughput, low latency, and availability, and that a stateless middle tier plus storage on every request does not provide data locality. Orleans uses virtual actors so developers write stateful objects while the runtime handles activation, placement, and failure recovery. The overview presents current Orleans as a .NET framework for cloud-native distributed applications.

## Core abstractions

**Verified** from the overview:

- **Grain**: a virtual actor with identity, behavior, and optional state. Keys can be GUID, integer, string, or compound. Grains are always addressable. Callers do not create or destroy the identity. The runtime activates and deactivates.
- **Silo**: process that hosts grains. Silos form a cluster.
- **Placement**: runtime chooses a silo. Policies exist; the overview says the 9.2 default changed from random to resource-optimized. That version note is from the overview's "what's new" section.
- **Timers versus reminders**: timers are not durable; reminders are a durable scheduling mechanism that can fire even if the grain is not currently active.
- **Stateless worker grains**: can activate in multiple places because they are marked stateless.
- **Filters**: incoming and outgoing grain call filters for cross-cutting logic such as authorization and telemetry.
- The overview also lists persistence via `WriteStateAsync`, streams, and distributed ACID transactions with serializable isolation and no central transaction manager. Transaction internals were not re-read.

**Verified** from the report abstract and introduction: virtual actors are the programming model that hides reliability and resource management that Erlang and Akka, in the authors' view at the time, still left to developers. The report is dated March 2014, MSR-TR-2014-41. Performance numbers in the paper were not used here.

## Execution model

A call to a grain identity activates it if needed, runs the call, and keeps state in memory while active. The overview says unused grains are deactivated and can be activated again on the next call. Single-threaded grain turn semantics are commonly described for Orleans; the overview text read here says actors encapsulate state and communicate by asynchronous messages, and that the runtime manages lifecycle. It does not, in the sections read, spell the current turn scheduler. Do not add a concurrency claim beyond that.

## Persistence model

In-memory while active. `WriteStateAsync` writes named state objects to a configured store. The overview says this should happen so durability matches the update. Reminders are the durable timer. Loss of a silo is recovered by activating the grain elsewhere from stored state, not by replaying an in-memory mailbox. Mailbox durability was not claimed on the pages read.

## Failure handling

The overview says failure recovery is transparent because identity is stable and callers do not track the server. Membership failure detection is mentioned as improved in 9.x (90 seconds versus 10 minutes in earlier versions). That is a product claim on the overview page, not a measurement repeated in this repository. Ambiguous in-flight calls during a crash are not given an exactly-once proof in the text that was read.

## Limitations

- Virtual "always exists" can obscure leases and explicit cancellation. Deactivation is not task completion.
- State not written with `WriteStateAsync` dies with the activation.
- Transactions are a separate feature. Do not assume every grain update is a distributed transaction.
- Orleans is a .NET cluster runtime. It is a large dependency for a local harness.
- The 2014 paper describes the model. Current defaults, including placement, have changed. Prefer the overview for current product facts, and re-check version numbers before coding.

## Agent Runtime mapping

| Orleans | This skill |
|---|---|
| Grain identity | Stable Task or Session ID, not "the object is the lifecycle" |
| Activation | Worker assignment |
| `WriteStateAsync` | Durable task state |
| Reminder | Durable wake-up, still subject to policy |
| Call filter | Host authorization and telemetry pipeline |
| Silo | Execution host, not a plugin |

Conflict: grains that mutate each other freely do not match plugin capability checks. Put authorization in a filter or a Router, not in hope that the caller grain is well behaved. See [leases](../distributed-systems/leases-fencing-retries.md) for why overlapping activations during failure still need a store-side version.

## References

| Title | Author / org | URL | Date | Type |
|---|---|---|---|---|
| Orleans overview | Microsoft | https://learn.microsoft.com/en-us/dotnet/orleans/overview | retrieved 2026-11-03 | Official doc |
| Orleans: Distributed Virtual Actors for Programmability and Scalability | Phil Bernstein, Sergey Bykov, Alan Geller, Gabriel Kliot, Jorgen Thelin | https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/Orleans-MSR-TR-2014-41.pdf | MSR-TR-2014-41, March 2014 | Research report |
| Publication page | Microsoft Research | https://www.microsoft.com/en-us/research/publication/orleans-distributed-virtual-actors-for-programmability-and-scalability/ | March 2014 | Bibliographic page |
