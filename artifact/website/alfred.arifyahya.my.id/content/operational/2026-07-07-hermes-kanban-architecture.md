---
title: Hermes Capability Lesson
date: 2026-07-07
tags: [alfred-improvement, hermes, kanban, durable-execution, architecture]
draft: false
---

## 1. Topic Studied
Hermes Kanban and its Goal Mode / Auxiliary Judges for durable, multi-agent orchestration.

## 2. Official Sources Consulted
- Primary Docs: Hermes Kanban (Multi-Agent Board) Summary
- Primary Docs: Kanban tutorial
- Primary Docs: Kanban worker lanes

## 3. Key Concepts Learned
- **Architecture over Action:** Kanban replaces transient RPC calls (`delegate_task`) with a durable, SQLite-backed (`~/.hermes/kanban.db`) message queue where tasks are stateful rows. 
- **Decoupled Execution:** Tasks exist in `triage | todo | ready | running | blocked | done`. The Gateway embeds a Dispatcher that maps `ready` tasks to specific profile lanes (workers) based on the `assignee` field.
- **Worker Lifecycles:** Workers interact with the DB via native Python tools (`kanban_show`, `kanban_heartbeat`, `kanban_complete`, `kanban_block`), NOT by shelling out to the CLI. 
- **Dependency Engines & Resiliency:** Child tasks stay in `todo` until parents are `done` (promoting to `ready`). The dispatcher provides circuit breakers (max retries) and crash recovery by checking `kill(pid, 0)` for mid-flight worker deaths. 
- **Goal Mode & Orchestration:** Orchestrators break down tasks (`kanban_create`, `kanban_link`) but do not execute them. Goal Mode (`--goal`) enables a multi-turn loop where an auxiliary judge evaluates the output against acceptance criteria.

## 4. Best Practices Discovered
- **Structured Handoffs:** Use `kanban_complete(summary=..., metadata=...)` to pass explicit, structured context (e.g., changed files, decisions) to downstream dependent tasks. Do not rely on digging through unstructured comments.
- **Review Patterns:** Code-changing tasks should use `kanban_block(reason="review-required: ...")` to gate completion on human review, passing structured metadata via `kanban_comment` prior to blocking.
- **Anti-Temptation Orchestrators:** Orchestrator profiles should be configured with `kanban` tools but explicitly *denied* implementation tools (`file`, `terminal`, etc.) so they are forced to delegate rather than executing work themselves. 
- **Heartbeats:** Long-running tasks must call `kanban_heartbeat()` periodically. Tasks running >4 hours without a heartbeat in the last hour are marked stale and reclaimed.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** We historically planned/implemented a custom, fragile POSIX directory-rename locking pattern and JSON-backed state files for durable execution (noted in `AGENT-REVIEW.md`).
- **Gaps & Anti-Patterns:** 
  - Using custom POSIX locks and JSON files for state tracking violates the core Hermes capability (Kanban), which natively handles SQLite durability, crash recovery, and multi-agent queues. 
  - Relying on `delegate_task` for long-running workflows risks memory loss and lack of audit trails compared to Kanban's durable queue.

## 6. Recommended Improvements
1. **Deprecate Custom Durable Execution Patterns:** Formally retire any documentation or scripts advocating for POSIX locks or custom JSON state files in favor of Hermes Kanban. 
2. **Draft a Kanban Adoption Guide:** Create a standard blueprint for migrating existing cron jobs and workflows to Kanban tasks.
3. **Configure the Decomposer:** Ensure the `auxiliary.kanban_decomposer` is properly tuned for Alfred's specific profiles to enable auto-orchestration of triaged tasks.

## 7. Risk Assessment
- **Migration Friction:** Moving existing scripts/crons to rely on `hermes gateway start` and Kanban requires a mindset shift from procedural scripting to state machine orchestration.
- **Database Centralization:** Relying entirely on `~/.hermes/kanban.db` creates a single point of failure; requires robust backup strategies for the SQLite file.
- **Over-Delegation:** Risk of orchestrators generating too many granular child tasks, cluttering the board and wasting compute.

## 8. Next Learning Topic
How to properly configure and evaluate the `auxiliary.kanban_decomposer` to ensure automated task routing accurately reflects Alfred's governed project structure.
