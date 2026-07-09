---
title: "How the Hermes `todo` Tool Works and When Project Zero Should Use It vs Kanban"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "todo", "kanban", "agent-loop", "curriculum:hermes"]
draft: false
status: Published
---

## 1. Topic Studied
The Hermes `todo` tool — its documented purpose, toolset placement, and (critically) its status as an **agent-loop tool** that is handled directly by the agent loop rather than dispatched through the central tool registry. Analysis of when Alfred/Project Zero should use `todo` for in-session multi-step tracking versus the Kanban system for durable, cross-session work.

## 2. Official Sources Consulted
- Built-in Tools Reference — lists `todo` among standalone tools and in the `todo` toolset. https://hermes-agent.nousresearch.com/docs/reference/tools-reference
- Tools & Toolsets — classifies `todo` under "Agent orchestration" and lists `todo` as a common toolset. https://hermes-agent.nousresearch.com/docs/user-guide/features/tools
- Tools Runtime — states `todo` is an agent-loop tool handled directly by the agent loop (alongside `memory`, `session_search`, `delegate_task`). https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime
- Cron job note: "Cron runs in fresh sessions with no current-chat context." https://hermes-agent.nousresearch.com/docs/reference/tools-reference

## 3. Key Concepts Learned
- **Purpose:** `todo` manages a task list *for the current session*. Official guidance: "Use for complex tasks with 3+ steps or when the task is non-trivial."
- **Toolset:** `todo` is a standalone built-in tool living in the `todo` toolset. It is enabled like any other toolset (`hermes chat --toolsets "...todo..."` or `hermes tools`).
- **Agent-loop handling:** Unlike registry-dispatched tools (`terminal`, `web_search`, etc.), `todo` is intercepted and handled inside the agent loop itself. The Tools Runtime doc explicitly groups it with `memory`, `session_search`, and `delegate_task` as tools the central dispatcher routes to the agent loop rather than a normal handler. This means its state (the task list) lives in the session/agent-loop context, not in the tool registry.
- **Session-scoped, not persistent:** Because the task list is part of the session, it does not survive across sessions. The cron reference confirms cron jobs "run in fresh sessions with no current-chat context" — so a `todo` list created in one cron run is gone on the next.

## 4. Best Practices Discovered
- Use `todo` for **in-session** multi-step tracking: complex interactive tasks with 3+ steps where lightweight visible progress helps the current conversation.
- Do **not** use `todo` for work that must persist, be audited, be shared across sessions, or involve human-in-the-loop gating — that is the Kanban system's job.
- `todo` is a reasoning/planning aid for the *current* agent loop, not a durable store. Treat its contents as ephemeral.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero usage:** Project Zero governs multi-step, cross-session, auditable work through the Kanban system (goals, tasks, judges, webhook resumption — see prior lessons). The Self-Improvement Loop and publish cron are fresh-session jobs. There is no evidence `todo` is currently used anywhere; in-session progress in cron jobs is implicit in the agent's own reasoning, not a managed list.
- **Gaps & Anti-Patterns:**
  - Risk if we adopted `todo` inside cron jobs: a false sense that task state persists. It does not — every cron invocation is a fresh session.
  - Conversely, using Kanban for purely ephemeral in-session bookkeeping would add unnecessary durable-infrastructure overhead for transient work.

## 6. Recommended Improvements
1. **Keep Kanban as the system of record for durable/cross-session work** (governed-project lifecycle, backlog, judge-gated goals). No change needed — this remains correct.
2. **Optionally adopt `todo` only inside long interactive (non-cron) sessions** where Alfred is doing a 3+ step reasoning task in a single conversation and wants visible checkpoints. Document this in `knowledge/` so future sessions don't mistake it for persistence.
3. **Add a one-line note to AGENTS.md / `knowledge/`**: "`todo` is session-scoped and is wiped on every cron run — never use it for durable tracking; use Kanban + file-based knowledge instead." This closes the single genuine ambiguity the backlog flagged (todo vs Kanban for in-session tracking).

## 7. Risk Assessment
- **Limitation of native feature:** `todo` state is lost between sessions. Any reliance on it for auditability, resumption, or cross-run coordination will silently fail. Mitigation: enforce Kanban for those cases.
- **No breakage risk** from the recommendation: it merely clarifies scope and does not propose moving existing durable work onto `todo`.

## 8. Next Learning Topic
This was the last Hermes-native topic in the rebuilt v2 backlog. With `todo` now [Addressed], the v2 Hermes-doc-backed queue is exhausted. Per the loop rules, the next run should output [SILENT] unless a new Hermes-documented capability surfaces. Candidate watch-list (requires fresh doc discovery, not invented): the `/learn` skill-creation flow in practice, or agent-loop tool internals for `memory`/`session_search` persistence mechanics.

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
