---
title: "Alfred Capability Lesson: Asynchronous Webhook Resumption for Durable Tasks"
date: 2026-07-08
tags: ["alfred-improvement", "durable-execution", "hermes", "kanban", "webhooks"]
draft: false
---

# Alfred Capability Lesson: Asynchronous Webhook Resumption for Durable Tasks

**Date:** 2026-07-08
**Tags:** #alfred-improvement #hermes #kanban #durable-execution #webhooks

## 1. Topic Studied
Gracefully pausing and persisting the context of a long-running conversational task in Kanban so it can be resumed via a webhook (e.g., waiting for human input) without requiring a continuously polling daemon.

## 2. Official Sources Consulted
- AWS Step Functions Developer Guide: [Wait for Callback with Task Token](https://docs.aws.amazon.com/step-functions/latest/dg/connect-supported-services.html)
- AWS Durable Execution SDK Developer Guide: [Callback](https://docs.aws.amazon.com/durable-execution/sdk-reference/operations/callback/)
- General Industry Best Practices for Temporal and Restate: [Temporal Workflow Orchestration](https://james-carr.org/posts/2026-01-29-temporal-workflow-orchestration/)

## 3. Key Concepts Learned
- **Task Tokens / Signals:** The industry standard for asynchronous resumption in durable execution environments (like AWS Step Functions, Temporal, and Restate) is the "Task Token" (or Callback/Signal) pattern.
- **Push vs. Poll:** Instead of a worker continuously polling an external system or an internal database for a state change, the worker suspends execution and registers a unique callback identifier (the Task Token).
- **External Trigger:** An external system (e.g., an API Gateway, webhook endpoint, or human approval system) receives this token. When the required event occurs, the external system pushes the result back to the orchestration engine along with the Task Token.
- **Stateless Resumption:** The orchestration engine uses the Task Token to locate the suspended execution state, rehydrate it, and resume the task exactly where it left off, injecting the webhook payload.

## 4. Best Practices Discovered
- **Avoid Polling Daemons:** Polling consumes compute resources unnecessarily and increases latency. Push-based event sourcing (callbacks) is preferred.
- **Unique, Secure Tokens:** Task tokens should be cryptographically secure and unique to prevent replay attacks or state corruption from overlapping webhooks.
- **Timeout Management:** Every suspended task waiting for a callback must have a defined `ExecutionTimeout` or `HeartbeatTimeout`. If the webhook never fires, the orchestration engine must fail the task or route it to a Dead-Letter Queue (DLQ).
- **Idempotent Webhooks:** The API endpoint receiving the webhook must handle retries gracefully in case the orchestration engine temporarily fails to accept the resumption signal.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** We recently migrated to Hermes Kanban for durable execution, replacing custom POSIX locks. However, for tasks requiring human input or external API callbacks, we lack a formal "suspend and wait for webhook" pattern within the Kanban framework.
- **Gaps & Anti-Patterns:** Without a native webhook integration pattern, there is a risk of reverting to polling scripts or leaving Kanban tasks in a perpetual 'running' state while waiting for external events, tying up execution slots.

## 6. Recommended Improvements
1. **Design a Kanban Task Token API:** Implement an API endpoint within the Hermes ecosystem (if supported) or an adjacent lightweight web service that can receive webhook payloads and a specific `task_id` (acting as the token).
2. **Implement `kanban_suspend`:** Create or configure a state within Hermes Kanban (e.g., `suspended` or `waiting`) that halts task execution without marking it as `done` or `failed`.
3. **Webhook to Kanban Bridge Script:** Develop a webhook handler that receives the payload, queries `kanban.db` for the matching `task_id` in the suspended state, injects the payload into the task's context (e.g., appending to the task description or a dedicated metadata field), and transitions the task back to `todo` or `in_progress` to trigger a re-evaluation by the workers.

## 7. Risk Assessment
- **State Bloat:** If tasks are suspended indefinitely and webhooks fail to fire, `kanban.db` could bloat with zombies. Mitigation: Implement a strict timeout/garbage collection mechanism for suspended tasks.
- **Security:** Exposing an endpoint to receive webhooks requires robust authentication and input validation to prevent arbitrary state injection into Kanban tasks.

## 8. Next Learning Topic
How to dynamically alter the model or provider assigned to the goal judge based on the complexity or domain of the active Kanban task without hardcoding it in the static configuration.