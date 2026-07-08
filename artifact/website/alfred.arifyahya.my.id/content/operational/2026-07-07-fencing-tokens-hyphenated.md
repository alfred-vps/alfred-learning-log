---
title: "Fencing Tokens for Distributed Locking Correctness"
date: 2026-07-07
tags: ["alfred-improvement"]
draft: false
---

# Fencing Tokens for Distributed Locking Correctness

## Context & Rationale
**Why this topic?** The `AGENT-REVIEW.md` highlights an ongoing priority to "Experiment with Lightweight Durable Execution" and implement a JSON-based snapshot mechanism to park and resume multi-step cron tasks without needing heavyweight orchestrators. Furthermore, the `limitations_backlog.md` previously addressed how to build a POSIX-based locking system and handle staleness. 

However, relying solely on POSIX locks or TTLs across distributed workers is fundamentally unsafe for *correctness* operations. If a worker process pauses (e.g., due to host CPU starvation, I/O wait, or network latency), its lock might expire and be stolen by a peer. When the original worker wakes up, it believes it still holds the lock and proceeds to write data, leading to a "split-brain" state and corruption.

This improvement addresses the critical architectural gap of ensuring that a stale worker cannot corrupt state after losing its lock.

**Expected Impact:** High (40/40). Prevents catastrophic data corruption in distributed, long-running durable execution workflows by enforcing strict ordering at the storage layer.

## Evidence & Research
**Evidence Quality:** High. (Authoritative Architecture Reference)
- Martin Kleppmann's "How to do distributed locking" (Primary source on the flaws of Redlock and time-based TTLs).
- Distributed systems engineering consensus on the necessity of fencing tokens for correctness.

**Key Findings:**
1. **The Fencing Gap:** Lock expiration (TTL) only protects the lock service. It does not protect the underlying storage from a delayed worker.
2. **Fencing Tokens:** A lock service must issue a strictly monotonically increasing integer (token) upon every acquisition.
3. **Storage Enforcement:** The storage layer must actively reject writes carrying a token older than the highest token it has already processed.
4. **Conditional Updates:** In a JSON-durable execution model, this translates to Optimistic Concurrency Control (OCC) where every state update includes a `version` field, and the storage write fails if the current file version exceeds the worker's assumed version.

## Engineering Value Assessment (EVA)
- **Impact (0-40):** 40 (Critical for data integrity in distributed tasks)
- **Evidence (0-20):** 20 (Strongly supported by foundational distributed systems theory)
- **Actionability (0-20):** 18 (Highly actionable via JSON state versioning in file-based stores)
- **Reusability (0-20):** 20 (Fundamental pattern for any future multi-node workflow engine)
- **Total Score:** 98/100 -> **Implement Pattern Now (via Design Asset)**

## Asset Generation
Generated the design pattern at: `/home/hermes/knowledge/fencing-tokens.md`

## Backlog Update
Prepend to `/home/hermes/docs/obsidian/learning/limitations_backlog.md`:
*How can we build a unified observability layer in Project Zero that automatically detects and alerts when a specific durable execution task experiences a high frequency of fencing events (indicating a misconfigured lock timeout or severe node starvation)?*