# Processes, threads, scheduling, and isolation

Status: written 2026-11-03. **Verified** statements restate sources read in this pass. **Interpretation** statements are consequences for Agent runtimes and are not vendor claims.

## 1. Problem statement

An Agent runtime must run several pieces of work at once without letting one stuck tool call, plugin, or model request freeze the interaction path or corrupt another task. Operating systems already separate that problem into execution units, a scheduler, and isolation boundaries. Copying thread pools into an agent without those distinctions produces shared-memory races and unclear ownership.

## 2. Core concepts

**Verified.** Linux `sched(7)` documents that Linux provides preemptive multitasking and several scheduling policies. A thread is the schedulable entity. Processes and threads are the units the kernel runs, preempts, and accounts.

**Verified.** Linux `namespaces(7)` documents namespaces as a mechanism for isolating global system resources so that a set of processes sees its own isolated instance of those resources. Linux `cgroups(7)` and the kernel cgroup v2 administrator guide document control groups as a way to organize processes hierarchically and distribute or limit resources such as CPU, memory, and I/O.

**Verified.** The Erlang/OTP design principles distinguish workers, which perform the work, from supervisors, which monitor workers. That is a user-space process tree, not a kernel scheduler, but it uses the same idea: a parent is responsible for the lifecycle of children.

**Interpretation.** For this skill, a Worker is closer to an isolated execution context with a mailbox than to a raw OS thread. An OS thread is an implementation option inside a Worker. It is not the architectural boundary.

## 3. Design principles

- The runtime owns lifecycle. A prompt must not be the only thing that starts, stops, or accounts a Worker.
- Isolation is a boundary with a stated mechanism: process, container, namespace, cgroup, or language-level actor. A working directory or a blocked-command list is not that boundary.
- Scheduling policy is explicit: priority, concurrency limit, and what happens when the machine is saturated.
- Preemption exists at the OS level, but application cancellation still has to be cooperative where the work holds external resources.

## 4. Internal mechanisms

**Verified.** Under preemptive scheduling, the kernel can stop a running thread and run another. That prevents one CPU-bound thread from permanently owning a core, but it does not make shared data structures safe. Two threads can still interleave writes.

**Verified.** cgroup v2 controllers apply limits to a hierarchy of processes. A limit stops a subtree from consuming unbounded CPU or memory. It does not understand task semantics such as "this tool call is stuck but the process is still runnable."

**Verified.** Namespaces change what a process can see (for example its own mount, PID, or network view, as documented by `namespaces(7)`). They are a visibility boundary. They are not an authorization system for model-visible tools unless the runtime actually places the tool process inside them.

**Interpretation.** Agent resource isolation should stack these mechanisms only as far as the threat requires:

1. logical isolation: separate task context and capability set;
2. process isolation: separate address space for untrusted tools;
3. namespace and cgroup isolation: separate filesystem/network view and resource budget for plugins or repository tools.

## 5. Architecture and execution flow

```text
Router (control)
  assigns Task to Worker slot under a concurrency budget
    Worker process or actor
      runs one Operation at a time, or a documented pool
        tool subprocess
          optional namespaces + cgroup limits
Scheduler
  kernel schedules threads inside those processes
Watchdog
  observes both OS liveness and task progress
```

The kernel scheduler decides which thread runs on a core. The runtime scheduler decides which Task is allowed to exist, which Worker owns it, and when to cancel it. Mixing those two decisions is how a system ends up with unbounded queues and no one responsible for progress.

## 6. Benefits

- A stuck tool can be killed without killing the Router.
- CPU and memory caps stop one plugin from starving interaction.
- Separate address spaces make accidental shared-memory mutation harder.
- Priority can keep user-facing work ahead of background reconciliation.

## 7. Trade-offs

- Processes cost more to start and to communicate with than threads or actors.
- cgroups and namespaces are Linux mechanisms. A portable runtime needs a host-specific adapter, not a pretence that every deployment has them.
- Preemption does not release locks, file leases, or external API side effects held by the preempted code.
- Over-isolating every model call adds latency that may dominate short tasks.

## 8. Failure modes

- Thread pool shared by interaction and tools: one blocking call occupies every interaction thread.
- Forking tools without a process group: cancellation kills the parent and leaves grandchildren.
- cgroup limit hit with no runtime reaction: the kernel OOM-kills or throttles, and the task stays `running` forever.
- Assuming a container is a security boundary when the runtime mounts the host docker socket or the parent credentials into it.

## 9. Mature implementations

- Linux scheduler, namespaces, and cgroups, as cited below.
- Erlang supervision trees structure lifecycle in user space. See [Erlang/OTP](../frameworks/erlang-otp.md).
- Ray places actors in worker processes and can require CPU or GPU resources. See [Ray](../frameworks/ray.md).

## 10. Agent Runtime mapping

| OS idea | Runtime concept | Constraint |
|---|---|---|
| Schedulable thread | Implementation detail inside a Worker | Runtime still owns scheduling |
| Process tree | Worker and tool subprocesses | Cancellation must reach the child |
| Namespace / cgroup | Capability-backed isolation for Plugin tools | Not a substitute for authorization |
| Run queue | Task queue with a bound | Saturation must reject, queue, coalesce, or degrade |
| Parent reaps children | Watchdog plus Worker lifecycle | Process exit is not task completion |

Router remains the control plane. Worker executes. Operation stays a thin wrapper around one bounded piece of work. Do not invent a second scheduler inside a Plugin.

## 11. When to use

Use process or container isolation when a tool runs repository commands, browser automation, or untrusted plugin code. Use a thread or actor only for trusted in-process work with no shared mutable runtime state.

## 12. When not to use

Do not start a new OS process per model token or per trivial policy check. Do not adopt a cluster scheduler because a single host has a thread pool. Do not treat cgroup limits as proof that a plugin cannot change policy or credentials if those secrets are still in its environment.

## 13. Original references

| Title | Author / org | URL | Date | Type | Concepts | Why it matters |
|---|---|---|---|---|---|---|
| sched(7) | Linux man-pages | https://man7.org/linux/man-pages/man7/sched.7.html | man-pages current; retrieved 2026-11-03 | Manual | Preemptive scheduling, policies, threads as schedulable entities | Separates kernel scheduling from application task scheduling |
| namespaces(7) | Linux man-pages | https://man7.org/linux/man-pages/man7/namespaces.7.html | retrieved 2026-11-03 | Manual | Resource visibility isolation | States what a process can see, which a tool sandbox must actually configure |
| cgroups(7) | Linux man-pages | https://man7.org/linux/man-pages/man7/cgroups.7.html | retrieved 2026-11-03 | Manual | Hierarchical resource control | Budget for plugin and tool processes |
| Control Group v2 | Linux kernel docs | https://docs.kernel.org/admin-guide/cgroup-v2.html | retrieved 2026-11-03 | Official doc | cgroup v2 hierarchy and controllers | Current kernel interface for limits |
| OTP Design Principles | Ericsson / Erlang | https://www.erlang.org/doc/system/design_principles.html | docs v29.1.2 at retrieval | Official doc | Workers and supervisors | User-space lifecycle tree analogous to a Worker host |

Reading limit: this note did not re-read a textbook treatment of CFS internals. Do not cite specific CFS vruntime formulas from this page.
