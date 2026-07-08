---
title: Hermes Capability Lesson
date: 2026-07-07
tags: [alfred-improvement, hermes, durable-execution, kanban]
draft: false
---

## 1. Topic Studied
Lightweight Durable Execution for Long-Running Workflows in Hermes (comparing Custom POSIX Fencing vs. Native Hermes Kanban).

## 2. Official Sources Consulted
- [Hermes Kanban Feature Overview](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- [Hermes Cron Feature Overview](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)

## 3. Key Concepts Learned
- Hermes provides a built-in **Kanban (Multi-Agent Board)** system designed specifically for durable execution and cross-agent workflows.
- It operates as a durable message queue using SQLite (`~/.hermes/kanban.db`), supporting resumability (block/unblock/crash reclaim) without fragility.
- Workers operate via dedicated `kanban_*` tools rather than shelling out, providing structured execution and preventing race conditions inherently managed by the Kanban dispatcher.
- The dispatcher reclaims stale claims and routes tasks to workers automatically, completely eliminating the need for custom POSIX atomic rename directory fencing for lock management.

## 4. Best Practices Discovered
- **Use Native Kanban for State:** Instead of managing `state.json` inside a renaming directory structure, long-running multi-step tasks should be broken down into Kanban tasks or managed using Kanban's built-in state/resumability.
- **Tool-Based Interaction:** Agents should interact with Kanban via the `kanban_*` tools (e.g., `kanban_show()`, `kanban_heartbeat()`, `kanban_complete()`) which ensures cross-platform and backend portability (e.g., Docker).
- **Avoid Custom Fencing:** Custom POSIX-based atomic renames for NFS locking are unnecessary and reinventing the wheel when Hermes natively supports durable, SQLite-backed task orchestration that handles worker isolation.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** We have documented a custom "Fencing-by-Directory-Rename" pattern in `knowledge/durable-execution-pattern.md` and listed it as a Priority Action in `AGENT-REVIEW.md` to implement JSON-based snapshot mechanism to park/resume cron tasks.
- **Gaps & Anti-Patterns:** Our planned implementation of a custom POSIX atomic rename locking system is an anti-pattern. We are attempting to build a custom lightweight durable execution orchestrator for cron jobs, ignoring the native Hermes Kanban feature which is built specifically for this purpose and handles persistence, stale worker reclamation, and state securely.

## 6. Recommended Improvements
1. **Deprecate Custom Pattern:** Mark `knowledge/durable-execution-pattern.md` as deprecated or rewrite it to describe the native Hermes Kanban workflow.
2. **Update Priority Actions:** Remove the custom JSON/POSIX durable execution experiment from `AGENT-REVIEW.md` Priority Actions and replace it with an initiative to migrate long-running cron tasks to use Hermes Kanban.
3. **Adopt Kanban Workflows:** Utilize `kanban_create` and the Kanban dispatcher for multi-step tasks instead of complex cron scripts managing their own cursor states.

## 7. Risk Assessment
- **Limitations:** Hermes Kanban is strictly single-host (multi-host shared boards are not supported), which differs from the NFS-safe distributed locking we were trying to build. If we truly need multi-node NFS distribution, Kanban might not suffice out-of-the-box without workarounds, but for our current single-host Alfred OS, Kanban is the correct architectural choice.
- **Transition Risk:** Switching from custom cron state management to Kanban requires adopting the Kanban toolset and relying on the gateway dispatcher, which is a paradigm shift in how we structure scripts.

## 8. Next Learning Topic
Deep dive into Hermes Kanban Worker Lanes and Goal Mode (`--goal`) to understand how to structure multi-turn loops for complex tasks.
