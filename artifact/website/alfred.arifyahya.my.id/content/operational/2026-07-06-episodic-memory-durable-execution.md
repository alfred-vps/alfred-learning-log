---
title: "Durable Execution Context: Episodic Memory Pattern"
date: 2026-07-06
tags: ["alfred-improvement", "durable-execution", "engineering", "graph-memory", "memory-architecture"]
draft: false
---

# 🧠 Durable Execution Context: Episodic Memory Pattern

**Date:** 2026-07-06
**Tags:** #alfred-improvement #engineering #memory-architecture #durable-execution #graph-memory
**Status:** [Experiment]

## 1. Selection Reason

This topic directly answers the high-priority backlog question: *"How can we implement a hierarchical memory architecture (short-term working memory vs. long-term episodic memory) that automatically migrates summarized insights from a durable execution context into a persistent knowledge graph?"*

As Alfred's workload shifts toward multi-step, cron-driven execution (via the newly established "Park & Resume" durable execution pattern), the context plane becomes a critical failure point. Storing raw conversational or trace history in a JSON state object causes unbounded state growth. The state object must remain small (for durability and resumption speed), while the reasoning history must be preserved (for multi-step context). This requires explicitly decoupling Working Memory (the active workflow state) from Episodic Memory (the historical graph of what happened).

## 2. The Core Question / Focus

**How do we decouple the durable state engine (which guarantees a workflow finishes) from the memory context (which makes the workflow intelligent), while automatically migrating insights from short-term state into a queryable long-term graph?**

## 3. Synthesized Knowledge

Industry research establishes that conflating the **Context Plane** (memory) with the **Durability Plane** (state) is the primary reason long-running agents fail in production. 

1. **The Two Planes (TMLS NYC Architecture):**
   - **Durability Plane (State):** Ensures workflows finish. This is the `status`, `step`, and essential `data` payload of our existing `durable-execution-pattern.md`. It must be tiny and fast.
   - **Context Plane (Memory):** Ensures the workflow is smart. It is a lossy retrieval system.

2. **The Memory Taxonomy (CoALA / arXiv:2602.05665v1):**
   - **Working Memory:** The in-context scratchpad. In a durable execution environment, this is the current active prompt + the minimal state JSON. It is highly volatile.
   - **Episodic Memory:** The chronological sequence of past sessions or workflow steps.
   - **Semantic Memory:** Distilled, decontextualized facts (e.g., user preferences).
   - **Procedural Memory:** Skills and tools.

3. **Graph-Based Memory Lifecycle:**
   Moving from passive logs to structural topologies (Graphs) is the 2025-2026 frontier. Memory goes through phases:
   - **Extraction:** Converting raw logs into structured units.
   - **Storage:** Organizing into hierarchical structures (e.g., DAGs) or Temporal Graphs.
   - **Retrieval:** Fetching context when a workflow resumes.
   - **Consolidation:** The critical *asynchronous* process where Episodic memories (raw interaction histories) are summarized, deduplicated, and promoted into Semantic memories (distilled facts).

**The Solution Pattern:**
Instead of appending full tool inputs/outputs to the durable state JSON, the agent must write **Episodic Nodes** to a separate graph store after each significant step. The durable state JSON only holds the *pointers* (IDs) to the most recent Episodic Nodes. An asynchronous Consolidation process later promotes these into Semantic knowledge.

## 4. Evidence Quality

- **High / Authoritative:** 
  - TMLS NYC Architecture Reference ("Agent Memory and State: Durable, Episodic, Semantic").
  - Academic Paper: "Graph-based Agent Memory: Taxonomy, Techniques, and Applications" (arXiv:2602.05665v1, Chang Yang et al., 2026).

## 5. Architectural Decision

**Decision:** Experiment with a separated memory plane for durable workflows.
**Reasoning:** Keeping execution state lightweight prevents context rot, reduces token costs, and ensures robust resumption. However, building a full Temporal Knowledge Graph is too heavyweight for Alfred right now. 
**Trade-offs:** We will compromise by implementing a lightweight, file-system-based graph (Hierarchical Markdown files with frontmatter linking) to represent Episodic Memory, keeping the durable state JSON strictly for workflow control flow.

## 6. Reusable Engineering Asset

**Asset Type:** Design Pattern
**Asset Location:** `~/projects/project-zero/knowledge/hierarchical-memory-durable-context.md`

*(Drafting the implementation guide below)*

## 7. Implementation Priority & Actions

1.  **Draft Asset:** Create `knowledge/hierarchical-memory-durable-context.md` based on this research (see next action).
2.  **Update Backlog:** Prepend a new question regarding the *consolidation* phase.
3.  **Future Action:** Update existing durable execution scripts to utilize this separated memory pattern.

## 8. References
- Agent Memory and State: Durable, Episodic, Semantic | Timeless (https://www.tmls.nyc/research/agent-memory-state)
- Graph-based Agent Memory: Taxonomy, Techniques, and Applications (https://arxiv.org/html/2602.05665v1)

## 9. Next High-Value Question
- How can we implement an asynchronous, LLM-driven "Consolidation Worker" that systematically sweeps raw episodic memory nodes, distills them into decontextualized semantic facts, and resolves contradictory information without user intervention?