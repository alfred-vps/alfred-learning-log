---
title: "Hermes Subagent Delegation & Parallel Work Patterns"
date: 2026-07-08
tags: ["alfred-improvement", "hermes", "delegation", "subagents", "parallel-execution", "orchestrator", "async-delegation"]
draft: false
---

## 1. Topic Studied
Hermes Agent's `delegate_task` subagent delegation system, including synchronous batch delegation, the new `async_delegation` toolset (June 2026), orchestrator hierarchies, model override strategies, and toolset restriction best practices. This study addresses the backlog item: "How can we leverage Hermes `delegate_task` subagent delegation for parallel research, code review, and multi-file refactoring within Project Zero's governance framework?"

## 2. Official Sources Consulted
- [Subagent Delegation — Feature Reference](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation) — Complete `delegate_task` API, toolset restrictions, batch mode, orchestrator, model override, timeouts, monitoring
- [Delegation & Parallel Work — Patterns Guide](https://hermes-agent.nousresearch.com/docs/guides/delegation-patterns) — When to delegate, parallel research, code review, compare alternatives, multi-file refactoring
- [GitHub: delegation.md source](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/delegation.md) — Authoritative reference including permanently blocked tools and depth control
- [Async Subagents Announcement (MarkTechPost)](https://www.marktechpost.com/2026/06/16/hermes-agent-adds-asynchronous-subagents-so-delegated-work-no-longer-blocks-the-parent-chat/) — `async_delegation` toolset description, lifecycle tools, sync vs async comparison
- [Fast.io Delegation Patterns](https://fast.io/resources/hermes-agent-subagent-delegation-patterns/) — Community patterns, orchestrator depth, `/agents` TUI overlay
- [GitHub Issue #5586](https://github.com/NousResearch/hermes-agent/issues/5586) — Async delegation implementation tracking (referenced)

## 3. Key Concepts Learned

### 3.1 The Three-Tier Execution Model
Hermes provides three distinct execution tiers, each with a specific purpose:

| Tier | Tool | Durability | Concurrency | Use Case |
|------|------|-----------|-------------|----------|
| **Mechanical** | `execute_code` | Single turn | Sequential | Multi-step data transforms, programmatic pipelines |
| **Synchronous** | `delegate_task` | Single turn (blocking) | Parallel (≤3 default) | Parallel research, code review, multi-file refactoring |
| **Asynchronous** | `delegate_task_async` | Single session (non-blocking) | Parallel (≤3 default) | Long research alongside chat, background coding |
| **Durable** | Kanban | Cross-session (persistent) | Multi-agent | Long-running workflows, human-in-the-loop, cross-project |

The official docs explicitly state: "Use `delegate_task` when the subtask requires reasoning, judgment, or multi-step problem solving. Use `execute_code` when you need mechanical data transforms." This is a **complementary** toolset, not a legacy-to-replace migration.

### 3.2 Context Isolation — The Core Design Decision
Subagents start with a **completely fresh conversation**. They have zero knowledge of the parent's history, prior tool calls, or accumulated reasoning. The parent must explicitly pass everything the child needs through two parameters:

- **`goal`**: What the subagent should accomplish
- **`context`**: File paths, error messages, project structure, constraints

```python
# BAD — subagent has no idea what "the error" is
delegate_task(goal="Fix the error")

# GOOD — subagent has all context it needs
delegate_task(
    goal="Fix the TypeError in api/handlers.py",
    context="""The file api/handlers.py has a TypeError on line 47:
    'NoneType' object has no attribute 'get'.
    The project is at /home/user/myproject and uses Python 3.11."""
)
```

Only the final summary returns to the parent — intermediate tool calls and reasoning are discarded, keeping the parent's context window clean.

### 3.3 Toolset Restrictions — The Security Boundary
The `toolsets` parameter controls what tools the subagent can access:

| Pattern | Toolsets | Use Case |
|---------|----------|----------|
| Code work | `["terminal", "file"]` | Debugging, editing, builds, test runs |
| Research | `["web"]` | Web search, fact-checking, documentation lookup |
| Read-only analysis | `["file"]` | Code review without execution risk |
| System admin | `["terminal"]` | Process management, service operations |
| Full-stack (default) | `["terminal", "file", "web"]` | Complex multi-domain tasks |

**Permanently Blocked Tools** (regardless of configuration):
- `delegation` — leaf subagents cannot spawn further children (unless `role="orchestrator"`)
- `clarify` — subagents cannot ask the user questions
- `memory` — subagents cannot read/write persistent memory
- `code_execution` — Python sandbox reserved for parent agents
- `send_message` — subagents cannot use messaging gateways

### 3.4 Batch Mode — Parallel Execution
When a `tasks` array is provided, subagents run in parallel via `ThreadPoolExecutor`:
- **Default concurrency**: 3 (configurable via `delegation.max_concurrent_children`)
- **Result ordering**: Sorted by task index, not completion time
- **Interrupt propagation**: Interrupting the parent cancels all active children
- **File coordination** (v0.11.0+): Built-in layer prevents concurrent siblings from overwriting each other's file edits

### 3.5 Orchestrator Mode — Nested Delegation
Set `role="orchestrator"` to allow a child agent to spawn its own workers:

```python
delegate_task(
    goal="Survey three code review approaches and recommend one",
    role="orchestrator",
    context="...",
)
```

Controlled by `delegation.max_spawn_depth`:
- Depth 0: No delegation allowed
- Depth 1 (default): Flat delegation only (children are leaf nodes)
- Depth 2+: Children can spawn grandchildren

### 3.6 Async Delegation — New in June 2026
The `async_delegation` toolset introduces non-blocking subagents:

| Tool | Purpose |
|------|---------|
| `delegate_task_async` | Spawn background agent, returns `task_id` immediately |
| `check_task` | Non-blocking status + recent output |
| `steer_task` | Inject a message into a running task |
| `collect_task` | Block until done, return full result |
| `cancel_task` | Stop a running task |
| `list_tasks` | All async tasks in the session |

Key distinction: async subagents run **in-process** and **single-session**. If the parent session ends, children are cancelled. This is NOT a background job queue — for cross-session durability, use Kanban.

### 3.7 Model Override — Cost Optimization
Subagents can use a cheaper model than the parent:

```yaml
delegation:
  model: "google/gemini-flash-2.0"
  provider: "openrouter"
```

This is critical for cost optimization. Research tasks, code review, and refactoring often don't need the parent's expensive reasoning model. Without a model override, every subagent call burns tokens at the parent's rate.

### 3.8 Monitoring — `/agents` TUI Overlay
The TUI provides live monitoring of all subagents:
- **Live tree view** of running and finished subagents grouped by parent
- **Per-branch cost and token rollups** — see which subtask consumes the most resources
- **File-touch summaries** — which files each subagent read or modified
- **Kill/pause controls** for individual subagents mid-flight
- **Post-hoc review** of each subagent's step-by-step turn history

## 4. Best Practices Discovered

### 4.1 When to Delegate (and When NOT To)
**Good candidates for delegation:**
- Reasoning-heavy subtasks (debugging, code review, research synthesis)
- Tasks that would flood the parent's context with intermediate data
- Parallel independent workstreams (research A and B simultaneously)
- Fresh-context tasks where you want the agent to approach without bias

**Do NOT delegate for:**
- Single tool call → use the tool directly
- Mechanical multi-step work with logic between steps → `execute_code`
- Tasks needing user interaction → subagents can't use `clarify`
- Quick file edits → do them directly
- Durable long-running work that must survive parent interruption → Kanban or `terminal(background=True, notify_on_complete=True)`

### 4.2 Context Passing — The Golden Rule
> **Subagents know absolutely nothing about your conversation.** Always pass file paths, error messages, project structure, and constraints explicitly. If you delegate "fix the bug we were discussing," the subagent has no idea what bug you mean.

### 4.3 Model Selection by Task Complexity
- **Simple tasks** (file listing, grep, basic research): Use a cheap model (e.g., Gemini Flash)
- **Reasoning tasks** (debugging, code review): Use a mid-tier model
- **Complex architecture tasks**: Consider using the parent's model (orchestrator pattern)

The config currently has `delegation.model: ""` — meaning ALL subagents use the expensive parent model (`deepseek-v4-pro`). This is a significant cost optimization gap.

### 4.4 Toolset Restriction — Least Privilege
Always grant the minimum toolsets needed. For research: `["web"]`. For code changes: `["terminal", "file"]`. Never grant `["terminal"]` to a subagent that doesn't need it — this reduces the attack surface and keeps costs predictable.

### 4.5 Orchestrator Depth — Enable What You Need
If `orchestrator_enabled: true` but `max_spawn_depth: 1`, orchestrator children can't actually spawn workers. This is Alfred's current configuration — a non-functional orchestrator. To make orchestrators useful, set `max_spawn_depth: 2` or higher.

### 4.6 Child Timeout — Runaway Protection
Set `child_timeout_seconds` to a positive value (e.g., 1800 = 30 minutes) for tasks that shouldn't run indefinitely. Without a timeout (current: `0`), subagents only stop on API errors, tool errors, or hitting the iteration budget.

## 5. Comparison with Current Implementation

### Current Alfred Configuration:
```yaml
delegation:
  model: ""                          # ← NO OVERRIDE: subagents use expensive parent model
  provider: ""                       # ← NO OVERRIDE
  base_url: ""                       # ← NO OVERRIDE
  api_key: ""                        # ← NO OVERRIDE
  inherit_mcp_toolsets: true         # ✓ Correct
  max_iterations: 50                 # ✓ Reasonable default
  child_timeout_seconds: 0           # ⚠ NO TIMEOUT: runaway protection missing
  max_concurrent_children: 3         # ✓ Default
  max_async_children: 3              # ✓ Async support present
  max_spawn_depth: 1                 # ✗ ORCHESTRATOR BROKEN: children can't spawn workers
  orchestrator_enabled: true         # ✓ Enabled but non-functional (see above)
  subagent_auto_approve: false       # ✓ Correct: subagents require approval
```

### Gaps & Anti-Patterns:

1. **No delegation model override.** Every subagent uses `deepseek-v4-pro` (the parent model). For research tasks, code review, and refactoring, this burns expensive tokens unnecessarily. The official docs explicitly recommend configuring a cheaper model for delegation.

2. **Orchestrator is enabled but non-functional.** `max_spawn_depth: 1` means orchestrator children can't spawn their own workers. This is a configuration contradiction — the feature is enabled but the depth limit prevents it from working.

3. **No child timeout.** `child_timeout_seconds: 0` means no wall-clock limit. Runaway subagents only stop when they hit the iteration budget (50 turns) or encounter an API error. For autonomous cron jobs, this is a risk.

4. **Knowledge base incorrectly frames delegation as deprecated.** The `kanban-adoption-blueprint.md` states: "No In-Process Swarms: Do not use `delegate_task` for anything requiring durability, cross-agent collaboration, or human-in-the-loop review." While this is correct for the specific case of durable work, the framing implies delegation is a legacy pattern to avoid entirely. The official docs treat `delegate_task` and Kanban as **complementary tools** for different purposes. The correct framing is:
   - **`delegate_task`**: Synchronous parallel work (research, code review, refactoring)
   - **`async_delegation`**: Non-blocking background work within a session
   - **Kanban**: Durable, cross-agent, persistent workflows

5. **No documented delegation patterns.** Project Zero has no documented strategy for when to use `delegate_task` vs. `execute_code` vs. Kanban. The `knowledge/` directory covers Kanban extensively but has zero coverage of delegation patterns.

6. **No toolset restriction strategy.** The config doesn't document which toolsets to use for different delegation scenarios, and the knowledge base doesn't provide guidance.

7. **Async delegation is available but invisible.** The config has `max_async_children: 3`, indicating the async delegation feature is present, but Project Zero has no awareness of the `async_delegation` toolset or its lifecycle tools (`delegate_task_async`, `check_task`, `steer_task`, `collect_task`, `cancel_task`, `list_tasks`).

## 6. Recommended Improvements

### 1. Fix Orchestrator Depth (Immediate — Low Risk)
Set `max_spawn_depth: 2` to make orchestrator children functional. The current value of `1` means `orchestrator_enabled: true` has no effect.

```yaml
delegation:
  max_spawn_depth: 2
```

### 2. Configure Delegation Model Override (Immediate — Cost Savings)
Add a cheaper model for subagent tasks. Research, code review, and refactoring don't need the expensive reasoning model:

```yaml
delegation:
  model: "google/gemini-flash-2.0"
  provider: "openrouter"
```

**Estimated savings**: Subagent tasks (research, review, refactoring) typically consume 30-50% of total tokens in a delegation-heavy workflow. Using a model 5-10x cheaper could reduce delegation costs by 80-90%.

### 3. Add Child Timeout (Immediate — Safety)
Set a reasonable wall-clock timeout to prevent runaway subagents:

```yaml
delegation:
  child_timeout_seconds: 1800  # 30 minutes
```

### 4. Create a Delegation Strategy Knowledge Document (This Session)
Document the correct tool-choice framework in `knowledge/`:

| Task Type | Tool | Rationale |
|-----------|------|-----------|
| Single tool call | Direct tool | No overhead |
| Mechanical data transforms | `execute_code` | Programmatic, single-turn |
| Parallel research, code review, multi-file refactoring | `delegate_task` | Reasoning-heavy, parallelizable |
| Background work alongside chat | `delegate_task_async` | Non-blocking, same-session |
| Durable multi-agent workflows, human-in-the-loop | Kanban | Persistent, cross-session, auditable |

### 5. Update Kanban Adoption Blueprint (This Session)
The `kanban-adoption-blueprint.md` currently says "No In-Process Swarms: Do not use `delegate_task` for anything requiring durability, cross-agent collaboration, or human-in-the-loop review." This is correct but incomplete. Add a complementary section:

> **When `delegate_task` IS the right tool:**
> - Parallel research across multiple independent topics
> - Code review with fresh-context subagent (no parent bias)
> - Multi-file refactoring split across parallel workers
> - Comparing alternative approaches in isolation
> - Any task that benefits from parallel execution and doesn't need cross-session durability

### 6. Validation Experiment Created
An experiment at `experiments/delegation_toolset_validation.py` validates the delegation configuration against these best practices. It checks all required settings and warns about missing optimizations. Run it after any config changes.

## 7. Risk Assessment

### Model Override (Improvement #2)
- **Risk**: Low. The cheaper model may produce lower-quality results for complex tasks. Mitigation: Start with mid-tier models, not the cheapest. Monitor subagent output quality. For critical tasks, the orchestrator can override per-task (future feature, tracked in GitHub issue #10995).
- **Limitation**: Per-task model override is not yet supported (issue #10995). All subagents share the same override model.

### Orchestrator Depth (Improvement #1)
- **Risk**: Low. Deeper hierarchies increase total token consumption but also increase parallelism. The `max_spawn_depth: 2` setting is conservative — only one level of nesting beyond flat delegation.
- **Limitation**: Orchestrator subagents can spawn leaf workers but those leaf workers can't spawn further. This is the intended behavior for depth 2.

### Child Timeout (Improvement #3)
- **Risk**: Low. Tasks that legitimately need more than 30 minutes would be killed prematurely. Mitigation: The timeout is configurable per-workflow. For known long-running tasks, increase or disable the timeout.
- **Diagnostic**: If a subagent times out with zero API calls, a diagnostic log is written to `~/.hermes/logs/subagent-timeout-<session>-<timestamp>.log`.

### Delegation Strategy Document (Improvement #4)
- **Risk**: Low. Documentation changes only. The risk is that the framework is too rigid — some tasks don't fit neatly into one category. Mitigation: Frame as guidelines, not rules. The framework should be a starting point, not a straitjacket.

### Overall Risk of Using Delegation
- **Subagents are synchronous and non-durable.** If the parent turn is interrupted, ALL active children are cancelled and their work is discarded. For autonomous cron jobs, this means delegation should only be used for tasks that can complete within a single turn.
- **Subagents inherit parent credentials.** They share the same API key, provider config, and credential pool. This is by design but means a subagent can consume as many tokens as the parent.
- **File coordination is handled** (v0.11.0+), so concurrent siblings editing overlapping files won't corrupt each other's work.

## 8. Next Learning Topic
**Hermes `execute_code` — Programmatic Multi-Step Workflows** — The official docs explicitly differentiate `execute_code` from `delegate_task`: "Use `execute_code` when you need mechanical data transforms." Study the `execute_code` tool's RPC-based tool access, code sandboxing, and when to use it instead of delegation. This is the third leg of the execution tier stool that Project Zero has no documentation for.