---
title: "Kanban Saga Pattern"
date: 2026-07-08
tags: ["alfred-improvement", "distributed-systems", "hermes", "kanban", "reliability"]
draft: false
---

## 1. Topic Studied
Automated Compensating Transactions and Dead Letter Queues (DLQ) within the Hermes Kanban distributed execution framework, focusing on how to gracefully recover from partial failures in multi-agent workflows without manual intervention or corrupting the project state.

## 2. Official Sources Consulted
- Hermes Kanban (Multi-Agent Board) Documentation: `https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban`
- Hermes Slash Commands Reference: `https://hermes-agent.nousresearch.com/docs/reference/slash-commands`
- Hermes Configuration Reference: `https://hermes-agent.nousresearch.com/docs/user-guide/configuration`
- Hermes Checkpoints and Rollback Documentation: `https://hermes-agent.nousresearch.com/docs/user-guide/checkpoints-and-rollback`

## 3. Key Concepts Learned
Hermes provides two distinct native mechanisms that, when combined, offer a robust solution for compensating transactions (the Saga pattern) in multi-agent workflows:

1.  **Durable Execution via Kanban:**
    *   **State Machine:** Hermes Kanban isn't just a task list; it's a durable state machine (`triage | todo | ready | running | blocked | done | archived`).
    *   **Dependency Graph:** Tasks are linked (Parent -> Child). A child task moves to `ready` only when all parent tasks are `done`.
    *   **Failure States:** If a worker crashes or exceeds its heartbeat timeout, the gateway dispatcher reclaims the task and resets its state, allowing another worker (or a retry) to pick it up. If a worker explicitly fails (via `kanban_block`), the task halts.
    *   **Workspaces:** The `workspace` parameter (`scratch`, `dir:<path>`, `worktree:<path>`) defines the isolation boundary for the worker. `worktree:<path>` is particularly crucial for safety.

2.  **Filesystem Durability via Checkpoint Manager:**
    *   **Automated Snapshots:** Hermes features an opt-in Checkpoint Manager (backed by a shared git repo in `~/.hermes/checkpoints/store/`). When enabled (`checkpoints.enabled: true`), Hermes automatically snapshots the directory *before* any destructive operation (e.g., `write_file`, `patch`, `rm`, `sed`).
    *   **Targeted Rollback:** The `/rollback` slash command and the underlying CLI (`hermes checkpoints`) allow restoring specific files or entire directories to a known good state. Crucially, the rollback process inherently understands the agent's context and can "undo the last chat turn" to keep the LLM synced with the filesystem.

## 4. Best Practices Discovered
To build reliable, multi-step agent workflows (Sagas) in Hermes, the official pattern avoids custom "undo" logic written into the agent's prompt and instead relies on the orchestrator (Kanban) and the environment (Checkpoints).

*   **Isolate Execution:** Always run complex, multi-agent tasks within a dedicated `worktree:<path>`. This isolates the execution environment from the main project branch.
*   **Enable Checkpoints:** Ensure `checkpoints.enabled: true` is set globally or passed via CLI for any workflow modifying the filesystem.
*   **Orchestrator-Driven Rollback (The Saga Pattern):**
    *   If a sub-task (child task) in a Kanban graph fails (enters the `blocked` state due to a tool failure or explicit abort), the parent orchestrator task must detect this.
    *   Instead of trying to manually "fix" the files, the orchestrator should issue a compensating transaction: triggering a rollback via the native checkpoint system (`/rollback <N>` or equivalent API/CLI call if automated) to restore the environment to the pre-task snapshot.
*   **Dead Letter Queue (DLQ):** Tasks that repeatedly fail and are reclaimed by the dispatcher, or tasks that require human intervention to resolve a conflict that automatic rollback cannot fix, should be moved to a specific `blocked` state with a designated `tenant` or tag (e.g., `dlq`) so they can be monitored and triaged by a human or a specialized `curator` profile.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** We have been experimenting with custom JSON-based state tracking and manual directory renaming (which was reversed in favor of Kanban). However, we lack a standardized pattern for handling *failures* within a Kanban workflow. If a "coder" agent fails halfway through a refactor, the repository is left in a broken state, requiring manual `git checkout` or `git reset` by Arif.
- **Gaps & Anti-Patterns:**
    -   We are not consistently enforcing `worktree` usage for Kanban tasks.
    -   We have not integrated the native Checkpoint Manager into our automated workflows; we rely on git commits after the fact.
    -   We lack a formalized DLQ mechanism for failed agent tasks.

## 6. Recommended Improvements
1.  **Draft an Architecture Decision Record (ADR): "Kanban Workflow Failure Handling and Compensating Transactions".** This ADR will mandate the use of `worktree` isolation and native Checkpoints for all multi-step Kanban tasks modifying project code.
2.  **Develop an Orchestrator Prompt Pattern:** Create a reusable prompt template (or a specialized skill) for Orchestrator agents. This pattern must instruct the orchestrator to monitor its child tasks. If a child task transitions to `blocked`, the orchestrator must automatically invoke the necessary rollback mechanism to revert the worktree to the pre-task state.
3.  **Implement a DLQ Monitoring Cron:** Create a lightweight cron script (`scripts/maintenance/check-kanban-dlq.sh`) that queries the `kanban.db` for tasks in the `blocked` state for more than X hours, escalating them (e.g., via Telegram notification) for human review.

## 7. Risk Assessment
- **Risk:** The Checkpoint Manager relies heavily on `git`. If `git` is unavailable or the directory structure is extremely large (>50,000 files), snapshots are skipped, negating the rollback capability.
- **Limitation:** Rollbacks revert the filesystem but do not inherently "un-complete" a Kanban task. The orchestrator must explicitly handle the state transition (e.g., moving the failed task back to `todo` or archiving it and creating a new one) after the filesystem rollback.
- **Complexity:** Coordinating a rollback across multiple parallel agent workflows on the same codebase requires careful handling of git branches/worktrees to avoid merge conflicts during the compensating transaction.

## 8. Next Learning Topic
How to securely isolate environment variables and secret credentials per Kanban worker profile (e.g., ensuring the `coder` profile cannot access the `deployer` profile's AWS keys) within a single Hermes Gateway instance.