---
title: "delegate_task_async and the Non-Blocking Delegation Gap"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "delegation", "async", "doc-drift", "curriculum:hermes"]
draft: false
status: Published
---

## 1. Topic Studied
The semantics, shipping status, and correct native substitute for **non-blocking background agent delegation** in Hermes Agent — specifically whether `delegate_task_async` (referenced by Project Zero's `AGENTS.md` execution-tier ladder) is a real, shipped tool, and what to use today for work that must not block the parent turn.

## 2. Official Sources Consulted
- **Subagent Delegation (feature doc):** https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation — confirms `delegate_task` is synchronous; children cancelled on parent interrupt.
- **Delegation & Parallel Work (guide):** https://hermes-agent.nousresearch.com/docs/guides/delegation-patterns — explicitly states `delegate_task` is synchronous and recommends `cronjob` or `terminal(background=True, notify_on_complete=True)` for durable non-blocking work.
- **Built-in Tools Reference:** https://hermes-agent.nousresearch.com/docs/reference/tools-reference — the `delegation` toolset row lists **only** `delegate_task`. No `delegate_task_async`, `check_task`, `collect_task`, `steer_task`, `cancel_task`, or `list_tasks`.
- **Feature proposal (context only, NOT shipped):** https://github.com/NousResearch/hermes-agent/issues/5586 — `async_delegation` toolset proposal (open), linked PRs #8482 / #15655 (open/rebase, not merged).

## 3. Key Concepts Learned
- **`delegate_task` is synchronous and blocking.** The parent agent is blocked inside the tool call until all subagents finish; only the final summary returns. Interrupting the parent cancels all active children and discards their work (per delegation guide: *"`delegate_task` is synchronous: if the parent turn is interrupted, active children are cancelled and their work is discarded."*).
- **Subagent context isolation is absolute.** Children start with a completely fresh conversation — zero knowledge of the parent's history. The parent must pass *everything* (file paths, error text, project structure, constraints) via `goal`/`context`. The same isolation model is specified for any async variant.
- **`delegate_task_async` is NOT a shipped tool.** The official tools reference registers only `delegate_task`. The `async_delegation` toolset (`delegate_task_async` / `check_task` / `collect_task` / `steer_task` / `cancel_task` / `list_tasks`) exists only as an open GitHub proposal (#5586, with PRs #8482 and #15655 still open as of research date). Treat it as future/experimental, not available.
- **Official native substitutes for non-blocking work today:**
  - `cronjob` — scheduled/durable jobs; *"Cron runs happen in fresh sessions with no current-chat context."* Best for work that should outlive the current turn.
  - `terminal(background=True, notify_on_complete=True)` — fire-and-forget shell work inside the session with completion notification.
  - The delegation guide lists these two explicitly as the right choice for *"Durable long-running work that must outlive the current turn."*

## 4. Best Practices Discovered
- **Use `delegate_task` only for synchronous, reason-and-return subtasks** where a structured summary is sufficient and the parent can wait (parallel research, code review, multi-file refactor). Default max concurrency is 3 (configurable; batches over the limit return a tool error, never silently truncated).
- **Do NOT promise non-blocking delegation via `delegate_task_async`** — it is not in the shipped toolset. Route genuinely background/durable work to `cronjob` or background `terminal`.
- **Pass full context to any subagent.** "Fix the error we discussed" fails; "TypeError at api/handlers.py:47, NoneType has no .get(), project at /home/user/myproject, Python 3.11" succeeds.
- **Subagents cannot use `clarify`** — delegating tasks that need user interaction is an anti-pattern.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** `AGENTS.md` Execution Tier Selection ladder (lines 135 & 143) lists four tiers, including **"Async Delegation (`delegate_task_async`) → Non-blocking work within a session"** as a distinct, available tier between Sync Delegation and Durable (Kanban).
- **Gaps & Anti-Patterns:**
  - **Documentation drift / false capability.** The ladder presents `delegate_task_async` as a usable tool. Official docs and the tools reference show it is not shipped. Any Alfred procedure that selects this tier would fail at runtime or silently fall back.
  - The ladder's framing ("non-blocking work within a session") overlaps exactly with the official guidance to use `cronjob` / background `terminal` — so a real native path already exists; it just isn't named there.

## 6. Recommended Improvements
1. **Annotate the Async Delegation tier in `AGENTS.md`** as *proposed / not yet shipped*. Replace its description with the native non-blocking path actually available today: `cronjob` (durable, outlives turn) and `terminal(background=True, notify_on_complete=True)` (in-session fire-and-forget). Keep `delegate_task_async` only as a noted future tier once the `async_delegation` toolset lands.
2. **Add a one-line note in `knowledge/execution-tier-selection.md`** (if it exists) clarifying that `delegate_task` blocks the parent and that true non-blocking delegation requires cronjob/background-terminal until upstream ships `async_delegation`.
3. **Track the upstream feature** (#5586 / PR #15655) so the ladder can be promoted to a real tier when `delegate_task_async` ships — at which point the steering/collect/cancel primitives become usable for Alfred background orchestration.

## 7. Risk Assessment
- **If left uncorrected:** any automation or runbook that selects the "Async Delegation" tier will invoke a non-existent tool, causing runtime failure or silent degradation of intended background behavior.
- **Limits of the native substitutes:** `cronjob` runs in a fresh session (no chat context — must pass full context like a subagent); background `terminal` is shell-only (no reasoning/LLM). They cover non-blocking *execution* but not non-blocking *agent reasoning with steering*, which is exactly what `async_delegation` would add. Until then, long reasoning background jobs are best modeled as cron-backed skills.
- **Upstream uncertainty:** #5586 is open and may be superseded by a different architecture (#4949 persistent ACP subagents). The API names could change — do not hard-code the proposal's tool names into durable configs.

## 8. Next Learning Topic
**Cron job delivery, chaining, and timeout limits** — directly complementary: since non-blocking in-session delegation is not yet native, `cronjob` is Alfred's real durable-background primitive. Confirm coverage of the existing `2026-07-08-hermes-cron-delivery-chaining-timeout.md` lesson and fill any gaps (chaining semantics, max timeout, fresh-session context rules). Source: https://hermes-agent.nousresearch.com/docs/user-guide/features/tools (cronjob).

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
