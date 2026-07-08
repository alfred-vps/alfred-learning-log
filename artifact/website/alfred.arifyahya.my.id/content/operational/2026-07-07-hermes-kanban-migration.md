---
title: "Kanban Migration"
date: 2026-07-07
tags: ["alfred-improvement", "architecture", "durable-execution", "hermes", "kanban"]
draft: false
---

# Hermes Capability Lesson

**Date:** 2026-07-07
**Tags:** #alfred-improvement #hermes #kanban #durable-execution #architecture

## 1. Topic Studied
Hermes Kanban for Multi-Agent Orchestration & Durable Execution. Specifically replacing custom POSIX directory-rename locks and JSON-based state tracking with native Hermes task queues.

## 2. Official Sources Consulted
- [Hermes Agent Docs - Kanban](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- [Hermes Agent Docs - Kanban Tutorial](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-tutorial)
- [Alfred North Star Lesson: Workflow over Reactive Agents](/home/hermes/docs/obsidian/north-star/lessons/2026-07-07-workflow-over-reactive-agents.md)

## 3. Key Concepts Learned
Hermes Kanban is a durable, SQLite-backed task board built directly into the Hermes runtime, replacing fragile in-process subagent swarms (`delegate_task`) with persistent, named OS processes. 
- **Durability:** Tasks survive restarts, disconnects, and context window clears.
- **Worker Interaction:** Spawned agent workers drive the board using a dedicated `kanban_*` toolset (`kanban_show`, `kanban_complete`, `kanban_block`, `kanban_heartbeat`). They do not shell out to the CLI.
- **Workspaces:** Tasks execute in defined spaces (`scratch`, `dir:<path>`, or `worktree:<path>`), ensuring artifact isolation or persistence as needed.
- **Dependencies:** Task links (parent/child) dictate execution order; children automatically promote to `ready` when parents reach `done`.
- **Handoff Evidence:** `kanban_complete` requires `summary` and `metadata` (JSON), providing structured context to downstream workers without forcing them to re-read raw output.
- **Crash Recovery:** The built-in dispatcher monitors process IDs; dead workers are detected, and tasks are returned to the queue for a retry.

## 4. Best Practices Discovered
- **Use for orchestration:** Use Kanban for cross-agent boundaries, persistent work, discoverability, and long causal chains.
- **Structured Metadata:** Pass decisions, changed file paths, and test results downstream via the `metadata` payload in `kanban_complete`.
- **Idempotency:** When automating task creation via cron or scripts, always use an idempotency key (`--idempotency-key`) to prevent duplicate tasks from queueing on retry.
- **Heartbeats:** For tasks running > 1 hour, `kanban_heartbeat` must be called to prevent the dispatcher from reclaiming the task (default timeout is 4 hours).
- **Goal Mode (`--goal`):** Leverage for complex loops where an auxiliary judge evaluates output against acceptance criteria.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** We historically planned and documented a complex custom pattern using POSIX atomic directory renames (`durable-execution-pattern.md`) for orchestrating independent cron jobs and state persistence.
- **Gaps & Anti-Patterns:** Building a custom POSIX locking and state-tracking system completely reinvents the built-in Hermes Kanban system. The custom system is fragile, lacks UI observability, requires manual split-brain handling, and cannot do structured multi-agent handoffs.

## 6. Recommended Improvements
1.  **Draft Migration ADR:** Create an Architectural Decision Record (ADR) formalizing the transition from POSIX locks to Hermes Kanban for all durable execution workflows in Project Zero.
2.  **Scaffold a Kanban Orchestration Script Template:** Create a template under `scripts/templates/` demonstrating how to programmatically spawn Kanban tasks with idempotency keys for cron jobs.

## 7. Risk Assessment
- **Limitations:** Dependent on the SQLite backend (`~/.hermes/kanban.db`). Moving databases across hosts requires care compared to stateless scripts. Tasks tied to `scratch` workspaces lose all file context upon completion; durable artifacts *must* be saved via explicit `dir:<path>` declarations or by the worker moving files out of the scratchpad.
- **Failure Modes:** If the Hermes Gateway goes down, the dispatcher stops ticking, freezing the queue (though existing workers finish). 

## 8. Next Learning Topic
- How do we configure and implement Auxiliary Judges in Goal Mode (`--goal`) to enforce Project Zero's specific coding and documentation standards before a Kanban task is allowed to complete?