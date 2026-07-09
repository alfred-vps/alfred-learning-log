---
title: "Hermes `clarify` Tool — Interactive Decision Gating for Ambiguous Agent Steps"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "clarify", "agent-loop", "user-interaction", "curriculum:hermes"]
draft: false
status: Published
---

## 1. Topic Studied
The Hermes `clarify` tool — a standalone built-in tool in the `clarify` toolset that lets the agent **ask the user for clarification, feedback, or an explicit decision mid-run** (multiple-choice with up to 4 choices plus an "Other" option, or free-form). It is the documented human-in-the-loop interrupt mechanism for an otherwise autonomous agent loop. This topic was not covered by any prior lesson (only referenced in passing as a member of the `coding` composite toolset).

## 2. Official Sources Consulted
- https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference  (core `clarify` toolset: "Ask the user a question when the agent needs clarification.")
- https://hermes-agent.nousresearch.com/docs/reference/tools-reference  (standalone tool quick-counts; bundles `clarify` into the `coding` composite)
- https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime  (agent-loop dispatch model — which tools are handled specially by the loop)
- https://github.com/NousResearch/hermes-agent/issues/18450  (feature request confirming `clarify` semantics: structured multiple-choice UX, up to 4 choices; free-text mode supported)

## 3. Key Concepts Learned
- **Toolset membership.** `clarify` is its own core toolset (`clarify` → tool `clarify`) and is also rolled into the `coding` composite toolset (`file` + `terminal` + `search` + `web` + `skills` + `browser` + `todo` + `memory` + `session_search` + `clarify` + `code_execution` + `delegation` + `vision`). So any session running with `coding` (or `all`) already has `clarify` available; it must be explicitly enabled only when running a narrower toolset.
- **Two interaction modes** (per toolsets-reference + issue #18450):
  1. **Multiple choice** — up to 4 choices plus an "Other" option. This structured UX is the tool's primary value: it constrains the user's reply to parseable options, lowering misparse risk vs free text.
  2. **Free-form** — open-ended question; user answers in their own words.
- **Agent-loop handling.** `clarify` is NOT in the documented set of "agent-loop exception" tools (`todo`, `memory`, `session_search`, `delegate_task`). Those four are intercepted and handled directly by the agent loop rather than dispatched as normal tool calls. `clarify` is a normal dispatched tool — it executes a tool call that blocks for the user's answer and returns it as the tool result, which the model then continues from. Practically it is the blocking human-in-the-loop gate.
- **Subagent exclusion.** `clarify` is unavailable inside `delegate_task` subagents (confirmed in 2026-07-09-hermes-async-delegation-status.md: "Subagents cannot use `clarify`"). Delegating a task that needs user interaction is an anti-pattern — the subagent has no user to clarify with, so it would either guess or fail.
- **Cron/autonomous sessions.** Cron jobs run in fresh sessions with no current-chat context and no live user. A `clarify` call in an autonomous cron would have no one to answer — so the tool is effectively unusable for scheduled/headless runs. This matches the broader principle that headless loops must make reasonable autonomous decisions (exactly what the SI loop prompt enforces).

## 4. Best Practices Discovered
- **Use multiple-choice over free-form when the decision space is enumerable.** The structured choice UX is what makes `clarify` useful; free-form loses that structure and is harder for the agent to parse reliably (per issue #18450).
- **Gate only genuinely blocking ambiguity.** `clarify` is the right tool when proceeding without the answer risks wrong/external-effectful work (e.g., destructive action, irreversible publish, ambiguous scope). Do NOT use it for things the agent should decide itself.
- **Never delegate a clarify-dependent task.** If a subtask needs user input, keep it in the main agent or restructure so the ambiguity is resolved before delegating.
- **Do not rely on `clarify` in cron/autonomous contexts.** There is no user present; design autonomous loops to decide without human gating.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** Alfred currently makes autonomous decisions in cron loops (e.g., the SI loop itself says "make reasonable decisions where needed"). For interactive (non-cron) sessions with Arif, Alfred does NOT currently invoke `clarify` — it reasons and proceeds, occasionally asking inline questions in prose. The `coding`/`all` toolset is in play for interactive sessions, so `clarify` is available but unused as a structured tool.
- **Gaps & Anti-Patterns:**
  - Alfred asks ambiguous-scope questions in free-form prose rather than using `clarify`'s structured multiple-choice, losing the parseable-choice benefit.
  - No rule encodes "delegate_task subagents cannot clarify" — risk of delegating a task that needs user input.
  - The execution-tier-selection knowledge file referenced by other lessons (`knowledge/execution-tier-selection.md`) does NOT exist yet, so the clarify/async-delegation exclusion is not documented in the decision framework.

## 6. Recommended Improvements
1. **Adopt `clarify` (multiple-choice) in interactive sessions** for genuinely blocking decisions — e.g., lifecycle promotions (Idea→Project), destructive portfolio changes, or ambiguous scope. Add a one-line convention to AGENTS.md: "Use `clarify` (multiple-choice) for blocking human-in-the-loop decisions in interactive sessions; never in cron."
2. **Document the subagent exclusion.** Create `knowledge/execution-tier-selection.md` (referenced but missing) and explicitly record: `delegate_task` subagents cannot use `clarify`; resolve ambiguity before delegating.
3. **Guardrail for autonomous loops.** Add to the cron-job AGENTS context (or SI-loop preamble) the rule that `clarify` must not be called in headless/scheduled runs — autonomy is required there.

## 7. Risk Assessment
- **Autonomy loss.** Overusing `clarify` in interactive sessions turns Alfred into a nag — only gate truly blocking ambiguity.
- **Headless failure mode.** If a cron accidentally calls `clarify`, it blocks with no answerer; the job would hang or fail. Mitigation: the rule in §6.3.
- **Parse risk in free-form.** Free-form answers are less structured; prefer the 4-choice mode. If free-form is needed, phrase the question to constrain the expected answer shape.
- **Toolset availability.** `clarify` is present under `coding`/`all`; for narrow interactive toolsets it must be enabled explicitly. Verify it is in the active toolset before relying on it.

## 8. Next Learning Topic
The v2 Hermes-native backlog is exhausted (all entries `[Addressed]`). Genuinely new capabilities are now discovered opportunistically. Candidate for a future opportunistic study: the **`read_terminal`** tool (listed in the tools-reference quick-counts as a third terminal tool alongside `terminal`/`process`) — its exact semantics vs `process(poll)` are not yet covered by a dedicated lesson. If confirmed Hermes-documented and uncovered, it would be the next topic.

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
