# Agent Systems Engineering Skill

Reusable skill for designing and implementing Agent runtimes, Super-agents, Harnesses, and plugins.

## Files

- `SKILL.md` — main skill instructions.
- `references/REFERENCE-MAP.md` — primary sources and computer-systems reading map.

## Install/use

Copy this directory into the skill directory supported by your coding agent, or provide `SKILL.md` as the project-level development instruction.

The skill is intentionally framework-agnostic. It guides architecture and engineering decisions; it does not prescribe a specific language, database, queue, or Agent framework.

## Scope

Covers:
- Router / Worker architecture
- Event-driven triggers and policy gates
- Async jobs and full-duplex interaction
- Durable state, checkpoints, and recovery
- Watchdog and task-progress supervision
- Context Inline / Spill and guarded decision reuse
- Plugin lifecycle, registries, capabilities, and compatibility
- Security, observability, testing, and design review
