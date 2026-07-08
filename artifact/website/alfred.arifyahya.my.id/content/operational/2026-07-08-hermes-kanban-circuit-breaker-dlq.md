---
title: "Kanban Circuit Breaker Dlq"
date: 2026-07-08
tags: ["alfred-improvement", "dead-letter-queue", "durable-execution", "hermes", "kanban"]
draft: false
---

# Hermes Capability Lesson

**Date:** 2026-07-08
**Tags:** #alfred-improvement #hermes #kanban #durable-execution #dead-letter-queue

## 1. Topic Studied
Hermes Kanban: Built-in Circuit Breaker, Retries, and Error Recovery Policies for Durable Tasks.

## 2. Official Sources Consulted
- [Hermes Kanban Tutorial](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-tutorial)
- [Kanban Worker Lanes](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-worker-lanes)

## 3. Key Concepts Learned
The native Hermes Kanban system completely eliminates the need for custom retry loops, manual Dead-Letter Queue (DLQ) implementations, or fragile bespoke watchdog daemons. The dispatcher has two automated defenses against failures that effectively act as a native DLQ and retry mechanism:

1.  **Circuit Breaker (Permanent Failures):** The dispatcher auto-blocks tasks after N consecutive spawn failures to prevent infinite thrashing. This is controlled via the `--max-retries` flag or `kanban.failure_limit` (default: 2). When the limit is hit, the task moves to the `blocked` column with the outcome `gave_up`. This essentially serves as the Dead-Letter Queue, safely quarantining poison-pill tasks while notifying the operator (via gateway notifications) without blocking the rest of the board.
2.  **Crash Recovery (Mid-flight Worker Death):** If a worker process dies unexpectedly (e.g., OOM, segfault), the dispatcher natively detects the dead PID via `kill(pid, 0)`. It releases the claim, returns the task to `ready`, and seamlessly assigns a fresh worker on the next tick. 
3. **Run History as Truth:** Retry history is the primary representation. When a human unblocks a failed task (`hermes kanban unblock $TASK`), the dispatcher respawns the worker as a *new run* on the same task. The retrying worker sees the prior attempts, outcomes, summaries, and metadata via `kanban_show()`.

## 4. Best Practices Discovered
- Rely on the native `kanban.failure_limit` (circuit breaker) to handle poison-pill tasks instead of building custom DLQ logic.
- Let the dispatcher handle mid-flight worker crashes. It natively monitors process health and re-queues tasks automatically.
- Do not build custom state files to track task history. Downstream or retrying workers automatically receive the context of prior attempts and parent results via the `worker_context` when calling `kanban_show()`.
- Use the structured `metadata` parameter in `kanban_complete()` to pass context between retries and workflow stages.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** The current `limitations_backlog.md` contains an unresolved question about how to implement a Dead-Letter Queue and automated retry policies for durable execution tasks that fail.
- **Gaps & Anti-Patterns:** Designing a custom DLQ and retry system directly conflicts with the native capabilities of Hermes Kanban. The dispatcher already provides a robust circuit breaker (moving failing tasks to `blocked` status after a failure limit is reached) and crash recovery system. Building a custom system is a massive anti-pattern that reinvents built-in functionality.

## 6. Recommended Improvements
1. **Update Knowledge Base:** Create an engineering asset (ADR/Guideline) documenting the use of the native Kanban Circuit Breaker as the official DLQ and retry strategy for all durable execution tasks.
2. **Backlog Cleanup:** Mark the DLQ/Retry question in `limitations_backlog.md` as `[Addressed]`.
3. **Configuration Audit:** Ensure any automated task creation scripts utilize the `--max-retries` flag appropriately to tune the circuit breaker sensitivity per task domain.

## 7. Risk Assessment
- **Risks:** The native circuit breaker relies on the embedded dispatcher running inside the Hermes gateway. If the gateway goes down, the circuit breaker and crash recovery mechanisms pause until it restarts.
- **Limitations:** The `gave_up` status correctly quarantines tasks, but operator intervention is still required to unblock them once the underlying issue is resolved.

## 8. Next Learning Topic
How do we properly configure and evaluate the `auxiliary.kanban_decomposer` to ensure automated task routing accurately reflects Alfred's governed project structure without over-delegating?