# NATS JetStream

Status: written 2026-11-03 from the JetStream concepts page. Clustering, exactly-once publish options, and KV implementation were not re-read.

## Problem it actually solves

Core NATS delivers a message only to subscribers connected at publish time, at most once, without replay. JetStream keeps a stream so consumers can restart, replay, and acknowledge. Publishers and consumers do not have to be online together.

## Core abstractions

**Verified.**

- **Stream**: store of messages for one or more subjects, with a sequence number. Storage can be memory or file. Retention is configurable, including whether limits or consumer acknowledgements remove messages.
- **Consumer**: server-side view of a stream with its own cursor. Many consumers can read one stream independently.
- **Client**: connects, receives, and acknowledges. Shared consumers can divide work. The page's examples use a durable pull consumer and explicit ack.
- If a message is not acknowledged in time, the server redelivers it. The page calls this at-least-once delivery.
- Start position can be the beginning, the latest message, a sequence, or a time.
- **Key Value** and **Object Store** are higher-level features on top of streams. Their consistency details were not re-read.

## Execution model

Publish appends to the stream. A pull consumer fetches a bounded batch. The application acks each message. Redelivery happens on a missed ack. Nothing on this page runs user workflow code inside the server.

## Persistence model

File or memory storage on the server, plus optional replication mentioned as a stream setting. Replica semantics were not re-read. The stream is a log, not an application task table.

## Failure handling

Consumer crash: unacked messages are redelivered. That duplicates processing unless the application dedupes. Publisher crash after the server appends is the publisher's uncertainty; this page does not document a transaction with an external database.

## Limitations

- At-least-once is not exactly-once processing.
- JetStream will not atomically commit your task row and the publish. Use an outbox if that coupling matters. See [outbox](../architecture-patterns/event-driven-outbox-cqrs.md).
- A memory stream does not survive a server restart. The examples that need durability set file storage.
- Ordering is per stream sequence, not a global order across subjects you did not bind into that stream.
- It is a bus. It is not a supervisor, a lease manager, or an agent policy engine.

## Agent Runtime mapping

| JetStream | This skill |
|---|---|
| Message | Event envelope |
| Stream sequence | Source cursor |
| Durable consumer | Router or Recorder subscription |
| Explicit ack | Handler finished the durable local commit |
| Pull max messages | Backpressure bound |
| KV | Not a substitute for the task ledger unless an ADR says so after reading KV docs |

Conflict: two consumers that both execute effects will double-apply. One consumer executes. Others may record or project. See [delivery](../distributed-systems/delivery-consistency-idempotency.md).

## References

| Title | Author / org | URL | Date | Type |
|---|---|---|---|---|
| JetStream | NATS | https://docs.nats.io/nats-concepts/jetstream | retrieved 2026-11-03 | Official doc |
