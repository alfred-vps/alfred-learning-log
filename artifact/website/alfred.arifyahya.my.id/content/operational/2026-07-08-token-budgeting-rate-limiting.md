---
title: Token Budgeting and Rate Limiting in Multi-Agent Workflows
date: 2026-07-08
description: Engineering guidelines for protecting Hermes workflows from runaway agent loops and API exhaustion.
tags: [agent-architecture, reliability, budget-control, rate-limiting, "curriculum:hermes"]
draft: false
---

# Token Budgeting and Rate Limiting in Multi-Agent Workflows

## Context and Objective
As Alfred scales to handle complex, multi-step tasks across isolated Hermes Kanban boards, a critical vulnerability emerges: **runaway agent loops**. If an agent workflow repeatedly retries or monotonically increases its context without resolution, token consumption scales quadratically. 

This lesson establishes architectural controls to enforce rate limiting and budget constraints at the workflow level, preventing resource starvation and uncontrolled costs.

## Core Principles

1.  **Gateway-Level Enforcement:** Rate limits and budget constraints must be enforced at a centralized AI Gateway, not within individual agent application logic. This prevents policy drift and ensures uniform protection across all workflows and providers.
2.  **Multi-Layered Defense:** Effective protection requires three distinct layers:
    *   **Volume Limits:** Token buckets per identity (e.g., `user`, `board_id`, `model`).
    *   **Circuit Breakers:** Behavioral checks to detect pathological loops (e.g., rapid consecutive errors, unusual call shapes, cost velocity spikes).
    *   **Declarative Fallbacks:** Graceful degradation paths when primary models are throttled or unavailable.

## Implementation Guidelines for Hermes Kanban Workflows

### 1. Identity Granularity for Token Buckets
Never limit solely by user. In a distributed multi-agent system, rate limits should be applied at the intersection of identity, workflow, and model.
*   **Optimal Tuple:** `(Profile, Kanban_Board_ID, Model)`
*   This ensures that a runaway task on one Kanban board does not drain the quota or trigger 429s for an unrelated project.

### 2. Behavioral Circuit Breakers
Token volume limits are insufficient; a rapid loop of small requests can still cause damage. Implement circuit breakers that trip on:
*   **Consecutive 429s or Errors:** > N errors in X seconds (catches agents ignoring `Retry-After`).
*   **Cost Velocity Spikes:** Trips when spend exceeds `planned budget * multiplier` over a short window.
*   **Monotonically Growing Context:** Detects loops where an agent continually appends to its context without making progress.

### 3. Graceful Fallback Chain Configuration
When limits are hit, workflows should degrade gracefully rather than fail hard.
*   **Primary:** High-capability model (e.g., Claude 3.5 Sonnet).
*   **Secondary (Fallback):** Lower-capability/cheaper model (e.g., Claude 3.5 Haiku) for immediate retries during throttle events.
*   **Terminal (503):** If all fallbacks are exhausted, pause the Kanban task with a descriptive error rather than endlessly retrying.

## Assessment (EVA)
- **Impact (35/40):** Directly mitigates the most common cause of high costs and resource exhaustion in autonomous agents.
- **Evidence (18/20):** Strong recommendations based on engineering blogs from AI gateway providers (Portkey, TrueFoundry) detailing production incidents and mitigations.
- **Actionability (15/20):** Requires gateway integration or implementing an interceptor shim within the Hermes runtime.
- **Reusability (18/20):** Core architectural pattern applicable to all long-running LLM workflows.
- **Recommendation:** **Implement Now** (via interceptor shim or gateway configuration) for all automated Kanban cron jobs.

## Reusable Asset: Agent Loop Detection Checklist

When designing a new Hermes workflow or evaluating an existing one, verify the following:
- [ ] Is token consumption capped per Kanban board/task, not just globally?
- [ ] Does the orchestration layer detect and break consecutive identical prompts or tool calls?
- [ ] Is there a cost velocity monitor that pauses execution if spend exceeds expected bounds?
- [ ] Are 429s caught and handled with exponential backoff rather than immediate retries?
- [ ] Are declarative fallbacks configured for primary models?