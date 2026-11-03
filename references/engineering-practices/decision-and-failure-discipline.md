# Decision records and failure discipline

Status: written 2026-11-03. This note is engineering practice for using the templates in this repository. It does not claim a new standard.

## 1. Problem statement

Architecture choices in an agent runtime are easy to reverse in a diagram and expensive to reverse once tasks are in production. Teams forget which failure mode a pattern was meant to stop, then copy a framework because its name appeared in a blog. A written decision and a written failure analysis keep the next change attached to evidence.

## 2. Core concepts

An architecture decision record (ADR) captures one significant choice: the problem, the alternatives, the decision, and the consequences. A failure analysis captures one incident or one designed failure drill: trigger, boundary, evidence, and what will change.

Significant means any choice that affects durability, authorization, delivery semantics, scheduling bounds, or the plugin boundary. Renaming a local variable is not significant. Choosing Temporal versus a task table is significant.

## 3. Design principles

- Identify the problem before naming a pattern. "We need tasks to survive restart" is a problem. "We need event sourcing" is a candidate.
- Compare at least two alternatives. One of them should be the smallest mechanism that fits the current deployment, not only the most famous framework.
- State operational cost: new processes, storage growth, on-call signals, and what a crash does.
- State failure modes the choice accepts, not only the ones it prevents.
- Label verified facts, vendor recommendations, and local interpretation. Do not write "Temporal is exactly-once" if the sources do not say that.
- If a pattern conflicts with a skill constraint, record the conflict. Do not edit the constraint silently.

## 4. Internal mechanisms

Use [templates/design-decision-record.md](../../templates/design-decision-record.md) when the decision will outlive the chat. Use [templates/failure-analysis.md](../../templates/failure-analysis.md) after a real incident or a recovery test that failed. Use [templates/architecture-review.md](../../templates/architecture-review.md) before implementing a new runtime surface.

Keep ADRs next to the design they govern, in the host repository. This skill repository only ships the templates.

## 5. Architecture and execution flow

```text
Problem statement
  read the reference map, open only the matching notes
  draft two or more options
  cost and failure review
  ADR if the choice is significant
  smallest vertical slice
  failure-path test
  update the ADR with what the test actually showed
```

## 6. Benefits

- Later changes can see why a fence or an outbox exists.
- Reviews have a checklist instead of taste.
- Framework copying becomes visible because the rejected alternative is written down.

## 7. Trade-offs

- Documents rot if they are not updated when the test contradicts them.
- Too many ADRs hide the few decisions that matter. Use the significance test above.

## 8. Failure modes

- ADR written after the code, repeating the code's structure as if it were a comparison.
- Failure analysis that lists action items with no owner and no test.
- Citing a paper that was not opened.

## 9. Mature implementations

The templates are local. The discipline of recording trade-offs is illustrated by Fowler's CQRS article, which spends as much space on when not to use the pattern as on the pattern itself: https://martinfowler.com/bliki/CQRS.html (14 July 2011).

## 10. Agent Runtime mapping

Decisions about Router, Worker, Event, Task, Session, Operation, leases, plugins, Recorder, and Observability use the existing names. Do not introduce a parallel vocabulary in the ADR.

## 11. When to use

Use an ADR for durability, delivery, isolation, and plugin boundary choices. Use a failure analysis when a recovery or security test finds a hole, and after production incidents.

## 12. When not to use

Do not write an ADR for every helper function. Do not file a failure analysis that proposes a new platform before the incident's missing receipt or missing test is fixed.

## 13. Original references

| Title | Author / org | URL | Date | Type | Concepts | Why it matters |
|---|---|---|---|---|---|---|
| CQRS | Martin Fowler | https://martinfowler.com/bliki/CQRS.html | 14 July 2011 | Article | Explicit non-goals for a famous pattern | Model for "when not to use" |
| Design decision record | this repository | [template](../../templates/design-decision-record.md) | 2026-11-03 | Template | Alternatives, costs, conflicts | Local form |
| Failure analysis | this repository | [template](../../templates/failure-analysis.md) | 2026-11-03 | Template | Evidence versus guess | Local form |
| Architecture review | this repository | [template](../../templates/architecture-review.md) | 2026-11-03 | Template | Skill checklist | Local form |
