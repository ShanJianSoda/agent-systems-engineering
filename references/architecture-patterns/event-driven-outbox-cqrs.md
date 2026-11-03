# Event-driven systems: bus, outbox, event sourcing, CQRS

Status: written 2026-11-03. **Verified** restates sources read in this pass. **Interpretation** is for Agent runtimes.

## 1. Problem statement

Agent systems react to facts: a message arrived, a job finished, a file changed. Those facts must be stored, routed, and sometimes published to other components. If the code updates a task row and then publishes, a crash can commit the row and lose the publication, or publish and then roll back the row. If every reader scans the task table, status queries contend with updates. These are different problems and they have different patterns.

## 2. Core concepts

**Verified.** Martin Fowler's Event Sourcing note (12 December 2005) says every change to application state is stored as an event object, in order, for the same lifetime as the state. The event log can rebuild past states. Fowler marks the page as draft material he was not continuing to revise.

**Verified.** Fowler's CQRS article (14 July 2011) attributes the pattern's name and the idea he first heard to Greg Young. CQRS uses a different model to update information than to read it. He says separate models are often different object models, possibly different databases, and that the split raises eventual consistency. He recommends limiting CQRS to a specific bounded context, and says that in his experience the majority of uses he had run into were not good. Benefits he does endorse: some complex domains, and separating read and write load. He says a reporting database can offload heavy queries without splitting every model.

**Verified.** Richardson's transactional outbox stores the outgoing message in the same database transaction as the business update. A separate relay sends the outbox rows to the broker. If the transaction rolls back, the message is not sent. The relay can still publish a message more than once. Order should follow the order the service recorded. He rejects 2PC across the database and the broker as often unavailable or undesirable.

**Verified.** NATS JetStream is a durable log of messages on subjects, with independent consumers. That is a bus plus retention, not an outbox. The application still has to record its own state change atomically with its decision to publish, or accept a gap. See [NATS](../frameworks/nats-jetstream.md).

**Verified.** Microsoft's event sourcing pattern page was reachable. This note does not add claims from it beyond Fowler and Akka, which were read. The Azure "transactional outbox" pattern URL tried in this pass returned 404, so it is not a source here.

## 3. Design principles

- An Event is an immutable fact. A Command asks for a change. A Task is work. Do not collapse them into one "message."
- Publish through an outbox when the task transition and the outbound event must commit together.
- Use event sourcing when you need the trajectory, not when you only need the current task row.
- Use CQRS when the read shape is actually different or the read scale requires it. Fowler's caution is part of the design, not a footnote.
- Consumers are idempotent because the relay and the broker are at-least-once.

## 4. Internal mechanisms

Outbox:

1. In one local transaction, update the Task and insert an outbox row (event ID, type, payload reference, aggregate sequence).
2. Commit.
3. Relay reads committed rows in sequence order, publishes, then marks them sent.
4. Crash after publish and before the mark: republish. Consumer dedupes.

Event sourcing:

1. Validate the command against current state.
2. Append the event.
3. Apply the event to state only after the append succeeds. Akka's persistence guide states this order: commands are not replayed; persisted events are.
4. Snapshots accelerate rebuild. They are not the source of truth.

CQRS:

1. Command side owns validation and the log or task table.
2. A projection builds the read model from events.
3. Queries hit the projection.
4. The projection may lag. Status that must be authoritative stays on the command side.

**Interpretation.** A Router's task table is the command model. A UI session list can be a projection. Observability dashboards are projections. If they disagree, the task table wins. That preserves "observability is a read-only projection."

## 5. Architecture and execution flow

```text
Ingress event
  dedupe
  command: accept / ignore / create task
  transaction: task row + outbox
relay
  bus (JetStream or equivalent)
consumers
  Worker executes
  Recorder appends trajectory
  Telemetry projection updates aggregates
```

The bus is not the only store. If the bus is the event log, the task row is a projection of that log and must say so. Two sources of truth will diverge.

## 6. Benefits

- The crash window between "task committed" and "nobody was told" closes, within the limits Richardson states.
- Replay supports audit and projection rebuild, which is why the skill separates Recorder from Telemetry.
- Read models can be shaped for UI and metrics without complicating transitions.
- Independent consumers can add a new projection without rewriting the Router.

## 7. Trade-offs

- Outbox relay duplicates are mandatory to handle, not an edge case.
- Event sourcing makes simple corrections hard: you add a compensating event, you do not update history in place. Fowler's shipping example is about capturing change, not about convenient edits.
- CQRS adds a consistency delay and more moving parts. Fowler says this has put systems into serious difficulties.
- Global ordering across aggregates is not provided by an outbox that only orders one aggregate's events. Richardson's pattern states ordering for messages the service sent, especially per aggregate sequence.

## 8. Failure modes

- Dual write without an outbox.
- Relay and Router both executing side effects.
- Projection treated as the authorization source.
- "Event sourced" systems that store mutable snapshots and delete the log.
- Using the broker as a workflow engine and losing the task state machine.

## 9. Mature implementations

- Richardson outbox.
- Akka Persistence event sourcing and projections mentioned in that guide. Projection internals were not re-read.
- NATS JetStream as a durable bus. It does not implement the outbox transaction for you.
- Temporal history is a durable execution log, not a general domain-event bus. Do not conflate them. See [durable execution](../distributed-systems/durable-execution.md).

## 10. Agent Runtime mapping

| Pattern | Use it for | Do not use it for |
|---|---|---|
| Event | Facts: message received, job finished, cancel requested | Permission grants |
| Outbox | Task transition plus `task.assigned` or `job.completed` | Plugin-private writes into the host table |
| Event sourcing | Recorder trajectory, audit, projection rebuild | Every setting toggle |
| CQRS | Status views, telemetry aggregates | The legal task transition function |
| Bus | Fan-out to Worker, Recorder, projections | A second hidden control plane |

**Trigger** is an event selected for evaluation, not a second copy of the event with different authority. **Subscription** still does not grant **Capability**.

Conflict: CQRS read models are eventually consistent. Authorization and "is this task cancelled" must be read from the command side or from a projection that is documented as synchronous enough for that decision. A stale projection must not allow an effect after cancel.

## 11. When to use

Use an outbox as soon as a task commit must wake another process. Use a durable bus when consumers restart. Add event sourcing for trajectories you will actually replay. Add a separate query model when the UI or metrics shape is distorting the task ledger.

## 12. When not to use

Do not introduce CQRS for a single process with one status table. Do not event-source plugin configuration. Do not replace the task state machine with "whatever the last event on the bus was" unless the bus is explicitly the ledger.

## 13. Original references

| Title | Author / org | URL | Date | Type | Concepts | Why it matters |
|---|---|---|---|---|---|---|
| Event Sourcing | Martin Fowler | https://martinfowler.com/eaaDev/EventSourcing.html | 12 December 2005; author says the page is unfinished draft | Article | Append events, rebuild state | Primary accessible statement of the pattern, with its draft status |
| CQRS | Martin Fowler | https://martinfowler.com/bliki/CQRS.html | 14 July 2011 | Article | Separate command and query models, caution | When not to apply the pattern |
| Pattern: Transactional outbox | Chris Richardson | https://microservices.io/patterns/data/transactional-outbox.html | retrieved 2026-11-03 | Pattern catalog | Same-transaction publish intent, at-least-once relay | Closes the dual-write gap without 2PC |
| Event Sourcing | Akka | https://doc.akka.io/libraries/akka-core/current/typed/persistence.html | retrieved 2026-11-03 | Official doc | Persist then apply, replay events not commands | Implementation order of command versus event |
| JetStream | NATS | https://docs.nats.io/nats-concepts/jetstream | retrieved 2026-11-03 | Official doc | Durable stream, independent consumers | A bus is not an outbox |
| Event Sourcing pattern | Microsoft Azure Architecture Center | https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing | retrieved 2026-11-03; body not fully re-read | Official doc | Vendor pattern page | Reachable alternate. Do not treat unread sections as cited |

Microsoft's CQRS pattern page also responded: https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs. It was not the source of the cautions above; Fowler was.
