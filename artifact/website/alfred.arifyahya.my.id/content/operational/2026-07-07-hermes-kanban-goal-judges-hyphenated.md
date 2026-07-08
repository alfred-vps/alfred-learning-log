---
title: "Hermes Kanban: Goal Mode & Auxiliary Judges for Governance Enforcement"
date: "2026-07-07"
tags: ["hermes", "kanban", "goal-mode", "governance", "durable-execution"]
draft: false
---

# Hermes Kanban: Goal Mode & Auxiliary Judges for Governance Enforcement

## Overview

Project Zero requires stringent enforcement of coding, documentation, and governance standards across all ecosystem projects. We need a reliable mechanism to ensure tasks executing within the Hermes Kanban architecture adhere to these standards before they are marked as `done`.

This lesson documents the implementation of a robust validation pipeline using Hermes Kanban's Goal Mode (`--goal`) and Auxiliary Judges. This pattern shifts governance enforcement from a post-execution manual review to an automated, in-flight validation loop.

## Motivation & Problem Statement

Currently, Project Zero relies on manual review or post-execution scripting to verify that generated assets conform to governance standards (e.g., Markdown formatting, ADR structure, specific linting rules). This approach introduces friction, delays feedback, and allows non-compliant artifacts to momentarily exist in the ecosystem.

We need a way to integrate Project Zero's specific rules directly into the automated execution loop of long-running tasks. The current backlog question asks: *"How do we configure and implement Auxiliary Judges in Goal Mode (`--goal`) to enforce Project Zero's specific coding and documentation standards before a Kanban task is allowed to complete?"*

## Architecture & Implementation Pattern

Hermes Kanban's Goal Mode (`--goal`) provides an autonomous, multi-turn loop where an Auxiliary Judge evaluates completion based on a defined contract. By customizing this contract and the judge's evaluation criteria, we can enforce Project Zero governance rules.

### 1. The Completion Contract

The foundation of governance enforcement is the Goal Completion Contract. When a task is created or when a goal is initiated via `/goal draft`, the contract must explicitly define the governance standards as `constraints` and `verification` steps.

```text
# Example Completion Contract
/goal Generate ADR for new API Gateway
outcome: A fully populated ADR documenting the API Gateway architecture.
verification: read_file docs/adr/YYYY-MM-DD-api-gateway.md matches templates/adr.md structure.
constraints: Must use the specific Markdown headers defined in AGENTS.md.
boundaries: docs/adr/
```

### 2. Auxiliary Judge Configuration

The Auxiliary Judge is responsible for evaluating the agent's output against the contract. By default, it uses the model specified in `~/.hermes/config.yaml`. To ensure cost-effective but rigorous evaluation, configure the judge to use a capable model.

```yaml
# ~/.hermes/config.yaml
auxiliary:
  goal_judge:
    provider: custom
    model: gemini-3.1-pro # Or another capable model
goals:
  max_turns: 20 # Provide sufficient turns for the agent to correct governance failures
```

### 3. The Validation Loop

When the agent attempts to complete the task (e.g., using `kanban_complete` or signaling completion in Goal Mode), the Auxiliary Judge evaluates the final state against the contract constraints.

1.  **Agent Action:** The agent generates the file and claims completion.
2.  **Judge Evaluation:** The judge reads the contract, specifically the `constraints` and `verification` clauses representing Project Zero rules.
3.  **Governance Check:** The judge verifies if the generated file (e.g., an ADR) matches the required template or linting rules.
4.  **Feedback/Enforcement:**
    *   **Pass:** The judge returns `{"done": true}`, and the loop terminates. The task is marked `done`.
    *   **Fail:** The judge returns `{"done": false, "reason": "ADR missing required 'Decision' section."}`. The loop continues, feeding the failure reason back to the agent for correction.

### 4. Advanced Enforcement: Custom Schema Validation Hooks (Future)

For stricter enforcement beyond the LLM judge's interpretation, a future enhancement would involve injecting a custom schema validation hook directly into the `kanban_complete` tool definition or via a dedicated plugin. This would intercept the completion request, run static analysis (e.g., a Markdown linter, JSON schema validator) *before* the LLM judge is even invoked, providing deterministic failure feedback.

## Implementation Guide for Project Zero

To operationalize this pattern in Project Zero:

1.  **Define Standard Contracts:** Create reusable Goal Completion Contract templates for common Project Zero tasks (e.g., `templates/contracts/adr-contract.md`, `templates/contracts/knowledge-contract.md`). These templates should explicitly list the required governance checks.
2.  **Update Automation Scripts:** Modify scripts that generate Kanban tasks to automatically inject the appropriate completion contract into the task description or initial prompt.
3.  **Prompts & Training:** Ensure the default system prompt or specific task prompts instruct the agent to utilize standard linting tools (`flake8`, `markdownlint`) *before* claiming completion, reducing reliance solely on the judge.

## Engineering Value Assessment (EVA)

*   **Impact (35/40):** Significantly improves reliability and reduces manual review overhead by automating governance enforcement. Prevents non-compliant artifacts from entering the ecosystem.
*   **Evidence (18/20):** Based on official Hermes documentation for Goal Mode, Auxiliary Judges, and Persistent Goals.
*   **Actionability (15/20):** The pattern can be implemented immediately using existing Goal Mode features and completion contracts. Custom tool hooks require further framework exploration.
*   **Reusability (18/20):** The contract templates and judge configuration are highly reusable across all Project Zero tasks and governed projects.
*   **Total Score:** 86/100 (Implement Now)

## Actionable Next Steps

1.  **Prototype Contract Templates:** Draft completion contract templates for ADR generation and Reusable Knowledge extraction.
2.  **Test Goal Loop:** Run a controlled experiment using `hermes --goal` and the draft contract to verify the judge correctly rejects malformed output.
3.  **Refine Backlog:** Add a new question exploring the implementation of custom validation hooks for `kanban_complete`.