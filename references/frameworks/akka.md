# Akka

Status: written 2026-11-03 from the typed actor introduction and the typed persistence page. Cluster sharding and delivery guarantees beyond those pages were not re-read.

## Problem it actually solves

Akka provides in-process and distributed actors: encapsulated state, mailbox concurrency without locks, and parent supervision. Persistence adds recovery of actor state after crash or migration by replaying events or by storing the latest state.

## Core abstractions

**Verified** from the actor guide:

- Mailbox, behavior, message, execution environment, address.
- Send does not transfer the caller thread.
- At most one message is processed at a time per actor, so internal updates need no lock.
- Parent creates children and supervises them. Stopping a parent stops children recursively.
- Domain errors are ordinary reply messages. Internal faults go to the supervisor, which may restart or stop the child.
- The guide states that restarts are not visible from the outside and that collaborators can keep sending during restart. Messages are not described as durable on that page.

**Verified** from persistence (page cites module version 2.10.24, license BUSL-1.1):

- Event sourced behavior stores events by append, not the full actor state. Snapshots are optional and shorten replay.
- A command is validated, events are persisted, and only then applied. Replay applies events, not commands.
- Durable State persists the latest state instead, described as closer to CRUD.
- A journal plugin is required. Plugin behavior was not re-read.

## Execution model

A scheduler runs actors that have mailboxes. Actors run concurrently with each other. The guide's claim that millions of actors can share a small thread pool is a vendor description of the model, not a measurement repeated here.

## Persistence model

Append-only journal plus optional snapshot, or latest-state storage. Recovery rebuilds the behavior by replay. Nothing in the pages read makes external HTTP calls idempotent.

## Failure handling

Supervision strategy on the parent. Persistence recovers state after the actor is restarted by a supervisor, a crash, or a move. The persistence page says events should not fail on replay because they already succeeded once. That is a constraint on event handlers.

## Limitations

- Mailbox is not a durable outbox unless persistence or an external broker is added.
- Restart plus at-least-once delivery can reprocess a command. Commands are supposed to be validated again; side effects inside a command handler would repeat. The guide says side effects of commands should become events that are persisted first. Putting I/O in the event handler replays that I/O.
- License on the persistence page is BUSL-1.1. Decide whether that is acceptable before adopting the dependency. This note is not legal advice.
- Akka is a toolkit. It will not enforce agent policy, session boundaries, or plugin capabilities unless you build them.

## Agent Runtime mapping

| Akka | This skill |
|---|---|
| Actor | Router or Worker mailbox |
| Parent supervisor | Runtime, not the plugin |
| Command | Command |
| Persisted event | Recorder or task transition |
| Snapshot | Context or state checkpoint |
| Durable state | Current task row when history is unnecessary |

Conflict: a plugin that creates a root actor bypasses host lifecycle. See [plugin case](../design-cases/plugin-host-isolation.md).

## References

| Title | Author / org | URL | Date | Type |
|---|---|---|---|---|
| How the Actor Model Meets the Needs of Modern, Distributed Systems | Akka | https://doc.akka.io/libraries/akka-core/current/typed/guide/actors-intro.html | retrieved 2026-11-03 | Official doc |
| Event Sourcing | Akka | https://doc.akka.io/libraries/akka-core/current/typed/persistence.html | page lists Akka 2.10.24; retrieved 2026-11-03 | Official doc |
