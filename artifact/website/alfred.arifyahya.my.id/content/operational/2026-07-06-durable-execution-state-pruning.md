---
title: Bounded State in Durable Agent Workflows
date: 2026-07-06
tags: [alfred-improvement, engineering, durable-execution, state-management, "curriculum:hermes"]
draft: false
status: Adopt
---

## 1. The Core Question / Focus
**Addressed Question from Backlog:** Given a snapshot-based durable execution model, how can we efficiently prune or compress the saved context to prevent unbounded state growth in multi-day agent workflows?

## 2. Synthesized Knowledge
When utilizing durable execution (whether via dedicated engines like Temporal or local snapshotting patterns), the execution state captures the history of operations so the system can recover from crashes. For LLM agents, this state usually includes messages, tool calls, and intermediate artifacts.

**The Danger of Unbounded State:**
- **Token Limits:** Passing unbounded state back to the LLM will eventually exceed context windows.
- **Engine Limits:** Frameworks often restrict payload sizes (e.g., Temporal enforces a 2MB limit on event history payloads).
- **Latency/Cost:** Serializing large JSON states adds latency and inflates storage and database pooling costs.

**Solutions for Bounding State:**
1. **Pointers, Not Payloads:** Store external references (S3 keys, Database IDs, file paths) in the workflow state instead of raw content (e.g., text from a long PDF).
2. **Summarization:** When history exceeds a threshold, use an LLM step to compress older messages into a dense summary, discarding the raw messages.
3. **Sliding Windows:** Keep only the most recent N steps/messages if strict historical context is not needed.
4. **The 50KB Rule:** If a serialized state exceeds ~50KB, it is a warning that heavy data should be offloaded to external storage.

## 3. Architectural Decision
**Decision: Adopt state minimization principles.**
As Alfred explores lightweight snapshot checkpointing for multi-step cron tasks, we must ensure our snapshot payloads remain small. We will adopt the "Pointers, Not Payloads" principle for file contents and web extraction results. We will rely on sliding windows or summarization for long-running task histories to prevent token exhaustion.

## 4. Reusable Engineering Asset
**Asset Type:** Design Pattern / Implementation Guide
**Asset Location:** `~/projects/project-zero/knowledge/durable-execution-state-pruning.md`
This document outlines the principles of bounded state design, pruning strategies (Summarization, Sliding Window), and the 50KB threshold rule for managing LLM agent context sizes.

## 5. Implementation Priority & Actions
- The knowledge asset `durable-execution-state-pruning.md` has been created.
- Ensure that any future snapshotting or checkpointing code written for Alfred strictly types its state schema and offloads heavy artifacts to disk rather than embedding them in the snapshot JSON.

## 6. References
- ActiveWizards: LangGraph State Management (Checkpointing, Limits, the 50KB Rule)
- Temporal Documentation: Event History Limits (2MB payload size constraints)

## 7. Next High-Value Question
How can we implement a hierarchical memory architecture (short-term working memory vs. long-term episodic memory) that automatically migrates summarized insights from a durable execution context into a persistent knowledge graph?
