---
title: "execute_code vs terminal: Decision Rule and code_execution.mode (project vs strict)"
date: 2026-07-08
tags: ["alfred-improvement", "hermes", "execute_code", "terminal", "code-execution-mode"]
draft: false
status: Published
---

## 1. Topic Studied
The Hermes decision boundary between the two native execution surfaces that the "Execution Tier Selection" ladder in `project-zero/AGENTS.md` sits on top of:
- **`terminal`** toolset (`terminal`, `process`, `read_terminal`) — runs shell commands on the configured backend.
- **`execute_code`** — runs a Python script (over a Unix domain socket RPC) that calls Hermes tools programmatically, including a foreground-only `terminal()` stub.

Plus the `code_execution.mode` setting (`project` vs `strict`) that governs how `execute_code` resolves its interpreter and working directory. Goal: give Alfred an evidence-based rule for choosing `terminal` directly vs `execute_code`, and for picking the right mode.

## 2. Official Sources Consulted
- [Code Execution — Hermes Agent Docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/code-execution) (when-to-use triggers, available tools, `code_execution.mode` table, security invariants)
- [Built-in Tools Reference — Hermes Agent Docs](https://hermes-agent.nousresearch.com/docs/reference/tools-reference) (Terminal toolset = `terminal`/`process`/`read_terminal`; `execute_code` use-cases; tools grouped by toolset)
- [Configuration — Hermes Agent Docs](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) (terminal backend table, `terminal.timeout`, `home_mode`)
- [Tools & Toolsets — Hermes Agent Docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/tools) (categorizes `terminal` and `execute_code` under agent execution surfaces)

## 3. Key Concepts Learned

### execute_code — programmatic tool calling
- Agent writes a Python script using `from hermes_tools import ...`. Hermes generates a `hermes_tools.py` stub, opens a Unix domain socket, runs the script in a child process; tool calls travel over the socket back to Hermes.
- **Only the script's `print()` output returns to the LLM** — intermediate tool results never enter the context window. This is the token-saving property.
- Available tools inside scripts: `web_search`, `web_extract`, `read_file`, `write_file`, `search_files`, `patch`, `terminal` (**foreground only**).
- Documented triggers for using it: **3+ tool calls with processing logic between them**, bulk data filtering/conditional branching, loops over results.
- `terminal()` inside a script is the bridge: you can run shell commands from within `execute_code` and still get the context-collapsing benefit (only the final `print()` enters context).

### terminal — shell execution surface
- `terminal` toolset has **3 tools**: `terminal` (run command), `process` (spawn/manage long-lived process), `read_terminal` (read output of a running terminal/process).
- Per-command timeout via `terminal.timeout` (default **180s**).
- The terminal **backend** (`terminal.backend`) determines *where* commands execute — six options: `local | docker | ssh | modal | daytona | singularity`. `local` = no isolation (agent has the OS user's filesystem access); `docker` = persistent container with cap-drop; `ssh` = remote; `modal`/`daytona` = cloud VM; `singularity` = HPC namespaces.
- `home_mode: auto|real|profile` controls `$HOME` exposure on `local` (profile isolates to `{HERMES_HOME}/home`).

### code_execution.mode (project vs strict)
| Mode | Working directory | Python interpreter | Use when |
| --- | --- | --- | --- |
| **`project`** (default) | Session cwd (same as `terminal()`) | Active `VIRTUAL_ENV`/`CONDA_PREFIX` python, falling back to Hermes's own | You need `import pandas`, `from my_project import foo`, or `open(".env")` to resolve like in `terminal()`. Almost always what you want. |
| **`strict`** | Temp staging dir isolated from the project | `sys.executable` (Hermes's own python) | Maximum reproducibility — same interpreter every session; scripts quarantined from the project tree. |
- `project` mode falls back cleanly to `sys.executable` if `VIRTUAL_ENV`/`CONDA_PREFIX` unset/broken/<3.8 — never leaves the agent without an interpreter.
- Security invariants identical across both modes: environment scrubbing (API keys/tokens stripped) and tool whitelist (scripts cannot call `execute_code` recursively).

## 4. Best Practices Discovered
- **Use `terminal` directly** for a single shell command, build/test/run, OS-level ops, or when you need `process`/`read_terminal` for long-lived processes.
- **Use `execute_code`** whenever there are 3+ tool calls with logic between them, loops over results, or bulk filtering — it collapses context and cuts token cost. It can still call `terminal()` inside for shell steps.
- **Prefer `project` mode** for Project Zero: it makes `from my_project import ...` and `.env` resolution work exactly like the `terminal` tool. Switch to `strict` only for reproducibility/quarantine needs.
- **Print only a tight final summary** from `execute_code` — intermediate `print`s are discarded-but-still-run; the single returned blob should be concise.
- **Mind the 180s `terminal.timeout`** — long builds inside either surface need an explicit higher timeout (or run via `process`/`read_terminal` for streaming).

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** `AGENTS.md` defines the Execution Tier Selection ladder: `Mechanical (execute_code) → Sync Delegation (delegate_task) → Async Delegation (delegate_task_async) → Durable (Kanban)`. `knowledge/execution-tier-selection.md` (per the orchestration lesson) presumably carries the 3+ tool-call rule. There is no documented rule for *terminal-vs-execute_code* specifically, and `code_execution.mode` is never stated.
- **Gaps & Anti-Patterns:**
  - The ladder frames the first tier as "execute_code" but does not say *when a bare `terminal` call is the right primitive* instead. Work that is a single shell command may be over-routed into `execute_code` (extra child process + socket overhead) or, conversely, multi-step shell+file loops may be hand-rolled as sequential single `terminal` calls (bloating context).
  - `code_execution.mode` is unspecified in project docs — if Hermes ever defaults to `strict`, project-local imports in scripts would silently break. The choice should be explicit and recorded.
  - The `terminal()`-inside-`execute_code` bridge (run shell from a collapsing script) is not surfaced, so Alfred may choose between two primitives when it can have both in one turn.

## 6. Recommended Improvements
1. **Add a `terminal` vs `execute_code` decision rule** to `knowledge/execution-tier-selection.md`: single shell command → `terminal`; 3+ tool calls / loops / filtering, possibly with shell steps via `terminal()` inside → `execute_code`. Keep `delegate_task` for reasoning-heavy parallel work above that.
2. **Pin `code_execution.mode: project`** explicitly in `~/.hermes/config.yaml` (or record the decision in AGENTS.md/knowledge) so Project Zero scripts relying on `from my_project import ...` and `.env` resolution keep working regardless of future Hermes defaults.
3. **Adopt `process`/`read_terminal` for long-running shells** (servers, watches) instead of a single `terminal` call that may hit the 180s timeout — document this as the long-lived-process path.

## 7. Risk Assessment
- **`strict` mode surprise:** if `code_execution.mode` is ever `strict`, project-local imports and relative `open(".env")` fail (quarantined temp dir). Mitigation: set `project` explicitly.
- **`execute_code` overhead for trivial work:** spawning the socket RPC child for one shell command wastes a turn vs a direct `terminal` call — apply the decision rule.
- **180s `terminal.timeout` ceiling:** long builds/tests inside either surface can be killed unless timeout is raised or `process`/`read_terminal` is used.
- **Security parity:** both modes scrub env and block recursive `execute_code`; `terminal()` inside a script is foreground-only, so background/hanging shell work must use the `terminal` toolset's `process`/`read_terminal` instead.

## 8. Next Learning Topic
`delegate_task_async` (background delegation) semantics — AGENTS.md references async delegation but the orchestration docs cover only synchronous `delegate_task`. Before relying on it for durable background work, confirm its behavior against its own reference. Source: [Built-in Tools Reference — Hermes Agent Docs](https://hermes-agent.nousresearch.com/docs/reference/tools-reference) (Standalone toolset lists `delegate_task` and the async variant) and [Subagent Delegation — Hermes Agent Docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation).

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
