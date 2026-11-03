# Actors, mailboxes, and synchronization

Status: written 2026-11-03. **Verified** restates sources read in this pass. **Interpretation** is for Agent runtimes.

## 1. Problem statement

Concurrent agent work shares sessions, files, and external APIs. Locks around those resources deadlock or get held across model calls. A mailbox isolates mutable state so that one Worker applies one message at a time, while other Workers proceed. The mailbox does not, by itself, make external side effects safe.

## 2. Core concepts

**Verified.** Hewitt, Bishop, and Steiger, IJCAI 1973, propose a modular actor architecture based on one kind of object. Actors are invoked uniformly by sending messages. The paper's abstract states that the architecture does not have or need goto, interrupt, or semaphore primitives, and that control and data flow are inseparable: an actor plays a role according to a script when it receives a message.

**Verified.** The current Akka typed guide states that a message send does not transfer the sender's thread to the destination. Each actor has a mailbox, a behavior, messages, an execution environment, and an address. The guide describes this processing sequence: enqueue the message, mark the actor runnable if needed, a scheduler runs it, the actor takes the front message, updates internal state and sends further messages, then the actor is unscheduled. It states that at most one message is processed per actor at a time, so internal invariants do not need locks. It also states that Akka organizes actors in a parent-child tree and that the parent supervises failure.

**Verified.** Ray's actor documentation states that methods on different actors execute in parallel and methods on the same actor execute serially in submission order, sharing that actor's state. Each actor runs in its own process. Class variables are not shared across actor instances.

**Interpretation.** "Actor" in this skill means: addressable Worker, private state, serial processing of its mailbox, explicit messages for anything that would have been shared memory. It does not mean adopting Akka, Orleans, or the 1973 PLANNER embedding.

## 3. Design principles

- Worker context is isolated by default. Another task's transcript is not shared memory.
- One authoritative owner updates a Task at a time, or an explicit coordination protocol exists.
- Messages are either facts (Event), work (Task assignment), or instructions (Command). Do not overload one untyped dict for all three.
- Serial execution inside one actor prevents internal races. It does not prevent two actors from racing on the same file or API.
- Mailbox growth is a resource. An unbounded mailbox is an unbounded queue.

## 4. Internal mechanisms

Hewitt's paper treats many familiar objects (functions, data structures, processes) as behaviors reachable by message send. That is a unification claim from 1973, not a description of a production scheduler.

Akka's guide adds the mechanical picture modern runtimes actually implement:

1. sender enqueues and continues;
2. recipient later runs on a scheduler thread;
3. replies are new messages, not return values on the sender's stack;
4. failure has no shared call stack, so a parent supervisor decides restart or stop.

**Interpretation.** A race remains wherever two mailboxes can touch the same external resource. The fix is not a global lock. It is a single writer: one Task owner, a lease, or a storage system that rejects stale fencing tokens. See [leases and fencing](../distributed-systems/leases-fencing-retries.md).

Synchronization primitives still belong inside a Worker when it protects a tiny in-memory structure and never holds the lock across a model or network call. They are the wrong tool for cross-Worker task state.

## 5. Architecture and execution flow

```text
Event arrives at Router mailbox
  Router applies policy serially for that session
  Router enqueues a Task assignment to one Worker
Worker mailbox
  validate lease
  run Operation
  send result message
  Router validates the state transition
```

The Router must not execute the Operation on its own thread. That is the existing control-plane rule, and it matches Akka's point that send does not transfer execution.

## 6. Benefits

- Internal Worker state avoids lock-based races.
- A slow Worker does not stop the Router from consuming its own mailbox.
- Failure is a message or a supervisor decision, not a torn shared stack.
- Location can later move from in-process to remote without changing the message contract, which Akka's guide claims as a reason the model maps onto both cache lines and network packets.

## 7. Trade-offs

- Serial actors stall if one message calls a slow tool. The tool belongs in a separate Operation or process, with the actor only holding coordination state.
- Lost or duplicated messages reintroduce the races actors removed, unless delivery and idempotency are designed. See [delivery](../distributed-systems/delivery-consistency-idempotency.md).
- Parent supervision that silently restarts a Worker looks healthy while repeating a side effect.
- Millions of actors, which Akka's guide describes as a goal, are irrelevant to a single-user assistant and expensive to observe.

## 8. Failure modes

- Shared dict of "current tasks" mutated by every plugin.
- Request-response hidden inside the mailbox by blocking the Router until the Worker replies.
- Assuming same-actor serial order applies to async or threaded actors. Ray documents that async or threaded actors can run tasks out of order.
- Using actor restart as recovery while the mailbox still contains the command that caused the bad side effect.

## 9. Mature implementations

- Akka typed actors and supervision. See [Akka](../frameworks/akka.md).
- Erlang processes and `gen_server`, which formalize the receive loop. See [Erlang/OTP](../frameworks/erlang-otp.md).
- Orleans virtual actors, which add activation and identity. See [Orleans](../frameworks/orleans.md).
- Ray actors as stateful worker processes. See [Ray](../frameworks/ray.md).

These are not interchangeable. Orleans virtual actors are always addressable and activated on demand. Akka actors are created and stopped explicitly. Ray actors are processes with resource requirements and, by default, no restart.

## 10. Agent Runtime mapping

- **Router** is an actor-like control owner for session coordination. It does not own tool execution.
- **Worker** is the isolated mailbox for a Task or a tightly bounded group.
- **Event** is an immutable message. **Command** asks for a state change. **Task** is the work record, not a message type by itself.
- **Plugin** may send documented events and commands through registries. It must not reach into another actor's private state.
- **INTERCEPT** handlers run on the control path and must stay synchronous and bounded. They are not a place to hide a second mailbox drain.
- **ASYNC** handlers are observers and must not block the producer.

Conflict: a framework that lets any actor create unsupervised children conflicts with "plugins must not create unmanaged background loops." If a plugin uses an actor library, the host registry still tracks those processes and cancels them on shutdown.

## 11. When to use

Use a mailbox per Session router and per Worker when more than one task can be in flight, or when a plugin must not share mutable host objects.

## 12. When not to use

Do not introduce a distributed actor cluster for a single process that only needs a mutex around an in-memory status map. Do not use actors to avoid defining Event, Task, and Command. Do not share mutable plugin state through class variables and call it an actor system. Ray's own actor page warns that class-level state stays local to the process and is the wrong way to share across actors.

## 13. Original references

| Title | Author / org | URL | Date | Type | Concepts | Why it matters |
|---|---|---|---|---|---|---|
| A Universal Modular ACTOR Formalism for Artificial Intelligence | Carl Hewitt, Peter Bishop, Richard Steiger | https://ijcai.org/proceedings/1973/31 | IJCAI, August 1973, pp. 235–245 | Paper | Uniform message send, actors as the unit of behavior | Primary source for the model. Page 31 is the IJCAI bibliographic record; PDF: https://ijcai.org/proceedings/1973/0031.pdf |
| How the Actor Model Meets the Needs of Modern, Distributed Systems | Akka | https://doc.akka.io/libraries/akka-core/current/typed/guide/actors-intro.html | current typed guide; retrieved 2026-11-03 | Official doc | Mailbox, serial processing, supervision tree | Concrete runtime mechanics, not the 1973 paper |
| Actors | Ray | https://docs.ray.io/en/latest/ray-core/actors.html | retrieved 2026-11-03 | Official doc | Process per actor, serial methods, cancellation limits | Shows a process-backed actor with different failure defaults |

Reading limit: the 1973 PDF text available from IJCAI is an OCR rendering. This note uses the abstract and the message-passing claim, not the paper's PLANNER logic embedding. Agha's 1986 actor semantics were not re-read.
