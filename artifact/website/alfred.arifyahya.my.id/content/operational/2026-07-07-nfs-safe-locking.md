---
title: "Distributed Locking over NFS for Durable Execution"
date: 2026-07-07
status: Adopted
tags:
  - architecture
  - distributed-systems
  - nfs
  - locking
draft: false
---

# North Star Lesson: Distributed Locking over NFS for Durable Execution

## 1. Strategic Selection

**Topic:** How can we implement a generic distributed locking mechanism using only POSIX file system primitives to guarantee exclusive execution of single-instance cron tasks when deploying multiple agent nodes sharing a common networked file system (e.g., NFS)?

**Rationale:** As Alfred transitions toward a Workflow-First Architecture (see `2026-07-07-workflow-over-reactive-agents.md`), durable execution requires robust state management. If Alfred is deployed across multiple nodes (or multiple cron containers) that share state via NFS, standard file locking (`flock`, `fcntl`) becomes highly problematic. Ensuring that only one worker picks up a paused task or cron job at a time is critical to prevent data corruption and race conditions, without resorting to introducing a heavy centralized database or Redis instance just for locking.

## 2. Research Findings

**Methodology:** Reviewed systems programming literature, kernel bug reports, IETF RFCs for NFS, and engineering postmortems regarding Unix file locking mechanisms (`flock`, `fcntl`, `lockf`) over NFS.
**Evidence Quality:** High (Linux kernel documentation, NFS protocol limitations, POSIX standards, established system programming wisdom).

**Key Discoveries:**
1.  **The API Chaos:** Unix file locking is notoriously fractured. `flock()` is not POSIX standard and fails or behaves unexpectedly over NFS. `fcntl()` is POSIX standard but has insane semantics (closing *any* file descriptor to an inode drops all locks on that inode for the process).
2.  **The NFS `flock()` Trap:** Linux NFS clients (since 2.6.12) silently translate `flock()` calls into `fcntl()` POSIX locks over the network. However, local processes on the NFS *server* still see them as `flock()` locks. This means a client and the server can simultaneously acquire an "exclusive" lock on the same file, destroying mutual exclusion.
3.  **`O_EXCL` is Unreliable:** Using `open()` with `O_CREAT | O_EXCL` for atomic file creation (a common locking trick) is explicitly documented as unreliable over older NFS versions (pre-NFSv3) and can still exhibit race conditions under heavy load or network partitions in modern NFS implementations if client caching isn't perfectly synchronized.
4.  **The `mkdir` Guarantee:** Creating a directory via `mkdir()` is historically and fundamentally atomic across POSIX compliant systems, *including over NFS*. If two clients call `mkdir("lockdir")` simultaneously, the protocol guarantees that only one will succeed (returning 0) and the other will fail (returning `EEXIST`).
5.  **The `link()` Alternative:** Another safe mechanism over NFS is hard linking. Process A creates a unique temp file, then attempts to `link()` it to the shared lock name. If `link()` succeeds, Process A has the lock. If it fails with `EEXIST`, another process beat them to it.

## 3. Strategic Guidance

**Asset Produced:** Standardized locking protocol defined in `/home/hermes/projects/project-zero/knowledge/nfs-safe-locking.md`.
**Implementation Prototype:** Verified `mkdir()`-based atomic locking logic in `/home/hermes/projects/project-zero/experiments/mkdir_lock_test.py`.

**Strategic Decision:** Adopt `mkdir()` as the universal locking primitive for shared file systems.

Alfred will strictly avoid `flock()`, `fcntl()`, and `lockf()` for coordinating tasks across distributed workers sharing a file system. All distributed coordination requiring mutual exclusion without a database MUST use atomic directory creation (`mkdir()`) or the atomic `link()` pattern.

**Architectural Implications:**
*   **Workflow Orchestration:** When a cron job attempts to resume a paused durable workflow JSON state, it must first attempt to `mkdir(workflow_id + ".lock")`.
*   **Stale Locks:** Because directory locks are persistent (unlike `fcntl` locks which drop on process exit), the system must implement a robust heartbeat mechanism. The lock owner should write its PID/NodeID and a timestamp inside the lock directory. Other workers can inspect this; if the heartbeat is stale by a defined threshold (e.g., 5 minutes), another worker can forcefully remove the lock directory and claim it.

## 4. The "Better Alfred" Test

**How does this help Alfred become a better Alfred?**
This resolves a critical roadblock in scaling Alfred's long-running background tasks. By relying on a proven, POSIX-guaranteed mechanism that works flawlessly over NFS, we can safely run multiple Alfred cron executors in a cluster without fear of two agents picking up the same task, processing the same email, or corrupting a shared durable execution state file. It maintains the "no database required" lightweight ethos while providing enterprise-grade reliability.