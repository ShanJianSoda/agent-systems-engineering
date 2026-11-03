# Design case: a plugin that cannot become a second runtime

Status: written 2026-11-03. Engineering comparison for the plugin boundary.

## Problem

A third party wants to add tools and a background watcher. The host must keep lifecycle, authorization, and task state. The plugin author will reach for an actor toolkit or a queue worker if the host API is vague.

## Alternatives actually compared

1. **Pass the host object into the plugin.** Fastest API and no boundary. Reject.
2. **Run the plugin as an unsupervised Akka or Erlang node inside the process.** Supervision and mailboxes are real mechanisms, but an embedded node that can spawn processes and write freely violates "no unmanaged loops" and "no private-state mutation."
3. **Host-owned registries and capabilities, OTP-style callbacks.** The host owns the loop, as OTP behaviours do. The plugin supplies manifest, tools, INTERCEPT and ASYNC handlers, and namespaced storage. Untrusted code runs in a separate process with a cgroup and a narrow token.

## Recommendation

Use alternative 3. Actors or Celery workers may exist inside the plugin process if the host started them, tracks them, and cancels them on `stop`. They are an implementation detail of the plugin, not a second Router.

## What must be true

- Missing capability fails closed.
- INTERCEPT overrun is a plugin fault, not a delayed ingress.
- ASYNC work uses host task slots or is rejected.
- Tests cover start failure, stop, dispose, and a handler that tries to write another plugin's state.
- The third-party library is not vendored into the host core.

## Conflict with framework habits

Akka parents supervise children they create. If the plugin is the parent, the host is blind. Invert it: the host supervisor is the parent, the plugin is a callback. See [plugins](../architecture-patterns/plugins-and-capabilities.md) and [Erlang/OTP](../frameworks/erlang-otp.md).

## Sources

- https://www.erlang.org/doc/system/design_principles.html
- https://doc.akka.io/libraries/akka-core/current/typed/guide/actors-intro.html
- https://man7.org/linux/man-pages/man7/cgroups.7.html
- https://homes.cs.washington.edu/~levy/capabook/ (book home only; chapters not re-read)
