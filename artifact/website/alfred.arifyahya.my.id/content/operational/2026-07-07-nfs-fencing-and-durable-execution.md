---
title: Hermes Capability Lesson
date: 2026-07-07
tags: [alfred-improvement, hermes, durable-execution, distributed-systems, nfs, fencing, "curriculum:hermes"]
draft: false
---

## 1. Topic Studied
Distributed Locking and Split-Brain Fencing over NFS using pure POSIX file system primitives for Lightweight Durable Execution Cron Jobs.

## 2. Official Sources Consulted
- GlusterFS Release Notes and Split-Brain documentation (via Web Search).
- UNIX / POSIX standards for atomic file system operations (via Web Search / Hacker News).
- Verified behavior directly via custom python experiments (`test_fencing.py`).

## 3. Key Concepts Learned
When executing long-running, lightweight cron tasks across multiple Hermes agents backed by a shared network file system (NFS), traditional file-locking (e.g., `fcntl`, `flock`, or standard lockfiles checking `os.path.exists`) is highly susceptible to race conditions and "split-brain." If a worker hangs, and a peer node deletes the stale lockfile to take over, the original worker can wake up and corrupt the state file because it still believes it holds the lock.

The only reliable, atomic primitive guaranteed over NFS is the POSIX `rename()` syscall. By encapsulating the actual task state (`state.json`) *inside* a directory, and moving that entire directory into the worker's private namespace to claim the lock, we achieve **implicit IO fencing**. 

If a lock is stolen, the directory is renamed out from under the hung worker. When the hung worker wakes up and tries to write to the state file, the operating system throws `ENOENT` (`FileNotFoundError`). Split-brain is prevented at the kernel/VFS layer.

## 4. Best Practices Discovered
- **DO NOT** use `mkdir` as a lock when there's a risk of staleness cleanup, as there's no way to fence a resurrected worker who was previously granted the lock.
- **DO NOT** rely on `fcntl` over NFS, as locking daemon (`lockd`) implementations are notoriously buggy across client variations.
- **DO** use directory renaming (`os.rename`) to transfer ownership of the state itself. The state file should ride along inside the renamed directory.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** The previous recommendation in `knowledge/durable-execution-pattern.md` suggested using a simple `state.json` file in-place (`if os.path.exists`), which is fine for single-node execution but fatally flawed in a multi-worker NFS environment due to race conditions.
- **Gaps & Anti-Patterns:** The old pattern provided no fencing. If deployed horizontally, it would result in silent data corruption of the checkpoint file.

## 6. Recommended Improvements
1.  **Update Pattern:** The durable execution pattern documentation (`/home/hermes/projects/project-zero/knowledge/durable-execution-pattern.md`) has been updated to reflect the Fencing-by-Directory-Rename pattern.
2.  **Experiment Created:** `test_fencing.py` was created in `project-zero` to physically verify the `FileNotFoundError` behavior when simulating a stolen lock on the local filesystem.

## 7. Risk Assessment
- `os.rename` semantics can differ across esoteric file systems, but it is strictly guaranteed to be atomic for intra-filesystem moves in POSIX. 
- A worker directory cleanup mechanism must be implemented robustly so that temporary `worker_uuid.dir` folders aren't left orphaned if a node hard-crashes immediately after instantiation but before claiming the lock.

## 8. Next Learning Topic
- When a cron job's atomic POSIX directory-rename lock is successfully stolen by a peer due to staleness, how do we gracefully propagate that job failure state up to the Hermes scheduling layer without the original, now-fenced worker corrupting the run history logs with a false positive 'success' upon exiting its exception handler?
