# Observability: traces, metrics, and logs

Status: written 2026-11-03. **Verified** restates sources read in this pass. **Interpretation** is for Agent runtimes.

## 1. Problem statement

When a task is slow or wrong, the operator needs to know whether the delay is the model, the queue, a tool, or a retry loop. A second event bus that "monitors" by consuming and republishing changes becomes another writer and changes the system it measures. Observability has to be a read-only projection over identifiers the runtime already has.

## 2. Core concepts

**Verified.** OpenTelemetry describes signals as outputs that describe activity. The signals page lists traces, metrics, logs, and baggage as supported, with profiles under development. The traces concept page was reachable; this note uses the signals page for the category list and does not restate API field names that were not read.

**Verified.** The Google SRE monitoring chapter, written by Rob Ewaschuk and edited by Betsy Beyer, distinguishes monitoring definitions used at Google: white-box versus black-box, symptom versus cause, and the four golden signals latency, traffic, errors, and saturation. It warns that mean latency hides tail latency, and recommends histogram buckets. It says alerts that page a human should be simple, urgent, actionable, and tied to user-visible impact. It also says not to merge every kind of inspection (profiling, logs, load tests) into one fragile system.

**Interpretation.** For this skill:

- Logs are records of specific events, correlated by IDs.
- Metrics are aggregated numbers, including the golden signals plus agent-specific series (queue depth, token use, decision reuse).
- Traces are the path of one task across Router, Worker, and tools.
- Recorder is the durable trajectory and is not the same as a debug log.
- Telemetry is the derived aggregate.
- Observability reads. It does not accept commands that change tasks.

## 3. Design principles

- Every log, metric, and span carries correlation ID, event ID, task ID, session ID, worker ID, plugin ID, and tool or effect ID when those exist.
- Symptom metrics page or repair. Cause metrics debug. SRE's table is the test: HTTP 500s are a symptom; a refused database connection is a cause.
- Track error latency separately from success latency, as the SRE chapter insists, because fast failures distort averages.
- Saturation includes queue depth and token or concurrency budgets, not only CPU.
- Dashboards read projections. They do not subscribe as a second consumer that acknowledges work.
- Drop high-cardinality labels such as raw prompts. They cost money and leak data.

## 4. Internal mechanisms

Emit points, all best-effort relative to the task commit:

1. Ingress: accepted, dropped, duplicate, reason.
2. Queue: depth, wait, reject.
3. Worker: start, progress, stall.
4. Operation: tool latency, model latency, tokens, error class.
5. Effect: retry, ambiguous receipt, dead letter.
6. Watchdog: action taken, human escalation.

The task commit must not wait on the metrics backend. Losing a metric is acceptable. Losing a receipt is not. If tracing export fails, the Recorder still has the trajectory.

**Interpretation.** Baggage, in OpenTelemetry's list, is context propagated across signals. Use it for correlation IDs, not for authorization grants. A grant that travels inside a trace header will be logged by sidecars.

## 5. Architecture and execution flow

```text
Runtime state and Recorder
  exporters (read)
    logs / metrics / traces
      dashboards and alerts
Watchdog reads metrics and task progress
  may act through the runtime API
  does not write telemetry back into the task as if telemetry were the ledger
```

## 6. Benefits

- One ID connects a user report to a tool failure.
- Golden signals catch saturation before a narrative postmortem.
- Tail histograms show a few stuck tools under a healthy mean.
- Read-only export keeps the control plane singular.

## 7. Trade-offs

- Full tracing of every model token is expensive and sensitive. Sample payloads, keep IDs.
- Histogram buckets must be chosen. The SRE chapter's factor-of-three example is a starting point, not a requirement.
- Symptom alerts lag the root cause on purpose. You still need white-box series for diagnosis.
- Two exporters can double-count if both scrape the same event without a documented source.

## 8. Failure modes

- Alert on every retry. Pages become noise, which SRE says leads people to ignore real pages.
- Mean latency only.
- Observability consumer that acks the work queue.
- Logs that contain secrets or full capability tokens.
- Health field written by both Watchdog and Plugin.

## 9. Mature implementations

- OpenTelemetry signal model.
- SRE golden signals and symptom-first paging.
- Temporal's documented UI and CLI for inspecting Event History, plus metrics for Prometheus-compatible systems, from the understanding page. That UI reads history; it is not the history service.

## 10. Agent Runtime mapping

| Signal | Runtime source | Writer |
|---|---|---|
| Trace | Task and Operation spans | Exporter, read-only |
| Metric | Counters and histograms | Telemetry projection |
| Log | Structured lines with IDs | Runtime and plugins through a log API |
| Trajectory | Recorder | Runtime, append-only |
| Health snapshot | Watchdog | Single writer per family |
| User status | Task projection | Command side or a documented projection |

Plugins may emit telemetry only through the host API, tagged with plugin ID. They must not open a side channel that mutates tasks.

## 11. When to use

Add correlated logs and the four golden signals, plus queue and token series, before adding a tracing vendor. Add traces when a task crosses more than one process.

## 12. When not to use

Do not build a custom distributed tracing database inside the agent. Do not alert on cause metrics that are not actionable. Do not store prompts in a metrics label.

## 13. Original references

| Title | Author / org | URL | Date | Type | Concepts | Why it matters |
|---|---|---|---|---|---|---|
| Signals | OpenTelemetry | https://opentelemetry.io/docs/concepts/signals/ | retrieved 2026-11-03 | Official doc | Traces, metrics, logs, baggage | Names the signal types without inventing a schema |
| Traces | OpenTelemetry | https://opentelemetry.io/docs/concepts/signals/traces/ | retrieved 2026-11-03; body not fully re-read | Official doc | Request path | Landing page confirmed. Field-level claims are not taken from it |
| Monitoring Distributed Systems | Rob Ewaschuk, Betsy Beyer (ed.), Google SRE | https://sre.google/sre-book/monitoring-distributed-systems/ | retrieved 2026-11-03 | Book chapter | Golden signals, symptom versus cause, paging | Alert and metric design |
| Understanding Temporal | Temporal | https://docs.temporal.io/evaluate/understanding-temporal | retrieved 2026-11-03 | Official doc | UI, CLI, history, metrics export | Inspection beside a durable log |
