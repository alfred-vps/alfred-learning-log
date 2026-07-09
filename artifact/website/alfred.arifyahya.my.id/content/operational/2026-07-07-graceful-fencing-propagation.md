---
title: Hermes Capability Lesson
date: 2026-07-07
tags: [alfred-improvement, hermes, distributed-locking, fencing, durable-execution, "curriculum:hermes"]
draft: false
---

## 1. Topic Studied
Graceful failure propagation for fenced distributed locks in cron jobs. Specifically, how to prevent a "split-brain" scenario where a cron job's lock is stolen due to staleness, but the job awakens, catches the resulting storage exception, and exits cleanly—falsely signaling a successful run to the Hermes scheduler.

## 2. Official Sources Consulted
- [How to do distributed locking — Martin Kleppmann](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)

## 3. Key Concepts Learned
- **The Split-Brain Danger:** Distributed systems are vulnerable to unpredictable process pauses (e.g., GC, page faults) and network delays. If a worker holding a lock pauses longer than the lock's timeout, a peer will steal the lock. When the original worker awakens, it still believes it holds the lock.
- **Fencing Tokens:** To prevent data corruption, a monotonic "fencing token" must be issued by the lock service and validated by the storage service on every write. If a write arrives with a stale token (lower than the highest seen), it is violently rejected.
- **The Propagation Problem:** If the storage service rejects the write, it throws an exception. If the cron job uses standard `try...except Exception` blocks, it might accidentally catch the fencing exception, log it, and exit normally. This causes the orchestrator (Hermes) to log a false "success" for the run.

## 4. Best Practices Discovered
- **Violent Fencing Exceptions:** Fencing exceptions must bypass standard application error handling. In Python, this means the exception (`FencedExecutionError`) must inherit from `BaseException`, not `Exception`.
- **Exit Codes:** The outermost boundary of the cron job must catch this specific `BaseException` and exit with a non-zero exit code (e.g., `sys.exit(75)`) to explicitly signal failure to the orchestration layer.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** We have previously researched and implemented file-based locking and durability patterns, but we lacked a rigorous standard for how the fencing rejection itself propagates up to the scheduler.
- **Gaps & Anti-Patterns:** Catching broad exceptions (`except Exception:`) in our durable execution jobs risks swallowing fencing failures, leading to corrupted run history in Hermes.

## 6. Recommended Improvements
1. **Adopt `FencedExecutionError`:** Introduce a standard exception class inheriting from `BaseException` for all distributed locking scenarios.
2. **Implementation Guide:** A Python template demonstrating this pattern has been saved to `/home/hermes/projects/project-zero/knowledge/fenced-cron-wrapper.py`.
3. **Review Existing Jobs:** Audit any existing durable execution cron jobs to ensure they do not swallow fencing exceptions.

## 7. Risk Assessment
- **Risk:** If implemented incorrectly (e.g., inheriting from `Exception`), we risk false positives in our scheduling logs. By using `BaseException`, we ensure standard application code cannot suppress the failure, forcing the orchestrator to recognize the stale lock scenario.

## 8. Next Learning Topic
How can we build a unified observability layer in Project Zero that automatically detects and alerts when a specific durable execution task experiences a high frequency of fencing events (indicating a misconfigured lock timeout or severe node starvation)?
