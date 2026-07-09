---
title: "Handling In-Flight Migrations for Durable Execution State"
date: 2026-07-08
status: Approved
tags:
  - architecture
  - durable-execution
  - state-management
  - schema-migration
  - "curriculum:legacy"
draft: false
---

# Handling In-Flight Migrations for Durable Execution State

## 1. Context and Problem Statement
**"How do we safely version and migrate the schema of durable execution state files when the underlying processing logic changes mid-flight?"**

This question from the `limitations_backlog.md` highlights a critical risk in long-running agentic workflows (like those built on Hermes Kanban). If a cron job or agent is suspended for days, and the underlying codebase is updated during that time to expect a new JSON structure, the suspended task will crash upon resumption due to schema mismatch. 

As Alfred adopts a "Workflow over Reactive Agents" architecture (as dictated by the North Star), managing the lifecycle and evolution of persisted workflow state becomes mandatory.

## 2. Research Findings

*   **Evidence Quality:** High (Temporal Official Documentation, State Machine Design Patterns).
*   **Industry Standard (Temporal):** Enterprise systems like Temporal solve this by *not* migrating state. Instead, they use **Worker Versioning (Rainbow Deployments)**. They pin the workflow to the exact code version that started it. Old workflows run on old code until they finish.
*   **The Constraint:** Alfred runs locally as a single-instance cron scheduler. We do not have the infrastructure to effortlessly run multiple isolated codebases simultaneously (no rainbow deployments). We must upgrade in place.
*   **State Machine Evolution:** When evolving finite state machines in flight, best practice dictates decoupling the *deserialization* of state from the *execution* of state, applying a unidirectional transformation pipeline upon load.

## 3. Engineering Value Assessment (EVA)

*   **Impact (35/40):** High. Prevents catastrophic failures of long-running tasks across codebase updates. Essential for data integrity in Hermes Kanban.
*   **Evidence (18/20):** Strong mapping from enterprise distributed systems (Temporal) adapted to a single-node local constraint.
*   **Actionability (18/20):** Very actionable. Implementing a migration pipeline wrapper around `json.load()` for Kanban states is straightforward.
*   **Reusability (20/20):** This pattern applies to *any* persisted state in the Alfred ecosystem, not just Kanban.
*   **Total Score:** 91/100 (Implement Now).

## 4. Architectural Decision & Asset Generation

I have produced the architectural pattern document: `/home/hermes/projects/project-zero/knowledge/handling-in-flight-migrations.md`.

### Core Directives:
1.  **The Schema Sentinel:** All durable state files must enforce a `schema_version` integer at the root level.
2.  **Migration Pipeline on Load:** The execution engine must NEVER load raw state directly into business logic. State must pass through a sequential migration pipeline (e.g., `v1 -> v2 -> v3`) immediately upon read.
3.  **Strict Engine Requirements:** The execution engine is hardcoded to only accept the current maximum `schema_version`. If a state file cannot be migrated to the current version, execution is halted, and the task is moved to a dead-letter queue.
4.  **Continue-as-New for Infinite Loops:** Workflows that never terminate naturally must implement a "Continue-as-New" boundary to force a clean restart on the new schema rather than accumulating infinite migration debt.

## 5. Next Steps

The next immediate action for Alfred is to integrate this Schema Sentinel pattern into the existing Hermes Kanban tools to ensure state safety as we migrate away from custom POSIX locks.