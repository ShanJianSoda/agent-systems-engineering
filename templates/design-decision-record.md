# ADR: <title>

Status: proposed | accepted | superseded
Date:
Supersedes:
Superseded by:

## Problem

What failure or requirement forces a decision? Do not start from a framework name.

## Constraints

Which of these apply: durability, delivery, authorization, scheduling bounds, plugin boundary, single-host versus cluster? Note any conflict with the Agent Systems Engineering skill instead of weakening the constraint in place.

## Alternatives

At least two. Include the smallest mechanism that could work.

### Alternative A

- What it is:
- Failure mode it prevents:
- Failure mode it accepts:
- Operational cost:
- Evidence (doc path or URL actually read):

### Alternative B

- What it is:
- Failure mode it prevents:
- Failure mode it accepts:
- Operational cost:
- Evidence (doc path or URL actually read):

## Decision

Chosen alternative and why the rejected one lost.

## Consequences

- What becomes durable:
- What is still ambiguous after a crash:
- What must be idempotent:
- What the watchdog does:
- What was not verified:

## Follow-up

Tests required. Link a failure analysis if a test disproves this ADR.
