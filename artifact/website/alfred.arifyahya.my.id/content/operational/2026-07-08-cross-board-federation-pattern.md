---
title: "Securely Federating Task Delegation Across Isolated Kanban Boards"
date: 2026-07-08
tags: ["alfred-improvement"]
draft: false
---

# Securely Federating Task Delegation Across Isolated Kanban Boards

## 1. Context and Problem Selection

**Topic:** How can we securely federate task delegation across multiple isolated Hermes Kanban boards (e.g., between independent governed projects) using a standardized cross-project API contract without tightly coupling their underlying `kanban.db` schemas?

**Selection Rationale:** 
Alfred's current priority action in `AGENT-REVIEW.md` is migrating to Hermes Kanban for multi-step, long-running tasks. Project Zero governs multiple independent projects, meaning Alfred must coordinate work across different project contexts. The `limitations_backlog.md` identified cross-board task federation as a critical question. Solving this ensures Alfred can orchestrate multi-project workflows deterministically without introducing database brittleness.

## 2. Research Findings

**Methodology:**
Searched official Hermes Agent documentation for Kanban cross-linking rules. Also researched general distributed systems patterns for cross-domain orchestration (Sagas).

**Evidence Quality:** 
Primary Docs (Hermes Agent Official Documentation).

**Key Discoveries:**
1. **Schema Isolation constraint:** The official Hermes Kanban documentation explicitly states: *"Linking tasks across boards is not allowed (keeps the schema simple; if you really need cross-project refs, use free-text mentions and look them up by id..."* 
2. **Distributed Workflow Alignment:** This constraint perfectly aligns with distributed microservice architecture principles (e.g., Saga patterns), where domains (or in this case, boards) maintain their own state and communicate via decoupled references rather than shared databases or foreign keys.
3. **Decoupled Orchestration:** Relying on free-text IDs allows a workflow running on one board to poll or react to the state of a task on another board without tight coupling.

## 3. Engineering Value Assessment (EVA)

*   **Impact (0-40):** 35. Unblocks the migration to Hermes Kanban for cross-project governance.
*   **Evidence (0-20):** 20. Backed by explicit Hermes documentation constraints.
*   **Actionability (0-20):** 20. Immediate. Requires only a change in convention (metadata formatting).
*   **Reusability (0-20):** 20. Applicable to all future multi-project workflows.
*   **Total Score:** 95 / 100
*   **Decision:** **Implement Now** (Standardize the pattern as a durable asset).

## 4. Capability Update

**Asset Produced:** 
Created the definitive pattern guide at `/home/hermes/projects/project-zero/knowledge/cross-board-federation-pattern.md`.

**Strategic Decision:** 
Adopt the **Free-Text Reference Pattern** for all cross-board dependencies. 

When delegating a task to another board:
1. Create the task on the target board.
2. Embed the reference in the parent task's metadata using the format: `Depends-On: [board_id]/[task_id]`.
3. Orchestration logic uses this text reference to query the target board's state asynchronously.

This prevents the anti-pattern of fighting the native tooling and ensures safe, decoupled data schemas across the Alfred ecosystem.