---
title: "Capability Lesson: Automated Evaluation & Resolution in Knowledge Graphs"
date: 2026-07-07
tags: ["alfred-improvement"]
draft: false
---

# Capability Lesson: Automated Evaluation & Resolution in Knowledge Graphs

## 1. Problem Statement
Alfred requires a robust, hands-off mechanism to evaluate the accuracy of its background memory consolidation pipeline. Specifically, we need to detect memory drift, retrieval misses, and entity resolution failures without requiring a human-in-the-loop to manually grade the consolidated insights. Relying solely on LLM absolute scoring (1-5 scales) introduces variance and bias.

## 2. Approach & Evidence
We evaluated recent academic papers and engineering methodologies regarding `LLM-as-a-judge` paradigms and Large Language Model Entity Resolution (LLM-CER). 

**Sources:**
1. *A Survey on LLM-as-a-Judge* (arXiv:2411.15594v4) - Primary Research
2. *In-context Clustering-based Entity Resolution with Large Language Models* (arXiv:2506.02509v1) - Primary Research

**Key Findings:**
*   **LLM-as-a-Judge Biases:** LLMs exhibit severe position, verbosity, and concreteness biases when evaluating unstructured text.
*   **Pairwise > Absolute:** LLMs are highly inaccurate at absolute grading but highly consistent at relative, pairwise comparisons.
*   **In-Context Clustering:** Shifting from $O(N^2)$ pairwise matching to in-context clustering (grouping multiple records in a single prompt) reduces API costs by up to 108x while improving accuracy. However, prompt size must be strictly bounded (max ~9 records for GPT-4 class models, targeting ~4 distinct entities).
*   **Misclustering Detection Guardrail (MDG):** An automated evaluation step that uses deterministic embedding similarity (Cosine/Jaccard) to flag LLM resolution hallucinations by comparing intra-cluster vs. inter-cluster similarity.

## 3. Engineering Value Assessment (EVA)
*   **Impact (35/40):** Solves the scaling and cost bottleneck for automated long-term memory maintenance and drift detection.
*   **Evidence (18/20):** Supported by recent, high-quality empirical studies on LLM behavior.
*   **Actionability (15/20):** MDG and in-context clustering limits are easily implementable using existing embedding/LLM tools.
*   **Reusability (20/20):** Highly applicable to any future RAG, memory consolidation, or data distillation pipeline.
*   **Total:** 88/100 (High Value)

## 4. Implementation Guidelines / Rules
1.  **Never Use Pairwise Entity Matching:** When consolidating memory or resolving entities, use chunked In-Context Clustering prompts containing a maximum of 9 records and targeting approximately 4 distinct entities.
2.  **Implement MDG for Automated QC:** Do not trust the LLM's entity resolution blindly. Embed the raw inputs and the consolidated output. Reject the consolidation if a raw input is mathematically closer to a different cluster than its assigned one.
3.  **Evaluate via Relative Scoring:** When designing an LLM judge to evaluate drift, ask it to compare two states (e.g., `State A` vs `State B`) rather than assigning a generic accuracy score to a single state. Randomize the order of A and B to prevent position bias.
4.  **Enforce JSON Schemas:** Always use constrained decoding or JSON schemas for the judge's output to prevent verbosity bias.

## 5. Artifacts Produced
*   `knowledge/llm-automated-evaluation-patterns.md`: Detailed architectural design pattern for implementing automated LLM evaluation and clustering-based entity resolution.