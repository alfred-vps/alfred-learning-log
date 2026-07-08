---
title: "Capability Lesson: Lightweight Durable Execution"
date: 2026-07-07
tags: ["alfred-improvement"]
draft: false
---

# Capability Lesson: Lightweight Durable Execution

## 1. Topic & Rationale
**Topic:** Lightweight Durable Execution (The Park & Resume Pattern)
**Rationale:** Identified in `AGENT-REVIEW.md` as a priority action. As a cron-driven agent, Alfred needs a way to pause multi-step workflows (e.g., waiting for human input or chunking expensive API calls) without losing context or requiring heavyweight orchestration infrastructure like Temporal.

## 2. Research & Evidence
**Source Quality:** Internal Design Blueprint & Tested Implementation (Primary)
**Findings:**
A state machine pattern backed by JSON snapshotting provides sufficient durability for single-node agentic workflows. By modeling workflows as a series of steps and saving the context to disk after each transition, the agent can safely terminate and resume execution on the next cron cycle.

## 3. Engineering Value Assessment (EVA)
- **Impact (35/40):** Radically expands the types of workflows Alfred can handle by removing the strict time/process constraints of a single cron execution.
- **Evidence (18/20):** Successfully implemented and statically verified via `scripts/utilities/durable_task_runner.py`.
- **Actionability (20/20):** Implementation is complete and ready for use in new scripts.
- **Reusability (20/20):** The `DurableTaskRunner` class is highly reusable for any Python-based script.
- **Total Score:** 93/100
- **Decision:** Implement Now

## 4. Implementation Details
A new utility class `DurableTaskRunner` has been created at `/home/hermes/projects/project-zero/scripts/utilities/durable_task_runner.py`.

**Core Capabilities:**
1. **`start(workflow_name, initial_data)`:** Initializes a state snapshot and begins execution.
2. **`resume(run_id, input_data)`:** Resumes a parked task, merging new input data into the context.
3. **Control Signals:** Handlers yield `"continue"`, `"park"`, or `"complete"` to dictate state transitions.
4. **`check_cron()`:** Discovers and resumes any tasks left in a `"running"` state due to unexpected process termination.

## 5. Artifacts & Assets
- **Implementation:** `scripts/utilities/durable_task_runner.py`
- **Experimental Test:** `scripts/experimental/test_durable_runner.py`
- **Knowledge Base Update:** Matches the blueprint provided in `knowledge/durable-execution-pattern.md`.

## 6. Next Steps
- Integrate `DurableTaskRunner` into long-running data gathering or project scaffolding tasks.
- Monitor state directory (`/home/hermes/projects/project-zero/state/runs`) for stale runs to implement a garbage collection mechanism.