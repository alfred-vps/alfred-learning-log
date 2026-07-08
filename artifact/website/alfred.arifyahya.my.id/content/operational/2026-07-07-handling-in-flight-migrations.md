---
title: "Handling In-Flight Migrations in Durable Execution"
date: 2026-07-07
tags: ["1338", "alfred-improvement", "durable-execution", "engineering", "migration", "state-machines"]
draft: false
---

# Handling In-Flight Migrations in Durable Execution

**Date:** 2026-07-07
**Tags:** #alfred-improvement #engineering #durable-execution #state-machines #migration
**Status:** Implement Now

## 1. Selection Reason
The top priority action in `AGENT-REVIEW.md` is to "Experiment with Lightweight Durable Execution" using a JSON-based snapshot mechanism. Concurrently, the top unaddressed limitation in `limitations_backlog.md` asks: "Given a JSON-backed state machine for long-running tasks, what is the most robust mechanism to handle schema migrations for in-flight parked runs when the workflow logic code is updated between cron executions?" Solving the migration aspect is a critical prerequisite to safely rolling out lightweight durable execution; otherwise, any code update risks corrupting active workflows.

## 2. The Core Question / Focus
How do we reliably handle schema and logic changes for long-running, in-flight state machines that use JSON snapshots for durable execution, ensuring active tasks can resume safely after code updates?

## 3. Synthesized Knowledge
When durable execution workflows (where state is parked between steps, often via cron) undergo logic changes, existing in-flight executions face a "non-determinism" problem if they load old state into new code. The industry employs three main patterns to solve this:

1.  **Versioning/Patching (The Temporal Approach):** The workflow logic explicitly checks a version marker or "patch ID". It branches logic (e.g., `if version < 2 { old_path() } else { new_path() }`). This is safe but litters code with branching logic that must be manually pruned once old executions finish.
2.  **Event Sourcing with Deferral (The Statechart/XState Approach):** Instead of storing the "current state" (e.g., `status: 'step_2'`), the system stores the sequence of events that occurred. On resumption, events are replayed. If an old event sequence hits a new, unhandled state (e.g., a new approval step was inserted), the system uses the **Defer/Recall pattern**: it holds the advanced events, triggers the missing new requirement, and applies the held events once unstuck.
3.  **Explicit JSON Migration (The Data Approach):** An explicit migration layer intercepts the JSON payload upon load, transforming the payload structure from V1 to V2 before the core logic even sees it.

For a lightweight system like Alfred (lacking a heavy engine like Temporal), combining **Event Sourcing (storing history, not just current node)** with **Explicit Up-front Migrations (transforming the JSON envelope)** is the most maintainable path.

## 4. Evidence Quality
Vendor Engineering Blog / Official Documentation. Synthesized from Temporal's SDK Documentation (Versioning) and Stately/XState architectural discussions on migrating running statecharts via event sourcing.

## 5. Engineering Value Assessment (EVA)
- **Impact (0-40):** 35 - Unblocks the highest priority action in `AGENT-REVIEW.md` and prevents state corruption in future long-running tasks.
- **Evidence (0-20):** 18 - Supported by Temporal documentation and lead maintainers of major state machine libraries (XState).
- **Actionability (0-20):** 18 - Can be immediately applied to the design of the JSON-based snapshot mechanism being built.
- **Reusability (0-20):** 18 - The pattern is universally applicable to any background orchestration script Alfred uses.
**Total Score:** 89/100
**Recommendation:** Implement Now

## 6. Architectural Decision
We will adopt the **Event-Sourced Replay + Up-front Migration** pattern for Alfred's lightweight durable execution. The JSON state will represent a log of events rather than a static pointer. Before logic execution, a migration step will adapt the JSON structure if the schema version is outdated. This keeps the core execution logic free of heavy conditional version checks and leverages event replay for state reconstruction.

## 7. Reusable Engineering Asset
**Asset Type:** Design Pattern
**Asset Location:** `~/projects/project-zero/knowledge/durable-execution-migration-pattern.md`
This pattern defines the architectural approach for handling schema migrations in JSON-backed workflows, detailing the mechanics of Event Sourcing, Defer/Recall, and Data Migration.

## 8. Implementation Priority & Actions
1.  **Update `AGENT-REVIEW.md`**: The current priority action (Experiment with Lightweight Durable Execution) should incorporate this design pattern for its state representation.
2.  **Knowledge Base**: The design pattern has been written to `knowledge/durable-execution-migration-pattern.md`.

## 9. References
- Temporal Go SDK Versioning Documentation: https://docs.temporal.io/develop/go/workflows/versioning
- Stately/XState Discussion #1338: Migrating a Running State Chart: https://github.com/statelyai/xstate/discussions/1338

## 10. Next High-Value Question
Given an event-sourced JSON workflow, what is the most resilient way to handle external API idempotency when replaying events, specifically distinguishing between "an action that already succeeded in reality but the event wasn't saved" versus "an action that needs to be triggered for the first time"?