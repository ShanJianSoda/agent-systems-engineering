# Reference map

This skill synthesizes system-design ideas for Agent runtimes and plugins. It is not an implementation of any single paper.

## Primary references

1. DeepSeek AI, `dsh-superagent-prototype`  
   https://github.com/deepseek-ai/dsh-superagent-prototype  
   Public research prototype. README explicitly says it is not production-ready and does not fully implement every design discussed in the article.

2. DeepSeek Harness, “Exploring Super-agent Design: Towards a More Autonomous Agent System”  
   Original article links supplied by the user:
   - https://mp.weixin.qq.com/s/zoS7GlReWPhfCjENGxXCdg
   - https://zhuanlan.zhihu.com/p/2099987320443237060

3. Roger-Lv, “Super-agent 设计探索：从架构理念到 dsh-superagent 原型实现”  
   https://roger-lv.github.io/2026/11/01/2026-11-01-Super-agent%E8%AE%BE%E8%AE%A1%E6%8E%A2%E7%B4%A2%EF%BC%9A%E4%BB%8E%E6%9E%B6%E6%9E%84%E7%90%86%E5%BF%B5%E5%88%B0dsh-superagent%E5%8E%9F%E5%9E%8B%E5%AE%9E%E7%8E%B0/

## Concepts worth studying

### Operating systems and concurrency
- Actor Model and message passing
- Event loops and asynchronous I/O
- Process supervision and watchdogs
- Thread pools, worker pools, scheduling
- Cancellation, deadlines, and cooperative shutdown
- Backpressure and bounded queues

### Distributed systems and data consistency
- CQRS and Event Sourcing
- Transactional Outbox
- Idempotency and at-least-once delivery
- Leases, fencing tokens, and optimistic concurrency
- Checkpointing and durable execution
- Retry budgets, exponential backoff, circuit breakers
- Reconciliation after ambiguous external side effects

### Computer architecture and context efficiency
- Cache hierarchy and locality
- Working sets and cache eviction
- Batching and throughput/latency trade-offs
- Fast Path / Slow Path
- Speculative execution, only for reversible low-risk work

### Security and extensibility
- Capability-based security
- Least privilege and isolation
- Plugin manifests and versioned extension APIs
- Untrusted input and prompt-injection boundaries
- Audit trails and policy versioning

## Reading discipline

- Read the DeepSeek article for the intended architecture and design rationale.
- Read the repository docs and source for what is actually implemented.
- Treat third-party explainers as secondary analysis.
- Distinguish “article proposal”, “public prototype implementation”, and “production recommendation”.
- Re-check repository state before relying on specific filenames, constants, APIs, or behavior.
