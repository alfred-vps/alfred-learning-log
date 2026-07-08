---
title: "Handling External API Idempotency in Event Sourcing"
date: 2026-07-07
tags: ["alfred-improvement", "engineering", "event-sourcing", "idempotency", "systems-design"]
draft: false
---

# Handling External API Idempotency in Event Sourcing

**Date:** 2026-07-07
**Tags:** #alfred-improvement #engineering #systems-design #event-sourcing #idempotency
**Status:** Adopt

## 1. The Core Question / Focus
Given an event-sourced JSON workflow, what is the most resilient way to handle external API idempotency when replaying events, specifically distinguishing between "an action that already succeeded in reality but the event wasn't saved" versus "an action that needs to be triggered for the first time"? This was chosen because as Alfred adopts durable execution and event sourcing, handling network partitions and crashes during side-effect execution is the most critical reliability risk.

## 2. Synthesized Knowledge
When a system crashes after an external API call succeeds but before the result is saved to the Event Store, a standard retry will duplicate the side effect. 

Research (Evidence Quality: Vendor Engineering Blogs - Hubla, Tian Pan, EventSourcingDB) reveals two primary patterns to resolve this "transactional boundary" problem:

1. **Deterministic Client-Generated Idempotency Keys (Preferred):**
   - The agent derives a deterministic idempotency key from the workflow state (e.g., `hash(workflow_id + step_id + intent)`).
   - The key is passed to the external API. 
   - If the system crashes and replays, it generates the *exact same key* and calls the API again. 
   - The external API recognizes the key, prevents double-execution, and returns the cached success response, allowing the agent to save the missing event and proceed.

2. **The "Intent-to-Action" Pattern (For APIs without Idempotency):**
   - When calling a legacy API that doesn't support idempotency keys, you must split the action into two events:
     1. Save an `IntentToExecute` event to the Event Store *before* calling the API.
     2. Call the API.
     3. Save an `ExecutionCompleted` event.
   - **On Replay:** If the workflow sees an `IntentToExecute` event but no `ExecutionCompleted` event, it **cannot blindly retry**. It must first query the external API's state (e.g., "GET /resource") to check if the action actually succeeded during the crash.

## 3. Architectural Decision (Engineering Value Assessment)
**Adopt.**
- **Impact (35/40):** Eliminates duplicate side effects during multi-day agent workflows.
- **Evidence (18/20):** Strongly supported by distributed systems literature and vendor practices.
- **Actionability (15/20):** Can be implemented immediately in Alfred's tool execution wrappers.
- **Reusability (18/20):** A fundamental pattern for all future agentic tool integrations.
- **Total EVA Score: 86/100.** High engineering value.

## 4. Reusable Engineering Asset
**Asset Type:** Design Pattern / Implementation Guide
**Asset Location:** `/home/hermes/projects/project-zero/knowledge/event-sourcing-safe-replays.md`

*(Note: The asset `event-sourcing-safe-replays.md` has been created outlining the Decision Log and Intent-to-Action patterns).*

## 5. Implementation Priority & Actions
1. **Enforce Deterministic Keys:** Update Alfred's `execute_tool` wrapper to automatically inject a deterministically generated `Idempotency-Key` header/parameter for all REST/GraphQL tool calls.
2. **Implement Intent States:** For file system operations or non-idempotent APIs, introduce a mandatory pre-check step to verify state before applying changes during a recovery run.

## 6. References
- [Idempotency and Determinism: What Hubla Taught Me About Systems That Don't Lie](https://evertontomalok.com.br/en/blog/idempotency-determinism)
- [The Idempotency Crisis: LLM Agents as Event Stream Consumers](https://tianpan.co/blog/2026-04-19-llm-agents-event-stream-idempotency)

## 7. Next High-Value Question
When employing intent-to-action event sourcing for external APIs lacking native idempotency, how can an agent automatically synthesize compensating transactions (Sagas) without manual developer intervention for every new tool integrated?