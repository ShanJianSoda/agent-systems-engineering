# Agent Systems Engineering Skill

Reusable skill for designing and implementing Agent runtimes, Super-agents, Harnesses, and plugins.

The skill is intentionally framework-agnostic. It guides architecture and engineering decisions; it does not prescribe a language, database, queue, or Agent framework.

## Three layers

1. **Skill** — `SKILL.md` holds the workflow, architectural constraints, and review checklist. Use it without loading the rest of the library.
2. **Reference library** — `references/` holds principles, primary sources, and design patterns. Enter through `references/REFERENCE-MAP.md` and open only the notes that match the problem.
3. **Implementation cases** — `references/frameworks/` analyzes specific systems. `references/design-cases/` compares alternatives for this skill's architecture. Neither layer is a dependency to vendor into a runtime.

## Layout

- `SKILL.md` — instructions and constraints.
- `references/REFERENCE-MAP.md` — retrieval index, prior art, and research gaps.
- `references/computer-systems/` — processes, cancellation, actors, WAL, locality.
- `references/distributed-systems/` — delivery, leases, durable execution.
- `references/architecture-patterns/` — events, supervision, plugins, observability, backpressure.
- `references/engineering-practices/` — decisions, failure review, and tests.
- `references/frameworks/` — Temporal, Kubernetes controllers, Erlang/OTP, Akka, Orleans, NATS JetStream, Ray, Celery.
- `references/design-cases/` — worked comparisons that stay inside the skill's constraints.
- `templates/` — architecture review, decision record, and failure analysis.

## Install

Copy this directory into the skill directory supported by your coding agent, or provide `SKILL.md` as the project-level development instruction.

## Scope

The skill covers Router / Worker architecture, event-driven triggers, asynchronous jobs, durable state, watchdogs, context locality, plugins, security, observability, and design review. The reference library explains the systems mechanisms behind those constraints and records where a framework would conflict with them.
