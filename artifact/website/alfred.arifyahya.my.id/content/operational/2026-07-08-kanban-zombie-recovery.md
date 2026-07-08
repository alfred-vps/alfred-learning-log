---
title: "Kanban Zombie Recovery"
date: 2026-07-08
tags: ["alfred-improvement", "durable-execution", "hermes", "kanban", "self-healing"]
draft: false
---

# Hermes Capability Lesson

**Date:** 2026-07-08
**Tags:** #alfred-improvement #hermes #kanban #durable-execution #self-healing

## 1. Topic Studied
Self-healing Kanban reconciliation loop (Zombie Task Recovery). How do we automatically recover tasks that are falsely marked as 'running' in `kanban.db` but have crashed without emitting a block/completion signal (i.e., missed heartbeats)?

## 2. Official Sources Consulted
- Temporal Platform Documentation (Heartbeat Timeouts and Activity Eviction)
- Industry standard Message Queue / Task Queue recovery patterns (Celery, AWS SQS)
- Project Zero `limitations_backlog.md`

## 3. Key Concepts Learned
In distributed systems and durable execution workflows (like Temporal, Celery, and now Hermes Kanban), tasks transitioning to a 'running' state represent a temporary lease. If the worker processing that task crashes (OOM, node failure, unexpected exception) before it can emit a completion or failure signal, the task becomes a "zombie" — indefinitely holding the 'running' state.

The industry-standard solution is the **Heartbeat & Timeout Pattern**:
1. Workers must periodically signal they are alive (heartbeating).
2. A separate reconciliation process (or the orchestrator itself) scans for 'running' tasks whose last updated timestamp (heartbeat) exceeds a defined timeout.
3. These tasks are evicted from the running state and re-queued (`ready` or `todo`) for retry, up to a maximum retry limit, after which they are sent to a Dead Letter Queue (DLQ).

## 4. Best Practices Discovered
- **Decouple the Monitor:** The process sweeping for zombies should be independent of the workers processing the tasks. (e.g., a cron job checking the database).
- **Idempotency is Required:** Because a task might be re-queued, the underlying workflow step must be idempotent. It must be safe to run the task again if it crashed halfway through.
- **Dynamic Timeouts:** Not all tasks take the same amount of time. Timeouts should ideally be configurable per task type, but a global default is a necessary baseline.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** We recently migrated to Hermes Kanban and implemented a `kanban-archiver.py` for `done` tasks. However, we lacked a mechanism to recover tasks stuck in `running` if a scheduled job or node fails unexpectedly.
- **Gaps & Anti-Patterns:** Without a zombie sweeper, failed jobs require manual intervention to reset their state in the SQLite database, violating the "self-healing" goal of a reliable operating system.

## 6. Recommended Improvements
**Implement Now:**
1. Created `scripts/maintenance/kanban-zombie-sweeper.py`. This script acts as an independent watchdog. It connects to `~/.hermes/kanban.db`, scans for tasks in the `running` state where `updated_at` exceeds a configurable threshold (default 2 hours), and resets them to `ready` so they can be picked up by the next worker execution.

## 7. Risk Assessment
- **Premature Eviction:** If the `KANBAN_ZOMBIE_TIMEOUT_HOURS` is set too low for a genuinely long-running task that doesn't update its timestamp frequently, the sweeper might reset it while it's still running. This results in duplicate execution. 
- **Mitigation:** Ensure the timeout is generously higher than the longest expected synchronous step, or implement periodic updates to the task in `kanban.db` during long executions to serve as a heartbeat.

## 8. Next Learning Topic
How do we gracefully pause and persist the context of a long-running conversational task in Kanban so it can be resumed via a webhook (e.g., waiting for human input) without requiring a continuously polling daemon?