# Erlang/OTP

Status: written 2026-11-03 from the OTP design principles and the `supervisor` manual. Armstrong's 2003 thesis PDF was reachable and was not re-read.

## Problem it actually solves

OTP structures long-running systems as trees of processes so that crashes are detected and restarted under an explicit policy, and so that common process shapes are callbacks instead of one-off receive loops.

## Core abstractions

**Verified.**

- **Worker**: process that does the work.
- **Supervisor**: process that monitors workers and restarts them according to strategy.
- **Supervision tree**: hierarchy of both.
- **Behaviour**: generic module (`gen_server`, `gen_statem`, `gen_event`, `supervisor`) plus a callback module.
- **Application**: component, either a library or a supervision tree, with a directory structure.
- **Release**: a bootable set of applications. Release handling upgrades a running system. The upgrade procedure was not re-read.

The design principles show a `gen_server`-style split: the generic loop receives messages, the callback `handle_call` / `handle_cast` updates state. The protocol stays hidden behind client functions.

## Execution model

Processes communicate by messages. The simplified server in the design principles receives one message at a time in `loop`. Supervisors start children and react to termination. Strategies on the current manual page:

- `one_for_one`: restart only that child. Documented as the default.
- `one_for_all`: terminate and restart all children.
- `rest_for_one`: restart the child and those started after it.
- `simple_one_for_one`: dynamic instances of one child type. The manual still documents it, including invalid `restart_child` for that strategy.

Restart intensity and period stop a crash loop. Child restart types include permanent (always) and temporary (never). The manual also documents a third restart type in the child specification; quote that page rather than this note if you need the exact atom.

## Persistence model

OTP design principles as read here do not define workflow history. Mnesia is named as a database application. Its durability guarantees were not re-read. Process state dies with the process unless a callback stores it somewhere.

## Failure handling

Let the process terminate and let the supervisor apply the strategy, within intensity limits. Isolation is the process boundary. This does not record whether an external message was sent. That gap is an interpretation: the manuals read here talk about process restart, not effect receipts.

## Limitations

- Restart duplicates any side effect the process performed after its last durable save.
- `one_for_all` couples failures.
- A supervision tree inside a plugin can hide loops from the host.
- The thesis was not used. Do not attribute quotes to Armstrong from this note.

## Agent Runtime mapping

| OTP | This skill |
|---|---|
| Supervisor | Runtime lifecycle and Watchdog escalation into restart |
| Worker process | Worker or tool process |
| `gen_server` callback | Plugin or Operation handler |
| Intensity | Retry budget |
| Application | Plugin package with a manifest, not a second runtime |

Conflict: "let it crash" is about process state, not about chat messages or shell commands. Pair restarts with the intent/receipt log. See [supervision](../architecture-patterns/supervision-watchdog-reconciliation.md).

## References

| Title | Author / org | URL | Date | Type |
|---|---|---|---|---|
| OTP Design Principles | Ericsson / Erlang | https://www.erlang.org/doc/system/design_principles.html | docs labeled v29.1.2 at retrieval | Official doc |
| supervisor | Erlang/OTP | https://www.erlang.org/doc/apps/stdlib/supervisor.html | retrieved 2026-11-03 | Official doc |
| Making reliable distributed systems in the presence of software errors | Joe Armstrong | https://www.erlang.org/download/armstrong_thesis_2003.pdf | 2003; PDF not re-read | Thesis |
