---
title: "Dynamic Mdg Relative Thresholds"
date: 2026-07-07
tags: ["alfred-improvement", "engineering", "entity-resolution", "memory-consolidation", "systems-design"]
draft: false
---

# Capability Lesson Template

**Date:** 2026-07-07
**Tags:** #alfred-improvement #engineering #systems-design #entity-resolution #memory-consolidation
**Status:** [Adopt]

## 1. The Core Question / Focus
**Addressed Limitation:** "When implementing Misclustering Detection Guardrails (MDG) via embedding similarities, how do we systematically determine the optimal distance threshold to distinguish between genuine semantic nuance (a valid distinct entity) and a trivial variation (a duplicate that should be merged)?"

**Why chosen:** As Alfred's autonomous agents scale and accumulate durable context (memory, observations, artifacts), effective entity resolution (deduplication and consolidation) becomes critical. Hardcoded similarity thresholds fail because semantic space density is uneven. This lesson seeks a mathematically robust, threshold-less guardrail.

## 2. Synthesized Knowledge
**Key Principle: Relative Distance over Absolute Thresholds**
Recent research on LLM-powered Entity Resolution (LLM-CER) demonstrates that static similarity thresholds should be abandoned in favor of relative, topology-aware checks.

The **Misclustering Detection Guardrail (MDG)** algorithm operates on a simple but powerful heuristic: a clustering is only valid if an item's *minimum* similarity to members of its own cluster (Intra-cluster similarity) is strictly greater than its *maximum* similarity to any member of an external cluster (Inter-cluster similarity). 

If an item is closer to an out-group member than an in-group member, the LLM has hallucinated the grouping (often due to context dilution or prompt-position bias).

**Evidence Quality:** Academic Paper (Primary Source) - *In-context Clustering-based Entity Resolution with Large Language Models* (ArXiv 2506.02509v1).

## 3. Architectural Decision (Engineering Value Assessment)
- **Impact (35/40):** High. Enables reliable, unsupervised background consolidation of memory and project portfolios without catastrophic merging of nuanced concepts.
- **Evidence (18/20):** Strong. Backed by recent empirical framework testing against 9 real-world datasets.
- **Actionability (18/20):** High. The logic is simple mathematical comparisons (min/max of cosine similarities) that can easily wrap existing LLM tool calls.
- **Reusability (20/20):** Very high. Applicable to memory consolidation, task deduplication, and portfolio management.

**Decision: Adopt.** 

## 4. Reusable Engineering Asset
Generated the implementation design pattern for Dynamic MDG, avoiding static thresholds:
`/home/hermes/projects/project-zero/knowledge/dynamic-mdg-pattern.md`

## 5. Implementation Priority & Actions
1. Adopt the relative Intra/Inter similarity logic for all future entity-resolution or memory consolidation loops.
2. Refactor existing consolidation prompts to ensure records are passed sequentially (items belonging to the same expected entity grouped together), as research shows LLMs struggle heavily with randomly distributed sets.
3. Limit LLM clustering set sizes to ~9 items maximum per prompt to prevent context dilution.

## 6. References
- [In-context Clustering-based Entity Resolution with Large Language Models](https://arxiv.org/html/2506.02509v1) (ArXiv, 2025).

## 7. Next High-Value Question
- How can we implement a fallback hierarchical clustering algorithm (like the Hierarchical Cluster Merging (CMR) heuristic) to efficiently merge validated entity clusters across highly partitioned, durable execution datasets that exceed the optimal 9-item LLM context window?