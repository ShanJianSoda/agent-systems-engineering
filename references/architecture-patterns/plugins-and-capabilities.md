# Plugins, registries, and capability boundaries

Status: written 2026-11-03. **Verified** restates sources read in this pass. **Interpretation** is for Agent runtimes. The classic capability literature was not re-read beyond a reachable book home page.

## 1. Problem statement

Plugins add tools, event handlers, and automations the host did not ship. If a plugin can touch private runtime objects, a prompt injection inside an event payload becomes a host compromise. A method call on a shared object grants whatever that object can do. A capability grant names the specific right a plugin may exercise.

## 2. Core concepts

**Verified.** Henry M. Levy's *Capability-Based Computer Systems* has a public book home page at the University of Washington. This pass confirmed the page is reachable and did not re-read the chapters. No mechanism below is claimed as a quotation from Levy.

**Verified.** The existing skill already defines the host contract: plugins register tools and handlers, subscribe and publish documented events, store namespaced state, and request capabilities declared in a manifest. They must not mutate private host state, patch internals, start unmanaged loops, bypass authorization, or write another plugin's storage. INTERCEPT handlers are synchronous and bounded. ASYNC handlers must not block the producer.

**Interpretation.** A capability here is an unforgeable reference the host issues, naming an operation and a scope: "this plugin may append files under this workspace" rather than "this plugin received a filesystem object." Possession of the event payload is not possession of the capability. That matches the skill's split between subscription and authorization.

## 3. Design principles

- Default deny. A plugin starts with the rights in its manifest and nothing discovered at runtime from message text.
- The host is the reference monitor. Tools call back into the host, which checks the capability, rather than receiving a raw client to the database or docker socket.
- Rights are specific and revocable. Unloading a plugin invalidates its capabilities.
- INTERCEPT code runs on the control path, so it cannot do unbounded I/O.
- ASYNC code can do slower work only after the host has accepted the event and only through granted capabilities.
- Third-party frameworks used inside a plugin stay behind the plugin boundary. They do not become the host runtime.

## 4. Internal mechanisms

Registration:

1. Read manifest: ID, version, host API range, capabilities, handlers, tools.
2. Fail closed if the host version or a required capability is missing.
3. Issue capability tokens bound to plugin ID and scope. Tokens are random or cryptographic, not the string `"admin"`.
4. Register handlers in the documented registries only.
5. On each call, the host checks token, scope, policy version, and cancellation.
6. On stop, cancel background work, drop tokens, then dispose.

**Interpretation.** Erlang applications and OTP behaviours are a mature version of "generic host plus callback module." The callback does not own the process loop. See [Erlang/OTP](../frameworks/erlang-otp.md). That is the right shape for a plugin. Giving the plugin the supervisor of the whole node is the wrong shape.

Prompt injection is an input problem and a capability problem. The payload can ask the model to call a tool. The host still checks the grant captured when the task was accepted. A plugin that forwards the payload into a shell without that check has no boundary, regardless of how polished its manifest looks.

## 5. Architecture and execution flow

```text
Plugin manifest
  host validates
  CapabilityRegistry issues scoped tokens
  EventHandlerRegistry records INTERCEPT and ASYNC handlers
Event
  INTERCEPT chain, bounded, may reject
  Router policy and grant
  ASYNC observers, non-blocking
Tool call
  host checks capability
  Operation runs
  plugin cannot write Task rows except through a command API
```

## 6. Benefits

- Extension without sharing the host's memory graph.
- Revocation has a single place.
- Compatibility ranges fail closed instead of half-loading.
- Audit can record which capability was used for which effect.

## 7. Trade-offs

- A narrow capability API is more work than passing the host object.
- INTERCEPT budgets will be violated unless the host enforces a timeout and treats overrun as failure.
- True OS capabilities (Levy's subject) are stronger than application tokens if the token can be copied as a string into logs. Treat logged tokens as compromised.
- Isolating every plugin in a process costs startup time. In-process plugins are acceptable only inside the same trust domain, and still must not receive raw host pointers.

## 8. Failure modes

- Manifest declares `filesystem`, and the host mounts the entire disk.
- Handler order undocumented, so one plugin's mutation of the event becomes another's input. Prefer annotations over silent mutation, and document it if mutation is allowed.
- Plugin opens its own database connection using host credentials found in the environment.
- "Capability" implemented as a boolean `trusted=true`.
- Reload that leaves the old background loop running.

## 9. Mature implementations

- OTP behaviours: generic server, state machine, event handler, supervisor. The callback is not the loop.
- Kubernetes controllers as extensions that talk to the API server rather than editing etcd. See [controllers](../frameworks/kubernetes-controllers.md).
- Orleans grain call filters for cross-cutting auth and telemetry, documented on the Orleans overview as incoming and outgoing filters. Filters are host pipeline, not plugin-owned mutable state. See [Orleans](../frameworks/orleans.md).

No claim is made that these products implement capability machines in Levy's sense.

## 10. Agent Runtime mapping

- **Plugin** extends through Registry, Event, Command, and Capability.
- **Capability** is the right to invoke. **Mandate / Grant** is the authorization captured for a task. Both are required. A capability without a grant, or a grant without a capability, does not execute.
- **INTERCEPT** may reject or annotate. It may not call a model or perform an effect.
- **ASYNC** may consume after the fact. It may not stall ingress.
- **Recorder** may be a plugin only if it appends through a granted command and cannot rewrite history.
- **Observability** plugins are read-only.

Conflict: actor frameworks encourage plugins to spawn processes. The host must still track them. A library that requires a singleton global runtime inside the plugin conflicts with "do not create a second runtime."

## 11. When to use

Use a manifest, registries, and capability checks as soon as code from more than one author runs in the agent. Use process isolation when the plugin is less trusted than the host.

## 12. When not to use

Do not build a capability operating system for a single-module prototype. Do not add a plugin API that takes the Router object "temporarily."

## 13. Original references

| Title | Author / org | URL | Date | Type | Concepts | Why it matters |
|---|---|---|---|---|---|---|
| Capability-Based Computer Systems | Henry M. Levy | https://homes.cs.washington.edu/~levy/capabook/ | book, 1984; page retrieved 2026-11-03 | Book home page | Capability-based architectures | Canonical monograph. Chapters were not re-read; do not quote mechanisms from this note as Levy's |
| OTP Design Principles | Ericsson / Erlang | https://www.erlang.org/doc/system/design_principles.html | docs v29.1.2 at retrieval | Official doc | Behaviour versus callback | Host owns the loop; extension supplies callbacks |
| Orleans overview | Microsoft | https://learn.microsoft.com/en-us/dotnet/orleans/overview | retrieved 2026-11-03 | Official doc | Grain call filters | A pipeline for auth and telemetry without handing out the runtime |

Unresolved: Mark Miller's *Robust Composition* thesis is often cited for object capabilities. The previously known PDF URL failed TLS verification in this pass and is not cited.
