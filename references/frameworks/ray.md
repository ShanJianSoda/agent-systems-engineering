# Ray

Status: written 2026-11-03 from the OSDI 2018 paper's front matter and model sections, the current actors guide, and the actor fault-tolerance page. Scheduler internals and current cluster security defaults were not fully re-read. A docs banner on retrieval mentioned token authentication changes in Ray 2.59 and 2.61; that banner is not part of the architecture analysis.

## Problem it actually solves

The OSDI paper presents Ray as a cluster framework for emerging AI applications, especially reinforcement learning, that need both stateless fine-grained tasks and stateful actors, dynamic dependencies, heterogeneous hardware, and millisecond-scale tasks at high fanout. The authors argue that stitching separate training, simulation, and serving systems together was too slow for those loops. Current docs present actors as stateful worker processes.

## Core abstractions

**Verified** from the paper's programming-model section and Table 1:

- `f.remote` submits a task and returns a future.
- `ray.get` waits for values. `ray.wait` returns when k futures complete or a timeout hits.
- `Class.remote` creates an actor. Methods return futures.
- Tasks are stateless and placeable, which helps load balancing and lineage recovery.
- Actors hold local state, which helps fine-grained updates and wrapping simulators that cannot be serialized.
- A distributed scheduler and a fault-tolerant metadata store hold control state. Other components are described as stateless. Lineage reconstructs tasks. The paper says actors also have lineage-based fault tolerance in that design. Current docs, below, put application checkpointing on the user. Treat 2018 and current docs as different layers, not as one behavior.

**Verified** from current actor docs:

- An actor is a dedicated worker process. Methods on one actor run serially in submission order and share that process's state. Different actors run in parallel.
- Resources can be requested (`num_cpus`, `num_gpus`).
- `ray.cancel` is best-effort. Sync actors should poll `is_canceled()`. Async actors use asyncio cancellation. The docs say cancellation is not guaranteed if the request never reaches the executor.
- Class variables are not shared across actor processes.

## Execution model

Tasks run on a worker pool. Actors stick to one process. Nested remote calls are part of the paper's model. Current cancellation and retry behavior is per actor options, not an implicit durable workflow.

## Persistence model

**Verified** on the fault-tolerance page: default `max_restarts` is 0. Default actor tasks are at-most-once (`max_task_retries=0`). The page warns that `RayActorError` can be raised even if the task actually finished, for example if the actor dies immediately after success. Restarts rerun `__init__` and do not restore fields unless the constructor loads a checkpoint. The application owns that checkpoint. If the checkpoint is on external storage it must be visible to whichever node restarts the actor.

Non-detached actors die with their owner process even if `max_restarts` is set. Detached actors do not have that owner fate-sharing. The page says detached actors still stop when the cluster is destroyed or restarts are exhausted.

## Failure handling

Optional at-least-once (`max_task_retries` greater than 0 or -1) can run a method twice, once on the failed actor and once after restart. The page says previously completed tasks are not re-executed, with a stated exception when retries are in play for object reconstruction. Async or threaded actors may run out of order, and only incomplete tasks are retried. For critical state the page says at-least-once fits ephemeral or read-only actors, and critical actors need application checkpoints.

`ActorUnavailableError` means the call may or may not have run. The page's guidance is to stop sending traffic and to retry only with idempotent methods if `retry_exceptions` is enabled. Default is not to retry user exceptions.

## Limitations

- Defaults are not durable execution.
- Owner fate-sharing surprises anyone who expected a cluster actor to outlive the client.
- Duplicate execution is possible when retries are on, and lost certainty is possible when they are off.
- The 2018 paper's throughput figures are for that system and workload. They are not agent runtime requirements.
- Ray does not provide agent policy, session authorization, or a task ledger.

## Agent Runtime mapping

| Ray | This skill |
|---|---|
| Actor process | Worker isolation boundary |
| Task future | Operation handle |
| `ray.wait` | Bounded wait, then cancel stragglers |
| Checkpoint | Explicit durable state, still needs receipts |
| `max_restarts` | Supervisor policy, default off |
| Detached actor | Host-tracked background work, not an unmanaged plugin loop |

Use Ray for heterogeneous compute fan-out. Put the task ledger and fences outside Ray unless a newer Ray API, re-read at implementation time, provides the durability you can name precisely. See [durable task case](../design-cases/durable-task-loop.md).

## References

| Title | Author / org | URL | Date | Type |
|---|---|---|---|---|
| Ray: A Distributed Framework for Emerging AI Applications | Philipp Moritz, Robert Nishihara, Stephanie Wang, Alexey Tumanov, Richard Liaw, Eric Liang, Melih Elibol, Zongheng Yang, William Paul, Michael I. Jordan, Ion Stoica | https://www.usenix.org/system/files/osdi18-moritz.pdf | OSDI, 8–10 October 2018 | Paper |
| Actors | Ray | https://docs.ray.io/en/latest/ray-core/actors.html | retrieved 2026-11-03 | Official doc |
| Actor Fault Tolerance | Ray | https://docs.ray.io/en/latest/ray-core/fault_tolerance/actors.html | retrieved 2026-11-03 | Official doc |
