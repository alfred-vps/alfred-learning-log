---
title: "Migrating from Reactive Agent Loops to Kanban Orchestration"
date: 2026-07-08
status: Approved
tags:
  - architecture
  - multi-agent
  - durable-execution
  - kanban
  - "curriculum:legacy"
draft: false
---

# North Star Lesson: Migrating from Reactive Agent Loops to Kanban Orchestration

## 1. Strategic Selection

**Topic:** How can we safely version, orchestrate, and deploy multi-step long-running tasks within Alfred using the native Kanban features rather than brittle custom-built filesystem locks?

**Rationale:** Previously, we documented an attempt to build a custom POSIX atomic rename locking system for durable execution. The `AGENT-REVIEW.md` explicitly calls out that this was a mistake and a governance failure ("Re-inventing Native Features"). The priority action for this cycle is to migrate to Hermes Kanban. We need to formalize how Kanban should be used for Alfred's own background maintenance tasks to align with the "Workflow-First Architecture" directive established on 2026-07-07.

## 2. Research Findings

**Methodology:** Reviewed the official Hermes Agent Kanban documentation and tutorial pages via web extraction.
**Evidence Quality:** Primary Docs (Hermes Agent official documentation).

**Key Discoveries:**
1.  **Durable State Machine:** Kanban uses a persistent, SQLite-backed task board (`~/.hermes/kanban.db`) shared across all profiles, eliminating the need for custom JSON files and directory locking.
2.  **Tool-Driven Workflow:** Agents should NEVER shell out to the CLI to manage tasks. They must use the native `kanban_*` toolset (e.g., `kanban_show`, `kanban_complete`, `kanban_block`).
3.  **Handoff Evidence:** When completing a task, agents should provide structured evidence in the metadata (e.g., `changed_files`, `verification`, `decisions`) via `kanban_complete`. This allows downstream or retrying workers to resume intelligently.
4.  **Resilience Features:** Kanban inherently supports parent/child task dependencies, human-in-the-loop blocking/unblocking, circuit breakers for infinite loops, and crash recovery (dispatcher reclaims dead PIDs).
5.  **Workspace Isolation:** Kanban natively handles working directory assignments (`scratch` for ephemeral, `dir:<absolute-path>` for shared persistent directories), which is critical for Alfred's operations across various projects.

## 3. Engineering Value Assessment (EVA) & Decision Gate

- **Impact (40/40):** Directly addresses the primary `AGENT-REVIEW.md` friction point. Moves from fragile custom state to a native, resilient, multi-agent orchestration engine.
- **Evidence (20/20):** Direct from official Hermes documentation.
- **Actionability (20/20):** Native tools (`kanban_create`, etc.) are already available in the current environment.
- **Reusability (20/20):** Defines the standard operating procedure for all future background automation and cron tasks in Project Zero.
- **Total Score:** 100/100

**Decision:** **Implement Now** (Document the standard pattern).

## 4. Synthesis & Asset Generation

**Asset Produced:** We are generating a standardized template/guide for how Alfred cron tasks should orchestrate multi-step workflows using Kanban, ensuring we do not reinvent durable execution.

### Standard Pattern: Kanban for Alfred Background Jobs

When Alfred (running as a cron job or background process) needs to execute a multi-step operation, a batch process over many items, or delegate to a specialized profile, it must use the Kanban board instead of trying to maintain state in memory or custom files.

#### Step 1: Idempotent Creation (Orchestrator)
The initiating job uses `kanban_create` (via the CLI or tool) to schedule the work. Always use an `idempotency_key` based on the date/item to prevent duplicate runs if the cron triggers multiple times.

```bash
hermes kanban create "Weekly Knowledge Consolidation" \
    --assignee alfred-researcher \
    --tenant project-zero \
    --idempotency-key "knowledge-sweep-$(date +%Y-%W)" \
    --body "Review draft notes and convert to formal knowledge base articles."
```

#### Step 2: The Worker Loop
The assigned worker uses the Kanban toolset exclusively:
1.  **Read context:** Uses `kanban_show` to understand dependencies and past attempts.
2.  **Report progress:** If the task takes longer than an hour, the worker *must* call `kanban_heartbeat()` to prevent the dispatcher from reclaiming it.
3.  **Handle blockers:** If required tools are missing or human input is needed, call `kanban_block(reason="needs_input", message="Clarification on ...")`. Do NOT just exit or hallucinate a result.
4.  **Structured Completion:** Call `kanban_complete` with a clear summary and metadata payload.

```python
# Example worker completion payload
kanban_complete(
    summary="Consolidated 3 draft notes into knowledge/kanban-patterns.md",
    metadata={
        "changed_files": ["/home/hermes/projects/project-zero/knowledge/kanban-patterns.md"],
        "verification": ["Markdown syntax check passed"],
        "residual_risk": ["User review recommended for specific categorization"]
    }
)
```

## 5. The "Better Alfred" Test

By enforcing this pattern, Alfred stops trying to be a complex monolithic agent attempting to hold massive context and manage its own filesystem state. Instead, Alfred becomes a reliable workflow orchestrator, utilizing the robust native Kanban engine for scheduling, resilience, and multi-agent delegation.