---
title: Implementing Metadata-Driven Validation in Hermes Kanban
date: 2026-07-08
status: Adopted
tags:
  - architecture
  - kanban
  - goal-mode
  - validation
draft: false
---

# North Star Lesson: Implementing Metadata-Driven Validation in Hermes Kanban

## 1. Strategic Selection

**Topic:** How can we implement a custom schema validation hook within the `kanban_complete` tool to enforce strict Project Zero asset formatting before the auxiliary judge even evaluates the goal completion status?

**Rationale:** As Alfred transitions to a Workflow-First Architecture using Hermes Kanban (as outlined in `AGENT-REVIEW.md` and previous lessons), we must ensure that completed tasks adhere to strict Project Zero governance rules. Relying solely on the auxiliary judge for formatting verification is inefficient and prone to AI hallucination. By leveraging machine-readable handoffs in `kanban_complete` metadata, we can perform deterministic pre-validation, reducing judge overhead and guaranteeing structural compliance.

## 2. Research Findings

**Methodology:** Reviewed Hermes Kanban documentation and slash command references.
**Evidence Quality:** High (Primary Docs).

**Key Discoveries:**
1.  **Kanban Metadata Field:** The `kanban_complete` tool supports a `metadata` JSON field.
2.  **Machine-Readable Handoffs:** The documentation explicitly recommends using this `metadata` field for machine-readable handoffs, including arrays for `changed_files`, `verification`, and `dependencies`.
3.  **Goal Mode Mechanics:** In Goal Mode (`--goal`), an auxiliary judge evaluates the task output against the criteria.
4.  **Pre-Judge Validation Opportunity:** By structuring the output and verification steps explicitly within the `metadata` JSON, we create a contract. While the judge ultimately decides, a deterministic script can intercept or preemptively validate this metadata *before* the judge's final verdict, or the judge can be instructed to strictly evaluate the deterministic evidence provided in the metadata.

## 3. Strategic Guidance

**Asset Produced:** Implementation Guide for Kanban Metadata Validation.

**Strategic Decision:** Enforce Metadata Contracts for Governed Tasks.

For all Project Zero governed tasks executed via Kanban Goal Mode, workers MUST use the `kanban_complete` metadata field to provide deterministic proof of formatting and structural compliance.

## 4. Engineering Asset: Implementation Guide

### Kanban Metadata Validation Pattern

When a worker completes a task that requires strict structural formatting (e.g., generating an ADR or modifying a portfolio entry), it must provide verifiable evidence in the `kanban_complete` metadata.

**1. The Contract (Worker Side)**

The worker must run a local validation script (e.g., `scripts/utilities/validate-adr.sh`) and include the execution command and its successful output reference in the metadata.

```json
// Example kanban_complete payload
{
  "task_id": "t_abc123",
  "result": "Drafted new ADR for Kanban validation.",
  "metadata": {
    "changed_files": ["docs/adr/0015-kanban-metadata.md"],
    "verification": ["scripts/utilities/validate-adr.sh docs/adr/0015-kanban-metadata.md"],
    "structural_compliance": "pass",
    "residual_risk": null
  }
}
```

**2. The Evaluation (Judge Side)**

The Auxiliary Judge's system prompt (or the Goal criteria itself) must be explicitly configured to require this deterministic proof:

*   **Goal Criteria Addition:** "The task is ONLY complete if the `kanban_complete` metadata contains `\"structural_compliance\": \"pass\"` AND lists the exact validation command run under the `verification` key."

**3. Future Enforcement (System Side)**

As Hermes capabilities expand, this metadata schema can be strictly validated by a pre-completion hook or a lightweight deterministic wrapper around the Kanban event bus, ensuring the judge is never invoked if the structural contract is violated.

## 5. The "Better Alfred" Test

**How does this help Alfred become a better Alfred?**
This pattern bridges the gap between probabilistic AI output and deterministic engineering standards. By forcing workers to prove compliance via metadata before completion, we reduce the burden on the auxiliary judge, minimize formatting regressions in Project Zero assets, and establish a rigorous, auditable trail of verification for every automated task.
