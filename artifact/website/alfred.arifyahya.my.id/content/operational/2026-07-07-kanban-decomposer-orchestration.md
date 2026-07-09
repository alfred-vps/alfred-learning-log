---
title: Hermes Capability Lesson
date: 2026-07-07
tags: [alfred-improvement, hermes, kanban, decomposer, orchestration, "curriculum:hermes"]
draft: false
---

## 1. Topic Studied
Hermes Kanban Auxiliary Decomposer: Configuring and evaluating the `auxiliary.kanban_decomposer` to ensure automated task routing accurately reflects Alfred's governed project structure without over-delegating.

## 2. Official Sources Consulted
- Primary Docs: Kanban tutorial (Auto vs Manual orchestration)
- Primary Docs: Kanban (Multi-Agent Board) Summary
- Primary Docs: CLI Commands Reference

## 3. Key Concepts Learned
- **Auto Orchestration Context:** When tasks land in the `triage` column on the Kanban board, they are automatically processed if the `auxiliary.kanban_decomposer` is configured.
- **Decomposer Execution:** The decomposer uses a built-in decomposition prompt *plus* the configured model path (`auxiliary.kanban_decomposer` in `config.yaml`). It reads the currently installed profiles and their descriptions to fan out the task into a graph of child tasks routed to those specialists.
- **Root Ownership:** The setting `kanban.orchestrator_profile` controls which profile owns the root/orchestration task *after* the fan-out has occurred, not who performs the decomposition.
- **Manual Orchestration:** If auto-decomposition is not desired, tasks stay in `triage` until manually processed via the Dashboard UI (⚗ Decompose or ✨ Specify) or CLI (`hermes kanban decompose`).

## 4. Best Practices Discovered
- **Profile Descriptions are Critical:** Because the decomposer relies on installed profiles and their descriptions to route tasks, these descriptions must clearly define the *scope* and *boundary* of each profile to prevent misrouting or over-delegation.
- **Avoid Over-Delegation:** The decomposer can generate too many granular child tasks if profiles are too narrowly defined or if the root task is too generic.
- **Tuning the Model:** Ensure a highly capable reasoning model is assigned to `auxiliary.kanban_decomposer` as this is a complex planning and routing task.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** We have been migrating towards Kanban workflows (as noted in `AGENT-REVIEW.md`) but have not explicitly mapped our governance model or profiles to the decomposer's routing logic.
- **Gaps & Anti-Patterns:** Without tightly scoped profile descriptions, the default decomposer behavior might fragment tasks in ways that don't align with Project Zero's "Governed Project" boundaries, leading to tasks assigned to incorrect or non-existent roles.

## 6. Recommended Improvements
1. **Audit Profile Descriptions:** Review all active Hermes profiles under Alfred's control to ensure their descriptions accurately reflect their roles within the Project Zero governance model. Update descriptions to be strict about what they *should* and *should not* handle.
2. **Configure Decomposer Model:** Verify that `config.yaml` explicitly sets a high-quality model for `auxiliary.kanban_decomposer`.
3. **Prototype Decomposition Limits:** Create a test board/task to observe the decomposer's fan-out behavior. Document guidelines in `knowledge/kanban-adoption-blueprint.md` on how to structure `triage` tasks to yield optimal child graphs.

## 7. Risk Assessment
- **Misalignment with Governance:** If the decomposer creates tasks that violate the "Governed Project" structure (e.g., creating a new project instead of extending an asset), it creates clean-up work.
- **Token Costs:** Auto-decomposition of every triage item using a large model can be expensive if the incoming task volume is high.

## 8. Next Learning Topic
How do we seamlessly transition an active POSIX-file-based durable execution task into a suspended, memory-reclaimed state that can be automatically awakened by an external webhook event without relying on a continuously polling daemon?
