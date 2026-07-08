---
title: "Production-Grade Prompt Evaluation and Registry Pipelines"
date: 2026-07-08
status: Adopted
tags:
  - architecture
  - prompts
  - evaluation
  - mlops
draft: false
---

# North Star Lesson: Production-Grade Prompt Evaluation and Registry Pipelines

## 1. Strategic Selection

**Topic:** How can we automate the evaluation and promotion of prompt versions within a Git-backed Prompt Registry, ensuring that regressions are caught before a new prompt reaches the Production status? (Selected from `limitations_backlog.md`).

**Rationale:** As Alfred scales, managing prompts manually through "gut feel" trial and error becomes a massive liability. Silent regressions in system prompts can cascade through long-running Kanban workflows, corrupting data or breaking structured JSON contracts. Treating "prompts as code" is insufficient without the CI/CD rigor of "prompts as tested code." We need a deterministic, metric-driven pipeline to gate prompt promotions.

## 2. Research Findings

**Methodology:** Reviewed industry practices for quantitative prompt evaluation, MLOps integration (like MLflow), and statistical testing requirements for LLM outputs.
**Evidence Quality:** High (Industry engineering blogs, MLOps framework design patterns, statistical modeling guidelines for LLMs).

**Key Discoveries:**
1.  **Prompts are Code, Requiring CI/CD:** Prompt changes must be gated by automated evaluations against a "golden dataset" (20-50 examples for basic regression, 100+ for statistical confidence).
2.  **Required Metric Triad:** Evaluations must concurrently measure Quality (Exact Match, LLM-as-a-Judge, Groundedness), System Health (Latency p95/p99, Failure rates), and Cost (Token usage).
3.  **Statistical Rigor over Averages:** Raw averages are misleading. Proving a prompt is "better" requires paired statistical tests (e.g., Wilcoxon) to ensure the lift isn't random noise.
4.  **Release Gating Pipeline:** 
    *   *Offline (CI/CD):* Test against the golden set. Block merge if primary metrics degrade.
    *   *Staging/Canary:* Run shadow traffic or 5% live traffic. Monitor latency and failure rates.
    *   *Production:* Continuous sampling (1-5%) to monitor for drift or safety violations.
5.  **LLM-as-a-Judge Constraints:** While scalable, using LLMs to judge other LLMs requires a stronger model (e.g., using GPT-4 to grade a smaller local model) and strict calibration to avoid verbosity/position bias. If human review and LLM judge disagree >20% of the time, the rubric is flawed.

## 3. Strategic Guidance

**Asset Produced:** Prompt Evaluation Pipeline Architecture (below).

**Strategic Decision:** Adopt a Metric-Driven Prompt Promotion Pipeline.

Alfred will enforce quantitative, automated evaluation for all system and skill prompts before they are promoted to Production status within the Git-backed registry.

**Architectural Implications:**
*   **Golden Datasets:** Every critical prompt in the registry MUST have an accompanying `.jsonl` golden dataset containing real-world edge cases and past failures.
*   **Automated Gating:** A Kanban task (or CI script) will execute upon prompt modification. It will run the new prompt against the golden dataset, compute the metrics (Accuracy, Latency, Cost, Format Adherence), and compare it to the current Production version.
*   **Immutable Versioning:** Prompts must be versioned immutably (e.g., via Git commit SHA or semantic versioning). The evaluation results must be permanently linked to that specific version ID.

## 4. The "Better Alfred" Test

**How does this help Alfred become a better Alfred?**
This eliminates "guesswork" and silent regressions when iterating on Alfred's core cognitive capabilities. It ensures that when we tweak a prompt to fix a specific bug (e.g., forcing better JSON formatting), we have mathematical proof that we didn't break 5 other capabilities in the process. This builds the trust required for long-term autonomous execution.

---

## 5. Engineering Asset: Prompt Evaluation CI/CD Pipeline Architecture

### Phase 1: Artifact Organization (The Registry)
Within the project repository (e.g., `project-zero/prompts/`), structure prompts alongside their tests:
```text
prompts/
├── judge/
│   ├── prompt.md         # The current development version
│   ├── golden_set.jsonl  # 50+ diverse test cases (inputs + expected outputs/criteria)
│   └── eval_rubric.md    # Instructions for the LLM-as-a-Judge
└── researcher/
    ├── prompt.md
    └── golden_set.jsonl
```

### Phase 2: The Evaluation Execution (The Script/Kanban Node)
When a PR is opened modifying `prompt.md`:
1.  **Execute:** Run the new prompt over all inputs in `golden_set.jsonl`.
2.  **Score:** 
    *   *Deterministic:* Check JSON schema adherence, exact string matches, or regex constraints.
    *   *Heuristic:* Use a stronger LLM (with `eval_rubric.md`) to grade subjective outputs on a 1-5 scale.
    *   *Telemetry:* Record latency per call and exact token usage.
3.  **Compare:** Fetch the historical scores of the current `Production` version on the same dataset.

### Phase 3: The Promotion Gates (The Decision)
*   **Gate 1 (Format):** 100% pass rate required for structural constraints (e.g., valid JSON).
*   **Gate 2 (Quality):** Statistical parity or improvement required on the primary quality metric (e.g., accuracy, usefulness). No regression allowed.
*   **Gate 3 (Cost/Latency):** Flag for human review if token cost or p95 latency increases by >15%, even if quality improves.

If all gates pass, the CI pipeline tags the prompt version as `Production-Ready`.