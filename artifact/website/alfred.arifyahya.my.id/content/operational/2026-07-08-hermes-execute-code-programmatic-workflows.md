---
title: "Hermes execute_code — Programmatic Multi-Step Workflows"
date: 2026-07-08
tags: ["alfred-improvement", "hermes", "execute-code", "execution-tiers", "programmatic-pipelines", "context-window-optimization"]
draft: false
---

## 1. Topic Studied
Hermes Agent's `execute_code` tool — the mechanical tier of the four-tier execution model. This is the third leg of the execution stool (mechanical `execute_code` → synchronous `delegate_task` → async `delegate_task_async` → durable Kanban) that Project Zero has no documentation for.

## 2. Official Sources Consulted
- [Code Execution (`execute_code`) — Feature Reference](https://hermes-agent.nousresearch.com/docs/user-guide/features/code-execution) — Architecture, RPC mechanism, execution modes, resource limits, security invariants, and four canonical usage patterns
- [Tools & Toolsets — Overview](https://hermes-agent.nousresearch.com/docs/user-guide/features/tools) — Toolset classification, `code_execution` toolset definition
- [Built-in Tools Reference](https://hermes-agent.nousresearch.com/docs/reference/tools-reference) — `execute_code` tool spec, parameters, toolset membership
- [Tips & Best Practices](https://hermes-agent.nousresearch.com/docs/guides/tips) — When to use skills vs. `execute_code`, context file management
- [Subagent Delegation Patterns](https://hermes-agent.nousresearch.com/docs/guides/delegation-patterns) — Cross-reference: `execute_code` vs. `delegate_task` decision boundary
- [Hermes Agent Skill](https://hermes-agent.nousresearch.com/docs/user-guide/skills/bundled/autonomous-ai-agents/autonomous-ai-agents-dynamic-workflow) — `execute_code` in dynamic workflow skill
- Source code: `tools/code_execution_tool.py` — Whitelist, resource limits, mode resolution, `_scrub_child_env`, sandbox mechanics

## 3. Key Concepts Learned

### 3.1 Architecture: Unix Domain Socket RPC
`execute_code` runs Python scripts in a **child process** that communicates with the parent Hermes process over a **Unix domain socket RPC**:

1. The agent writes a Python script using `from hermes_tools import ...`
2. Hermes generates a `hermes_tools.py` stub module with RPC function wrappers
3. Hermes opens a Unix domain socket and starts an RPC listener thread
4. The script runs in a child process — each tool call is a blocking RPC over the socket
5. **Only the script's `print()` output is returned to the LLM**; intermediate tool results never enter the context window

This is the killer feature: you can chain 20 tool calls with filtering, parsing, branching, and aggregation, and the LLM only sees the final `print()` — a single line of context. Without `execute_code`, each intermediate tool result would inflate the context window, consuming tokens and reducing the LLM's effective reasoning capacity.

### 3.2 Available Tools Inside Scripts
The whitelist is deliberately restricted:

| Tool | Purpose |
|------|---------|
| `web_search` | Web search with result parsing |
| `web_extract` | Extract page content from URLs |
| `read_file` | Read files with line numbers |
| `write_file` | Write/overwrite files |
| `search_files` | Ripgrep-backed file/content search |
| `patch` | Targeted find-and-replace edits |
| `terminal` | Shell commands (foreground only) |

**Not available** (by design):
- `execute_code` — no recursive code execution
- `delegate_task` — no subagent spawning from scripts
- MCP tools — no external tool server access
- `memory` — no persistent memory access
- `clarify` — no user interaction

### 3.3 Auto-Trigger Heuristic
The agent automatically chooses `execute_code` when:
- **3+ tool calls** with processing logic between them
- Bulk data filtering or conditional branching
- Loops over results

This is not a tool the user requests — it's an optimization the agent applies when it detects a multi-step pipeline that would benefit from collapsing into a single turn.

### 3.4 Execution Modes

| Mode | Working Directory | Python Interpreter | Use Case |
|------|-------------------|-------------------|----------|
| **`project`** (default) | Session's working directory | Active `VIRTUAL_ENV` / `CONDA_PREFIX`, fallback to `sys.executable` | Import project modules, relative paths, same env as `terminal()` |
| `strict` | Temp staging directory | `sys.executable` (Hermes's own Python) | Maximum reproducibility, quarantined from project tree |

**Fallback behavior**: If `VIRTUAL_ENV`/`CONDA_PREFIX` is unset, broken, or points to Python < 3.8, the resolver falls back to `sys.executable` — never leaves the agent without a working interpreter.

**Security invariants are identical in both modes**: environment scrubbing, tool whitelist, resource limits.

### 3.5 Resource Limits

| Resource | Limit | Behavior |
|----------|-------|----------|
| **Timeout** | 300s (5 min) | SIGTERM → 5s grace → SIGKILL |
| **Stdout** | 50 KB | Truncated with `[output truncated at 50KB]` notice |
| **Stderr** | 10 KB | Truncated, last 10 KB preserved |
| **Max tool calls** | 50 (configurable via `code_execution.max_tool_calls`) | Script killed on exceeding |

### 3.6 Security: Environment Scrubbing
The child process environment is scrubbed before execution. API keys, tokens, and credentials are stripped from `os.environ`. The `_scrub_child_env` function in `code_execution_tool.py` removes known sensitive variables. This is critical for cron jobs and automated workflows where the script might `print(os.environ)` or accidentally leak credentials.

### 3.7 Four Canonical Patterns

**Pattern 1: Data Processing Pipeline**
```python
from hermes_tools import search_files, read_file
import json

matches = search_files("database", path=".", file_glob="*.yaml", limit=20)
configs = []
for match in matches.get("matches", []):
    content = read_file(match["path"])
    configs.append({"file": match["path"], "preview": content["content"][:200]})

print(json.dumps(configs, indent=2))
```

**Pattern 2: Multi-Step Web Research**
```python
from hermes_tools import web_search, web_extract
import json

results = web_search("Rust async runtime comparison 2025", limit=5)
summaries = []
for r in results["data"]["web"]:
    page = web_extract([r["url"]])
    for p in page.get("results", []):
        if p.get("content"):
            summaries.append({
                "title": r["title"],
                "url": r["url"],
                "excerpt": p["content"][:500]
            })

print(json.dumps(summaries, indent=2))
```

**Pattern 3: Bulk File Refactoring**
```python
from hermes_tools import search_files, patch

matches = search_files("old_api_call", path="src/", file_glob="*.py")
fixed = 0
for match in matches.get("matches", []):
    result = patch(
        path=match["path"],
        old_string="old_api_call(",
        new_string="new_api_call(",
        replace_all=True
    )
    if "error" not in str(result):
        fixed += 1

print(f"Fixed {fixed} files out of {len(matches.get('matches', []))} matches")
```

**Pattern 4: Build and Test Pipeline**
```python
from hermes_tools import terminal
import json

result = terminal("cd /project && python -m pytest --tb=short -q 2>&1", timeout=120)
output = result.get("output", "")

passed = output.count(" passed")
failed = output.count(" failed")
errors = output.count(" error")

report = {
    "passed": passed,
    "failed": failed,
    "errors": errors,
    "exit_code": result.get("exit_code", -1),
    "summary": output[-500:] if len(output) > 500 else output
}

print(json.dumps(report, indent=2))
```

## 4. Best Practices Discovered

### 4.1 The Four-Tier Execution Decision Framework

| Task Type | Tool | Durability | Concurrency | Decision Criterion |
|-----------|------|-----------|-------------|-------------------|
| Single tool call | Direct tool | Single turn | N/A | No overhead needed |
| **Mechanical multi-step** | `execute_code` | Single turn | Sequential | 3+ tool calls with logic between them |
| Reasoning-heavy parallel | `delegate_task` (sync) | Single turn (blocking) | Parallel (≤3) | Requires judgment, multi-step reasoning |
| Background alongside chat | `delegate_task_async` | Single session (non-blocking) | Parallel (≤3) | Non-blocking research/coding while chatting |
| Durable cross-session | Kanban | Cross-session (persistent) | Multi-agent | Long-running, human-in-the-loop, cross-project |

### 4.2 When `execute_code` Is the Right Choice
- **Collapsing context**: You have 5+ tool calls that would each add 500-2000 tokens of intermediate output to the context window. `execute_code` collapses this to one `print()`.
- **Mechanical transforms**: Filtering, parsing, aggregating, counting, formatting — operations that don't require LLM reasoning.
- **Loops**: Iterating over search results, file lists, or API responses.
- **Conditional logic**: Branching based on tool call results ("if the file exists, read it; otherwise, search for alternatives").
- **Data pipelines**: Search → extract → filter → aggregate → format.

### 4.3 When NOT to Use `execute_code`
- **Single tool call**: Just use the tool directly. `execute_code` adds overhead.
- **Reasoning-heavy tasks**: If the task requires understanding nuance, making judgment calls, or creative problem-solving, use `delegate_task`.
- **Tasks needing user interaction**: Scripts can't use `clarify`.
- **Tasks needing subagent spawning**: Scripts can't use `delegate_task`.
- **Durable work**: Scripts die when the turn ends. For cross-session persistence, use Kanban.

### 4.4 Anti-Patterns Explicitly Discouraged
- **Using `delegate_task` for mechanical transforms**: Burning a full subagent (with context isolation, LLM turns, and token overhead) to do a grep-and-filter that `execute_code` could do in one turn is wasteful.
- **Using `execute_code` for `delegate_task` work**: If the task requires reasoning, `execute_code` can't do it — it's a mechanical pipeline, not an LLM.
- **Expecting `execute_code` to be durable**: Scripts are single-turn. If the parent turn is interrupted, the script dies.

### 4.5 Config Best Practices
```yaml
code_execution:
  mode: project          # Default. Use strict for reproducible, quarantined scripts
  max_tool_calls: 50     # Default. Increase for very large pipelines
  timeout: 300           # 5 minutes. Increase for long-running scripts
```

- `project` mode is correct for Project Zero — we want `import` access to project modules and relative paths.
- `max_tool_calls: 50` is generous (default). Most pipelines use 5-20 calls.
- `timeout: 300` (5 minutes) is sufficient for most pipelines. Adjust upward for scripts that do heavy `terminal()` work.

## 5. Comparison with Current Implementation

### Current Alfred Configuration:
```yaml
code_execution:
  mode: project          # ✓ Correct for Project Zero
  max_tool_calls: 50     # ✓ Default, sufficient
  timeout: 300           # ✓ Default, sufficient
```

The `code_execution` toolset is enabled in both CLI and Telegram platform toolsets — the feature is available and correctly configured.

### Gaps & Anti-Patterns:

1. **No documentation of the execution tier model.** Project Zero has extensive Kanban documentation (ADR-0001, ADR-0002, `kanban-adoption-blueprint.md`, `kanban-migration-guide.md`, `kanban-saga-pattern.md`, `durable-execution-pattern.md`, etc.) and a delegation patterns lesson, but zero documentation covering `execute_code`. This means Alfred has no explicit guidance on when to use the mechanical tier.

2. **The `kanban-adoption-blueprint.md` frames delegation as deprecated.** The blueprint states "No In-Process Swarms: Do not use `delegate_task` for anything requiring durability..." This is correct but incomplete. It creates the impression that `delegate_task` and `execute_code` are legacy patterns to avoid entirely. The official docs treat all four tiers as **complementary**. The correct framing:
   - **`execute_code`**: Mechanical data transforms (search → filter → aggregate)
   - **`delegate_task`**: Reasoning-heavy parallel work (code review, research synthesis)
   - **`delegate_task_async`**: Non-blocking background work within a session
   - **Kanban**: Durable, cross-session, multi-agent workflows

3. **No knowledge document covering the complete execution spectrum.** The `knowledge/` directory has 20+ documents but none covering the decision framework for which execution tier to use.

4. **No `execute_code` patterns in experiments.** The `experiments/` directory has delegation validation, deny rules validation, and environment isolation — but no `execute_code` patterns.

5. **`execute_code` is invisible to the agent's self-awareness.** The `SOUL.md` and `AGENTS.md` mention Kanban and delegation but never reference `execute_code` as an available capability. This means the agent may underutilize it.

6. **No context-window optimization strategy.** `execute_code` is the primary mechanism for keeping the context window clean during multi-step pipelines. Project Zero has no documented strategy for when to collapse tool calls into `execute_code` vs. letting them inflate the context.

## 6. Recommended Improvements

### 1. Create Execution Tier Selection Knowledge Document (Immediate)
Document the complete four-tier decision framework in `knowledge/execution-tier-selection.md`. This should be the single source of truth for when to use each tier.

### 2. Create `execute_code` Patterns Experiment (Immediate)
An experiment at `experiments/execute_code_patterns.py` demonstrating the four canonical patterns from the official docs. This validates the tool's behavior and serves as a reference for future workflows.

### 3. Update `kanban-adoption-blueprint.md` (This Session)
Add a complementary section clarifying that `execute_code` and `delegate_task` are not deprecated — they are the right tools for mechanical and reasoning-heavy single-session work respectively. Kanban is the right tool for durable cross-session work.

### 4. Add `execute_code` to AGENTS.md Decision Framework (This Session)
Add an explicit line to the classification ladder or tool-choice section of `AGENTS.md`:
> "Multi-step mechanical pipelines → `execute_code`. Reasoning-heavy parallel work → `delegate_task`. Durable workflows → Kanban."

### 5. Surface `execute_code` in Self-Improvement Cron (Ongoing)
The self-improvement cron job should be aware of `execute_code` as a first-class optimization tool. When the cron produces a multi-step pipeline, it should be willing to collapse into `execute_code`.

## 7. Risk Assessment

### Using `execute_code` in Production Cron Jobs
- **Risk**: Low. `execute_code` is a mechanical pipeline — it doesn't make judgment calls, so its behavior is deterministic and predictable.
- **Limitation**: Scripts are single-turn. If the cron job's parent turn is interrupted, the script dies. For durable work, use Kanban.
- **Mitigation**: For cron jobs that need multi-step pipelines, `execute_code` is the right tool because cron jobs are already single-turn by nature.

### Context Window Collapse
- **Risk**: Low. `execute_code` reduces context window inflation, which is the opposite of risk.
- **Caveat**: The agent must learn to recognize when a pipeline would benefit from collapsing. Without explicit guidance, the agent may underuse `execute_code`.

### Resource Limits
- **Risk**: Low. The 5-minute timeout and 50-call limit are generous for most pipelines. The 50 KB stdout limit means scripts should produce concise output.
- **Mitigation**: For pipelines that need more than 50 KB of output, break into multiple `execute_code` calls or use `write_file` to persist intermediate results.

### Security
- **Risk**: Very low. Environment scrubbing prevents credential leaks. The tool whitelist prevents recursive execution, subagent spawning, and MCP tool access.
- **Note**: The `terminal` tool IS available inside scripts, but with the same command approval policy as the parent. If the parent requires approval for terminal commands, `execute_code` scripts will also require approval.

## 8. Next Learning Topic
**Hermes Episodic Memory & Session Search** — The `memory` and `session_search` toolsets. Project Zero's `AGENTS.md` says "Preserve context across projects and sessions" and "Build reusable knowledge instead of repeating work," but the current implementation relies on file-based knowledge artifacts (`knowledge/` directory). Study the official memory system (Honcho dialectic user modeling, FTS5 cross-session recall, LLM summarization, memory capacity management) and the `session_search` tool. Identify gaps between the documented memory capabilities and Project Zero's current file-based approach.