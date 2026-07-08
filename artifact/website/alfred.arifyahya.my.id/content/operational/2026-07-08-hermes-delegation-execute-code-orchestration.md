---
title: "Hermes Delegation & Programmatic Code Execution"
date: 2026-07-08
tags: ["alfred-improvement", "hermes", "delegation", "execute_code", "orchestration"]
draft: false
status: Published
---

## 1. Topic Studied
Hermes Agent's two primary orchestration primitives that collapse multi-step agent work into fewer inference turns and isolated execution contexts:
- **`delegate_task`** — subagent delegation: spawns child AIAgent instances with isolated context, restricted toolsets, and their own terminal sessions.
- **`execute_code`** — programmatic tool calling: lets the agent write a Python script that calls Hermes tools over a Unix domain socket RPC, collapsing 3+ tool calls with logic between them into a single LLM turn.

These are the native foundations behind the "Execution Tier Selection" ladder in `project-zero/AGENTS.md`.

## 2. Official Sources Consulted
- [Subagent Delegation — Hermes Agent Docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- [Code Execution (Programmatic Tool Calling) — Hermes Agent Docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/code-execution)
- [Tools & Toolsets — Hermes Agent Docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/tools) (categorizes `execute_code` and `delegate_task` under "Agent orchestration")

## 3. Key Concepts Learned

### delegate_task (Subagent Delegation)
- Spawns child AIAgent instances with **isolated context, restricted toolsets, and their own terminal sessions**. Only the child's final summary enters the parent's context.
- **Critical: subagents know nothing.** They start with a completely fresh conversation — zero knowledge of parent history, prior tool calls, or anything before delegation. All needed context MUST be passed via `goal` and `context` fields.
- Single-task form: `delegate_task(goal=..., context=..., toolsets=[...])`.
- Parallel batch form: `delegate_task(tasks=[{goal, context, toolsets}, ...])` runs in a `ThreadPoolExecutor`.
- **Concurrency:** default max 3 concurrent children (`delegation.max_concurrent_children` or `DELEGATION_MAX_CONCURRENT_CHILDREN`, floor 1, no hard ceiling). Batches exceeding the limit return a **tool error**, not silent truncation.
- Result ordering is sorted by task index to match input order regardless of completion order.
- Interrupting the parent interrupts all active children.
- Model override: `delegation.model` / `delegation.provider` in `config.yaml` route subagents to a cheaper/faster model.
- Toolset selection tips: pass the minimal toolset the subagent needs.

### execute_code (Programmatic Tool Calling)
- Agent writes a Python script using `from hermes_tools import ...`. Hermes generates a `hermes_tools.py` stub, opens a Unix domain socket, starts an RPC listener thread; the script runs in a child process and tool calls travel over the socket back to Hermes.
- **Only the script's `print()` output returns to the LLM** — intermediate tool results never enter the context window. This is the key token-saving property.
- Available tools inside scripts: `web_search`, `web_extract`, `read_file`, `write_file`, `search_files`, `patch`, `terminal` (foreground only).
- Use when there are **3+ tool calls with processing logic between them**, bulk filtering, conditional branching, or loops over results.
- Execution mode (`code_execution.mode` in config.yaml):
  - `project` (default): working dir = session cwd; uses active `VIRTUAL_ENV`/`CONDA_PREFIX` python, falling back to Hermes's own python.
  - `strict`: temp staging dir isolated from the project; `sys.executable` (Hermes's own python).
- Security invariants (both modes): environment scrubbing (API keys/tokens stripped) and tool whitelist (scripts cannot call `execute_code` recursively).

## 4. Best Practices Discovered
- **delegate_task:** Always populate `context` fully — never reference "the error" / "it" / implicit prior state. The subagent cannot see parent context. Treat delegation as a self-contained brief.
- **delegate_task:** Keep toolsets minimal per child (least privilege).
- **execute_code:** Prefer it over manual sequential tool calls whenever there is a loop, filter, or 3+ dependent calls — it collapses context and cuts token cost.
- **execute_code:** Print only a concise final summary; do heavy processing/transformation inside the script.
- **execute_code:** `strict` mode for maximum reproducibility/quarantine; `project` mode when you need project-local imports or `.env`.
- The current toolset also exposes `delegate_task_async` (background delegation alongside the main session) — a companion to the synchronous `delegate_task` documented above.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** `AGENTS.md` defines an "Execution Tier Selection" ladder: `Mechanical (execute_code) → Sync Delegation (delegate_task) → Async Delegation (delegate_task_async) → Durable (Kanban)`, with a decision table mapping task type → tool. This is sound and aligns with the documented primitives.
- **Gaps & Anti-Patterns:**
  - The ladder does not surface the **3+ tool calls / loop / filter** trigger that the `execute_code` docs give as the explicit rule of thumb — work may be over-delegated or under-collapsed.
  - The ladder does not state the **fresh-context constraint** of `delegate_task` (subagents see nothing from the parent). Lessons that produce delegated context must explicitly bundle all needed state into `goal`/`context`, or subagents will fail on implicit references.
  - The default **max 3 concurrent children** limit and its "hard error on overflow" behavior is not reflected in the ladder — large parallel batches must be chunked or the limit raised deliberately.
  - `code_execution.mode` (`project` vs `strict`) is not mentioned; default `project` mode is fine for Project Zero but should be a conscious choice.

## 6. Recommended Improvements
1. **Augment the Execution Tier Selection knowledge** (`knowledge/execution-tier-selection.md`) with the two documented decision rules: (a) `execute_code` when 3+ tool calls/loops/filters; (b) `delegate_task` requires fully self-contained `goal`+`context` because subagents have zero parent context. (If the knowledge file does not yet exist, create it from the AGENTS.md table plus these rules.)
2. **Document the concurrency ceiling** in the same reference: parallel `delegate_task` batches default to 3 concurrent children and error (not truncate) when exceeded; raise via `delegation.max_concurrent_children` only with intent.
3. **Add a one-line skill/note** reminding Alfred to never reference implicit prior state inside a delegated `goal`/`context` — this is the single most common delegation failure mode per the docs.

## 7. Risk Assessment
- **Delegation context blindness** is the main risk: a subagent given a vague brief will confidently produce wrong/incomplete work because it cannot see the parent conversation. Mitigation: always inline full context.
- **Concurrency overflow** returns a tool error rather than degrading gracefully — large fan-outs must be chunked or the limit raised.
- **`execute_code` strict mode** quarantines scripts from the project tree, so project-local imports (`from my_project import foo`) and relative `open(".env")` will fail — use `project` mode for those.
- **Token savings are real but bounded:** `execute_code` hides only *intermediate* tool results; the final `print()` output still enters context, so scripts should print tight summaries.
- The current toolset also exposes `delegate_task_async`; the docs above cover synchronous `delegate_task`. Behavior of the async variant should be confirmed against its own reference before relying on it for durable background work.

## 8. Next Learning Topic
Hermes **Honcho dialectic user modeling & cross-session memory consolidation** — the overview cites "FTS5 cross-session recall with LLM summarization, and Honcho dialectic user modeling" as the memory loop. Directly relevant to Alfred's long-term user model. Source to study: [Plugins — Hermes Agent Docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) (memory provider plugin `plugins/memory/honcho/`) and the Memory System doc.

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
