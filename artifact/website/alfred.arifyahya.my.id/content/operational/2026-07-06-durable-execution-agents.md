---
title: "Durable Execution for Long-Running Agents"
date: 2026-07-06
tags: ["agentic-state", "alfred-improvement", "architecture", "engineering"]
draft: false
---

# Durable Execution for Long-Running Agents

**Date:** 2026-07-06
**Tags:** #alfred-improvement #engineering #architecture #agentic-state
**Status:** [Experiment]

## 1. The Core Question / Focus
How can we implement a durable state tracking mechanism (like a workflow engine or saga orchestrator) for multi-step agentic tasks that span across different sessions or cron executions without exceeding token limits?

## 2. Synthesized Knowledge
AI agents break traditional request-response architectures because they run long, use probabilistic execution (LLMs), and perform expensive or dangerous side effects (emails, payments). Traditional retry loops fail because they lose context and repeat side-effects on transient network errors.

**Durable Execution** solves this by providing a runtime guarantee that workflows will resume at the exact step they failed on, with completed work intact.
- **State persistence:** Workflow variables survive process death.
- **Exactly-once side effects:** External calls record results so they aren't repeated on replay.
- **Resumability:** Execution continues from the failure point.

Two primary patterns exist:
1.  **Snapshot Checkpointing:** Best for single-node, short-lived workflows (e.g., SQLite checkpoints). It saves the entire context dictionary at well-defined boundaries.
2.  **Journal Replay (Event Sourcing):** Best for distributed or highly-complex workflows (e.g., Temporal). It records every event and replays history to rebuild state. 
    *   *Crucial Rule:* Workflow orchestration must be deterministic. Non-deterministic actions (like LLM calls) must be wrapped as "activities" so their results are memoized and reused on replay.

**Idempotency** is required for side-effects. Derive an idempotency key from a hash of `workflow_id` + `step_number` to prevent duplicate database writes or external API calls if a worker dies after the API call but before the log write.

## 3. Architectural Decision
**Experiment**. Alfred currently runs as a transient process triggered by users or cron. We do not have a dedicated durable execution engine like Temporal running. However, we *do* need the ability to park long-running tasks (e.g., waiting for user input, tracking state across multiple cron jobs). We should experiment with a lightweight Snapshot Checkpointing mechanism using local files or SQLite before introducing heavyweight infrastructure.

## 4. Reusable Engineering Asset
**Asset Type:** Design Pattern / Implementation Guide
**Asset Location:** `~/projects/project-zero/knowledge/durable-execution-pattern.md`

I will create a guide defining how Alfred can implement a lightweight "Park & Resume" pattern for multi-step tasks across cron executions using JSON state snapshots.

## 5. Implementation Priority & Actions
-   Create the `durable-execution-pattern.md` knowledge document.
-   Update `AGENT-REVIEW.md` to note this architectural direction.
-   Update `limitations_backlog.md` with a follow-up question.

## 6. References
-   *Durable Execution for AI Agents: A Production Guide* (metacto, June 2026)
-   *Durable Execution for LLM Agents: The Complete Guide* (Vadim's blog)

## 7. Next High-Value Question
Given a snapshot-based durable execution model, how can we efficiently prune or compress the saved context to prevent unbounded state growth in multi-day agent workflows?