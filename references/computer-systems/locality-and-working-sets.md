# Locality, working sets, and context as a cache

Status: written 2026-11-03. **Verified** restates sources read in this pass. **Interpretation** is for Agent runtimes. This note does not cite a working-set paper, because a stable full text was not re-read in this pass.

## 1. Problem statement

Model context is small, expensive, and latency-sensitive. Stuffing every memory, file, and prior tool result into each call is the equivalent of bypassing every cache and fetching all of memory for every instruction. The runtime needs a working set: what must be inline now, what can be spilled behind a reference, and what is cold.

## 2. Core concepts

**Verified.** The Akka actor guide argues that actor state stays local and that messages propagate changes, which "maps to how modern memory hierarchy actually works": often only the cache lines containing the message move, while local state stays cached on the original core. The same shape maps to remote machines, where state stays in that machine's memory and packets carry changes.

**Verified.** PostgreSQL's WAL chapter notes that sequential WAL writes are much cheaper than flushing randomly touched data pages, especially when many small transactions touch different parts of the store. Access pattern matters, not only byte count.

**Interpretation.** The skill's Inline / Spill / Locality rules are a cache policy for model context:

- Inline is the hot working set.
- Spill is the next level: durable objects addressed by ID.
- Locality means the next retrieval prefers the same task, thread, parent, and recent tool result before a global search.
- Chain of recall is pointer chasing with a hop budget, not a full scan.

No claim is made here about a specific CPU's cache-line size or about Denning's formal working-set theorem. Those details were not re-measured or re-read.

## 3. Design principles

- Context is a cache, not the database. Recorder and task storage are the database.
- Every cache entry has an ID, source, time, summary, and a pointer to the original.
- Retrieval has a token budget and a maximum hop count.
- Reuse a cached decision only while its guards hold: input fingerprint, policy version, authorization, and absence of a newer cancel.
- Batch cold work away from the interaction path. Do not batch extra tokens into the hot call to "be safe."

## 4. Internal mechanisms

Treat each model call as a read of a cache:

1. Start from the Task and Session IDs.
2. Include the smallest set of facts required to choose the next Operation.
3. If a reference is missing, fetch that object, not the neighborhood of the whole repository.
4. If the fetched object is large, inline a bounded summary and keep the body spilled.
5. Record what was inlined so the next call can reuse the selection when guards match.

**Interpretation.** Temporal locality: the latest tool result and the latest user instruction are hot. Spatial locality: adjacent messages and files touched by this task are more likely to be useful than similarly named files in another session. That is a retrieval heuristic, not a proof.

Akka's cache-line argument supports a related runtime choice: send a small message to the Worker that already holds the task state, instead of copying the whole session to every Worker.

## 5. Architecture and execution flow

```text
Session store (cold, durable)
  -> locality query by task, thread, recency
    -> working set (bounded)
      -> model call
        -> new references and summaries
          -> Recorder appends the trajectory
            -> next call rebuilds a new working set
```

Observability may report token counts and spill rates. It must not push extra context back into the call.

## 6. Benefits

- Lower latency and token cost on the interaction path.
- Restart can rebuild context from IDs instead of from a giant prompt blob.
- Workers stay isolated because they receive references, not the Router's entire memory.
- Decision reuse becomes auditable rather than "the prompt looked similar."

## 7. Trade-offs

- A too-small working set causes extra retrieval round trips and missed facts.
- Summaries can drop the constraint that mattered. The original reference has to remain reachable.
- Locality heuristics fail on cross-cutting requests ("what did we decide last month?"). Those need an explicit search Operation, not a larger default window.
- Caching a decision across a policy change is a security bug, not a performance win.

## 8. Failure modes

- Dumping all memory into every call until the window truncates the system prompt.
- Spill without an ID, so the model cannot ask for the original.
- Sharing one global context cache across tenants or sessions.
- Using speculative prefetch of secrets or other users' tasks because they are "nearby."

## 9. Mature implementations

This pattern is an application of storage and actor locality, not a single product. Related mechanisms:

- PostgreSQL's preference for sequential log writes over random page flushes, as a reminder that access pattern dominates cost.
- Akka's local-state-plus-message argument.
- Ray's OSDI paper distinguishes stateful actors, which update local state without shipping it, from stateless tasks that can be placed near inputs. See [Ray](../frameworks/ray.md). The paper's performance numbers are not reused here as agent benchmarks.

## 10. Agent Runtime mapping

- **Session** is the context boundary. Locality searches start inside it.
- **Task** is the hottest key.
- **Operation** receives a bounded view, not a license to reload the world.
- **Recorder** stores the trajectory that future recall walks.
- **Telemetry** may aggregate token and retrieval cost. It is not a context source.
- Fast path versus slow path is the decision cache: deterministic checks before a model call, matching the skill's guarded reuse rules.

## 11. When to use

Use inline/spill on every model call once transcripts, tool output, or repository files exceed a budget you are willing to pay on the interaction path.

## 12. When not to use

Do not build a hierarchical memory manager before you have measured a real overflow. A fixed budget and stable IDs are the first mechanism. Do not cite CPU cache papers as justification for a vector database.

## 13. Original references

| Title | Author / org | URL | Date | Type | Concepts | Why it matters |
|---|---|---|---|---|---|---|
| How the Actor Model Meets the Needs of Modern, Distributed Systems | Akka | https://doc.akka.io/libraries/akka-core/current/typed/guide/actors-intro.html | retrieved 2026-11-03 | Official doc | Local state, cache lines, message propagation | A concrete locality argument from an actor runtime |
| Write-Ahead Logging (WAL) | PostgreSQL | https://www.postgresql.org/docs/current/wal-intro.html | Documentation 18; retrieved 2026-11-03 | Official doc | Sequential versus random writes | Access pattern changes cost |
| Ray: A Distributed Framework for Emerging AI Applications | Moritz, Nishihara, Wang, et al. | https://www.usenix.org/system/files/osdi18-moritz.pdf | OSDI 2018, 8–10 October 2018 | Paper | Stateful actors versus stateless tasks, data locality | Why placement and statefulness differ. Intro and model sections read; not the full evaluation |

Unresolved: Peter Denning's working-set paper (CACM 1968) was not used. An automated request to the ACM landing page returned 403, and no full text was read. Do not invent its theorems from this note.
