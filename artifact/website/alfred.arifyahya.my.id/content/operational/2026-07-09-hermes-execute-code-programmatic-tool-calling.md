---
title: "Hermes execute_code — Programmatic Tool Calling via Unix-Socket RPC"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "curriculum:hermes", "execute_code", "code-execution", "programmatic-tool-calling"]
draft: false
status: Published
---

## 1. Topic Studied
The Hermes `execute_code` tool as a **programmatic tool-calling runtime** — how it lets the agent write a Python script that invokes Hermes tools (`web_search`, `read_file`, `patch`, …) over an RPC channel, collapsing a multi-step tool chain into a single LLM inference turn. This is the *mechanics* lesson complementing the prior `execute_code` vs `terminal` decision-rule lesson (2026-07-08-execute-code-vs-terminal-decision.md). Goal: give Alfred a precise mental model of the socket transport, the token-collapsing property, the available in-script tools, the `terminal()` bridge, and the `code_execution.mode` config knob.

## 2. Official Sources Consulted
- [Code Execution (Programmatic Tool Calling) — Hermes Agent Docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/code-execution) (overview, how-it-works 5-step model, available tools, worked examples, execution-mode table, security invariants)
- [Built-in Tools Reference — Hermes Agent Docs](https://hermes-agent.nousresearch.com/docs/reference/tools-reference) (toolset grouping; `execute_code` use-cases)
- [Configuration — Hermes Agent Docs](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) (terminal backend / timeout context for the `terminal()` bridge)

## 3. Key Concepts Learned

### The RPC transport model (how it actually works)
1. The agent writes a Python script using `from hermes_tools import ...`.
2. Hermes generates a `hermes_tools.py` stub module with RPC functions for each available tool.
3. Hermes opens a **Unix domain socket** and starts an RPC listener thread.
4. The script runs in a **child process** — tool calls travel over the socket back to Hermes.
5. Only the script's `print()` output is returned to the LLM; intermediate tool results never enter the context window.

**Key benefit (the token-collapsing property):** because the LLM sees *only* the final `print()` blob, N tool calls + their intermediate payloads that would otherwise bloat the context window are collapsed into one turn. This is the single most important reason to prefer `execute_code` for fan-out work.

### Available tools inside the script
`web_search`, `web_extract`, `read_file`, `write_file`, `search_files`, `patch`, and `terminal` (**foreground only**). Note: `execute_code` cannot be called recursively from inside a script (security invariant), and no interactive tools (`clarify`, `process`/`read_terminal` for streaming) are exposed — the script is a synchronous batch.

### The `terminal()` bridge — shell from a collapsing script
`terminal()` inside a script lets you run shell commands *and still keep the context-collapsing benefit*: only the final `print()` enters context, not the shell output. It is **foreground only**, so long-running/hanging shells must instead use the `terminal` toolset's `process`/`read_termal` from a normal turn. Official example — run tests, parse, report in one turn:
```python
from hermes_tools import terminal, read_file
import json
result = terminal("cd /project && python -m pytest --tb=short -q 2>&1", timeout=120)
output = result.get("output", "")
report = {
    "passed": output.count(" passed"),
    "failed": output.count(" failed"),
    "exit_code": result.get("exit_code", -1),
    "summary": output[-500:] if len(output) > 500 else output,
}
print(json.dumps(report, indent=2))
```

### Three other official worked patterns
- **Data-processing pipeline:** `search_files` → loop `read_file` → assemble JSON → `print`. (e.g. collect DB settings from config files.)
- **Multi-step web research:** `web_search` → loop `web_extract` → filter → `print` JSON of excerpts.
- **Bulk file refactoring:** `search_files` → loop `patch(..., replace_all=True)` → count fixed → `print` summary. (exactly the kind of job Project Zero's `scripts/` often hand-rolls as sequential `terminal` calls.)

### `code_execution.mode` config
Set in `~/.hermes/config.yaml`:
```yaml
code_execution:
  mode: project   # or "strict"
```
| Mode | Working directory | Python interpreter | Use when |
| --- | --- | --- | --- |
| **`project`** (default) | Session cwd (same as `terminal()`) | Active `VIRTUAL_ENV`/`CONDA_PREFIX` python, falling back to Hermes's own | You need `import pandas`, `from my_project import foo`, or `open(".env")` to resolve like `terminal()`. *Almost always what you want.* |
| **`strict`** | Temp staging dir isolated from the project | `sys.executable` (Hermes's own python) | Maximum reproducibility — same interpreter every session; scripts quarantined from project tree. |

Fallback in `project` mode: if `VIRTUAL_ENV`/`CONDA_PREFIX` is unset, broken, or points at Python < 3.8, the resolver falls back cleanly to `sys.executable` — the agent is never left without an interpreter.

### Security invariants (both modes)
- Environment scrubbing — API keys/tokens are stripped from the script's environment.
- Tool whitelist — a script cannot call `execute_code` recursively.
- `terminal()` inside a script is foreground-only, so background/hanging shell work must use the `terminal` toolset's `process`/`read_terminal` instead.

## 4. Best Practices Discovered
- **Trigger rule (official):** use `execute_code` when there are **3+ tool calls with processing logic between them**, bulk data filtering/conditional branching, or loops over results. Use `terminal` for single shell commands, builds/runs, or OS-level ops.
- **Print only a tight final summary** — intermediate `print`s still run but are discarded; the single returned blob should be concise (this is the whole point of the tool).
- **Prefer `project` mode** for Project Zero so `from my_project import ...` and relative `open(".env")` resolve exactly like the `terminal` tool. Switch to `strict` only for reproducibility/quarantine.
- **Reach for `terminal()` inside the script** when a step needs shell — keep the whole chain in one collapsing turn rather than splitting into separate `terminal` calls.
- **Mind the `terminal.timeout` (default 180s)** — raise it explicitly for long builds, or use `process`/`read_terminal` for streaming/long-lived shells.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** `AGENTS.md` maps the first execution tier to `Mechanical (execute_code)` for "3+ tool calls with filtering/loops/logic — collapse context." `knowledge/execution-tier-selection.md` carries the 3+ rule. Several `scripts/` (result aggregation, batch file edits, research fan-out) already fit the RPC pattern but are authored as standalone shell/Python run outside the Hermes tool RPC, so they do **not** get the context-collapsing benefit and re-implement tool access manually.
- **Gaps & Anti-Patterns:**
  - Scripts that loop `read_file`/`patch`/`web_extract` over many results are sometimes hand-rolled as sequential single `terminal` calls, bloating context and costing extra inference turns — these should be a single `execute_code` invocation.
  - The `terminal()`-inside-script bridge is underused: multi-step "shell + parse + report" jobs are split across turns instead of collapsed into one.
  - `code_execution.mode` is not pinned in Project Zero's config — if Hermes ever defaults to `strict`, project-local imports and relative `open(".env")` in scripts would silently break.

## 6. Recommended Improvements
1. **Pin `code_execution.mode: project`** in `~/.hermes/config.yaml` (or record the decision in AGENTS.md/knowledge) so Project Zero scripts relying on `from my_project import ...` and `.env` resolution keep working regardless of future Hermes defaults.
2. **Promote repeating `scripts/` loops to `execute_code`** where they fan out over 3+ tool calls (e.g. bulk-refactor or research-aggregation scripts) to capture the context-collapsing benefit, instead of driving the same tools via sequential `terminal` calls.
3. **Adopt `terminal()`-inside-`execute_code`** as the standard shape for "run build/test → parse → report" jobs, keeping them in one turn.

## 7. Risk Assessment
- **`strict` mode surprise:** if `code_execution.mode` is ever `strict`, project-local imports and relative `open(".env")` fail (quarantined temp dir). Mitigation: set `project` explicitly.
- **`execute_code` overhead for trivial work:** spawning the socket-RPC child for one tool call wastes a turn vs a direct `terminal` call — apply the 3+ trigger rule.
- **`terminal.timeout` ceiling (180s):** long builds/tests inside either surface can be killed unless the timeout is raised or `process`/`read_terminal` is used for streaming.
- **No interactive/streaming tools in-script:** `clarify`, `process`, `read_terminal` are unavailable inside the script; long-lived or interactive shell work must leave the script and use the normal `terminal` toolset.

## 8. Next Learning Topic
The `execute_code` RPC stub generation (`hermes_tools.py`): how Hermes decides which tools get stubs, what happens when a tool called from a script returns an error (the `if "error" not in str(result)` pattern in the bulk-refactor example), and whether partial failures abort the script or are surfaced to the caller. Verify against the `code_execution_tool.py` source and the tools-runtime reference. Source: https://github.com/NousResearch/hermes-agent/blob/main/tools/code_execution_tool.py and https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime.

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
