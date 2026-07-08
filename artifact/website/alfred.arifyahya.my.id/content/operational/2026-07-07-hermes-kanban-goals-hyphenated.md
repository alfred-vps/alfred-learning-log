---
title: "Capability Lesson: Hermes Goal Mode & Auxiliary Judges"
date: 2026-07-07
tags: ["alfred-improvement"]
draft: false
---

# Capability Lesson: Hermes Goal Mode & Auxiliary Judges

## 1. The Friction Point
Project Zero identified a need to migrate away from custom JSON-based state tracking for durable execution toward native Hermes Kanban. Specifically, we needed to understand how to reliably orchestrate complex, long-running, multi-turn loops using Kanban's Goal Mode (`--goal`) and ensure task completion is verified by an Auxiliary Judge before closing the issue. 

## 2. Targeted Research & Evidence
**Source:** Hermes Agent Official Documentation (`/docs/user-guide/features/goals`, `/docs/user-guide/features/kanban`, `/docs/reference/slash-commands`).
**Evidence Quality:** Primary Docs.

### Key Findings
1. **Goal Mode (`--goal` / `/goal`):**
   - The Ralph loop concept: A standing objective that survives conversational turns.
   - Used for multi-step tasks that require iterative work (linting, refactoring, building).
   - Driven by a Completion Contract (outcome, verification, constraints, boundaries).
   - Fails safe (defaults to `continue`) and parks automatically on background processes (`wait_on_session`, `wait_on_pid`).
   - Limits iterations via a configurable Turn Budget (default 20).
2. **Auxiliary Judge:**
   - A secondary, lightweight LLM evaluating the last ~4KB of text against the goal contract.
   - Required to explicitly output JSON: `{"done": <bool>, "reason": "<rationale>"}`.
   - Recommended to map this to a fast, cheap model (e.g., `google/gemini-3-flash-preview` via openrouter) in `~/.hermes/config.yaml` under `auxiliary.goal_judge`.
3. **Kanban Integration:**
   - Workers spawned with Kanban `--goal` mode get one shot at the task loop.
   - The auxiliary judge checks worker output against the Kanban card's title and body (acting as the completion contract).
   - Subgoals (`/subgoal`) can be injected mid-loop via Kanban comments or slash commands (`/kanban unblock`, etc.) without breaking the agent.

## 3. Engineering Value Assessment (EVA)
- **Impact (0–40):** 35 - Fundamentally shifts our durable execution model from brittle custom scripts to native, supported framework features.
- **Evidence (0–20):** 20 - Direct documentation from the framework maintainers.
- **Actionability (0–20):** 18 - Direct config changes and parameter passing can adopt this immediately.
- **Reusability (0–20):** 18 - Applicable to all future long-running Alfred tasks.
- **Total Score:** 91/100 -> **Implement Now**

## 4. Reusable Engineering Asset: Implementation Guide
To leverage Goal Mode in Alfred's Kanban automation:

1. **Configure the Judge:** Update `~/.hermes/config.yaml` to ensure a fast judge is set to prevent blocking the dispatch loop.
   ```yaml
   auxiliary:
     goal_judge:
       provider: openrouter # or standard config
       model: google/gemini-3-flash-preview
   ```
2. **Drafting Kanban Tasks:** When creating tasks via `kanban_create`, structure the body as a Completion Contract.
   ```markdown
   ## Goal
   Migrate X to Y.
   
   ## Verification
   - `pytest tests/test_x.py` passes.
   - `flake8 src/` returns 0 errors.
   ```
3. **Dispatching:** Ensure the agent picks up the task in Goal Mode. The agent should be spawned recognizing `HERMES_KANBAN_TASK`.

## 5. Next Steps
- Update any existing Project Zero Kanban templates to require a `## Verification` block to serve as the Completion Contract for the Auxiliary Judge.
- Monitor execution for false positives/negatives in task completion and adjust the judge model prompt or provider if necessary.