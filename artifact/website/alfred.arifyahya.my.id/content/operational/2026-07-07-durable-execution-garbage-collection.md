---
title: "Semantic Garbage Collection for JSON-Backed Durable Execution"
tags: ["#alfred-improvement", "#engineering", "#durable-execution", "#garbage-collection"]
date: 2026-07-07
status: "Implement Now"
draft: false
---

# Semantic Garbage Collection for JSON-Backed Durable Execution

## 1. Selection Reason
The second limitation in the `limitations_backlog.md` asks: "How can we implement a generic garbage collection mechanism for stale durable execution states to prevent disk clutter without risking the deletion of genuinely paused long-term tasks?" Since Alfred is actively experimenting with lightweight JSON-based durable execution (identified as the top priority in `AGENT-REVIEW.md`), unbounded disk accumulation of state files and the silent failure of zombie workflows are guaranteed future friction points. Addressing this completes the lifecycle management of the lightweight durable execution pattern.

## 2. The Core Question / Focus
How do we reliably identify and clean up durable execution state files (JSON snapshots) for multi-step cron tasks without accidentally deleting valid, long-running processes that are just waiting (e.g., waiting for external events or user input)?

## 3. Synthesized Knowledge
Standard garbage collection based solely on file age (e.g., `mtime > 30 days`) is extremely dangerous for durable workflows because a workflow might legitimately wait for 31 days. Industry-standard workflow engines (like Temporal and Airflow) and Dead Letter Queue (DLQ) architectures solve this through **Semantic Garbage Collection**:

1.  **Retention Periods for Terminal States:** When a workflow finishes (`completed`, `failed`, `cancelled`), it is not deleted immediately. It is retained for a fixed audit window (e.g., 7 days) to allow developers to inspect the outcome. Once `completed_at + 7 days` is reached, it is safely purged.
2.  **Heartbeats and Dead Letter Queues (DLQ) for Stalled States:** If a workflow is in an active state (`running`) but hasn't updated its state file (heartbeat) in a threshold (e.g., 30 days), it is considered a "zombie" or "stalled". Instead of deleting it (which destroys the context of the failure), it is moved to a Dead Letter Queue (DLQ) directory. This removes it from the active polling loop but preserves it for operator intervention.
3.  **Explicit "Wait" States:** Workflows that legitimately sleep for long periods must explicitly declare a `status: "waiting"` state. The GC mechanism ignores this state or applies a significantly longer threshold.

By tying GC to the *semantic state machine* rather than file system metadata, we guarantee no data loss for valid workflows.

## 4. Evidence Quality
Primary Documentation / Architecture Patterns. Synthesized from Temporal's retention policies, Airflow's AIP-103 (task state garbage collection), and standard enterprise Dead Letter Queue (DLQ) design patterns for message brokers and workflow engines.

## 5. Engineering Value Assessment (EVA)
- **Impact (0-40):** 30 - Prevents future disk bloat and provides a safe observability path for failed/zombie cron jobs.
- **Evidence (0-20):** 20 - Direct alignment with enterprise workflow engine architecture (Temporal/Airflow).
- **Actionability (0-20):** 18 - Very simple to implement as a generic Python script across any JSON-backed workflow.
- **Reusability (0-20):** 20 - Universally applicable to any asynchronous background processing script Alfred manages.
- **Total Score:** 88
- **Recommendation:** Implement Now

## 6. Architectural Decision
Alfred will adopt **Semantic Garbage Collection with DLQ Fallback** for all lightweight durable execution scripts. Every state JSON file must strictly include `status` and `last_updated_at`. Terminal states will be retained for a 7-day audit window before deletion. Stalled active states will be moved to a DLQ directory rather than deleted.

## 7. Reusable Engineering Asset
**Asset Type:** Design Pattern / Implementation Guide
**Asset Location:** `/home/hermes/projects/project-zero/knowledge/durable-execution-garbage-collection.md`
This asset defines the semantic rules, states, and a reference Python implementation for safely garbage collecting workflow JSON files.

## 8. Implementation Priority & Actions
1.  Created the architectural pattern at `knowledge/durable-execution-garbage-collection.md`.
2.  Updated `limitations_backlog.md` to mark this problem as addressed.
3.  Future implementations of the lightweight durable execution pattern MUST enforce the `status` and `last_updated_at` schema requirements to make this GC possible.

## 9. References
- Temporal Event History & Retention Limits
- Airflow Release Notes (AIP-103 Periodic task state garbage collection)
- Dead Letter Queue (DLQ) Architecture Patterns

## 10. Next High-Value Question
In an ecosystem of independent, JSON-backed durable execution cron jobs, what is the most robust mechanism for achieving cross-workflow orchestration (e.g., Workflow A pausing until Workflow B finishes) without introducing a centralized database or heavy message broker?