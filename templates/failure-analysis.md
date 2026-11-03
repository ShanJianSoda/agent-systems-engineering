# Failure analysis

Date:
System or prototype:
Severity:

## Symptom

What was observed, and by whom? Separate the user-visible symptom from the suspected cause.

## Boundary

Where did it happen: ingress, Router, queue, Worker, Operation, external effect, plugin, projection?

## Timeline

Include crash points: before intent, after intent, after effect, before receipt, after lease expiry.

## Evidence

Task version, event ID, receipt, logs, traces. Mark guesses as guesses.

## Classification

- [ ] Transient transport
- [ ] Ambiguous timeout after a possible external effect
- [ ] Deterministic reject or bad input
- [ ] Saturated resource
- [ ] Stale Worker or missing fence
- [ ] Process dead
- [ ] Process alive but not progressing
- [ ] Authorization or prompt injection
- [ ] Duplicate delivery
- [ ] Observability-only (ledger was correct)

## Why restart or retry is or is not safe

State whether a second attempt can repeat a message, file write, or payment.

## What will change

One fix, one test that reproduces the injection point, and who owns it. Do not propose a new platform before the missing receipt or missing test is addressed.

## What remains unknown
