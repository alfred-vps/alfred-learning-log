---
title: "Hermes session_search: Cross-Session Retrieval via FTS5"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "session-search", "memory", "context"]
draft: false
status: Published
---

## 1. Topic Studied
How `session_search` works as a Hermes-native cross-session retrieval capability (FTS5 full-text search over `~/.hermes/state.db`), and how Project Zero / Alfred should combine it with file-based knowledge to honor the AGENTS.md mandate: *"Preserve context across projects and sessions."*

## 2. Official Sources Consulted
- https://hermes-agent.nousresearch.com/docs/user-guide/sessions (primary — Sessions, FTS5, resume, handoff, management commands)
- https://hermes-agent.nousresearch.com/docs/reference/tools-reference (lists `session_search` among standalone tools)
- https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime (agent-loop handling of `todo`/`memory`/`session_search`/`delegate_task`)
- https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference (`session_search` included in the `coding` composite toolset)

## 3. Key Concepts Learned
- **Automatic session capture.** Every conversation — CLI, Telegram, Discord, Slack, cron, batch, webhook, etc. — is stored as a session with full message history in a SQLite database at `~/.hermes/state.db`. Stored fields include: session ID, source platform, user ID, human-readable title, model/config snapshot, system prompt snapshot, full message history (role, content, tool calls, results), token counts, timestamps (`started_at`/`ended_at`), and a parent session ID (for compression splits).
- **FTS5 full-text search.** The store uses SQLite FTS5, enabling full-text cross-session search. `session_search` is the tool that exposes this at agent runtime.
- **Agent-loop tool.** Per the tools-runtime doc, `session_search` (with `todo`, `memory`, `delegate_task`) is handled directly by the agent loop rather than the generic `handle_function_call` dispatch path.
- **Toolset membership.** `session_search` is bundled into the `coding` composite toolset (`file`+`terminal`+`search`+`web`+`skills`+`browser`+`todo`+`memory`+`session_search`+`clarify`+`code_execution`+`delegation`+`vision`) and is also listed as a standalone tool in the registry quick-counts.
- **Resume / lineage.** Sessions support resume by ID or title (`hermes --resume`, `hermes -c`), auto-lineage on compression (`"my project" → "#2" → "#3"`), and cross-platform `/handoff` that preserves the same session ID. CLI management: `hermes sessions list`, `hermes sessions export`, `hermes sessions prune`.
- **Media is turn-scoped.** Raw media bytes are not re-sent into future prompts; images are attached or pre-analyzed to text, audio transcribed, docs extracted. The documented #1 cause of context growth is verbose *text* (pasted transcripts, full logs, large diffs, repeated status reports), not media.

## 4. Best Practices Discovered
- Use `session_search` (or the Sessions surface) as a retrieval index for *ephemeral conversational context* — prior decisions, rationale, and in-flight discussion not yet distilled into durable files.
- Keep **durable, authoritative knowledge in files** (knowledge/, AGENTS.md, portfolio/) — do NOT treat sessions as the system of record; sessions can be pruned via `hermes sessions prune`.
- Prefer file paths, focused excerpts, and tool-backed lookups over copying large artifacts into chat (directly aligns with AGENTS.md "Reusable knowledge instead of repeating work").
- `/compress` reduces active context but is explicitly *not* a privacy delete — sessions remain searchable.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** AGENTS.md mandates "Preserve context across projects and sessions" and routes durable context to files — memories, `knowledge/`, portfolio entries, and project-scoped files. Cross-session continuity is achieved mainly through written artifacts, not through Hermes' session search.
- **Gaps & Anti-Patterns:** Nothing is broken, but we are likely **underusing `session_search` as a complementary retrieval index**. Decisions made in transient chat/cron sessions may not be distilled into `knowledge/` and thus become hard to recover. We also have no routine step that, at the start of a governed-project task, queries prior sessions for related context.

## 6. Recommended Improvements
1. **Add a retrieval habit, not a new system:** When starting work on a governed project, issue `session_search` for related prior sessions to recover decisions/rationale not yet promoted to `knowledge/`. Treat file-based knowledge as authoritative; treat session hits as leads.
2. **Document the boundary in AGENTS.md or a knowledge note:** sessions = ephemeral conversational index (prunable, not authoritative); `knowledge/` + portfolio = durable system of record. This prevents the anti-pattern of relying on `session_search` as truth.
3. **Optional hygiene:** Periodically run `hermes sessions prune` on old ended sessions to bound storage, while ensuring anything valuable was first distilled into `knowledge/`.

## 7. Risk Assessment
- **Prunability:** Sessions are not durable; `hermes sessions prune` can delete them. Relying on `session_search` as the sole source of truth risks silent loss of context. Mitigation: distill valuable findings into `knowledge/` (already the AGENTS.md rule).
- **Availability gating:** `session_search` is available when the `coding` composite toolset (or an explicit `session_search` toolset) is enabled; in very constrained toolset configs it may be absent. Verify toolset config for the running profile.
- **Privacy:** Full message history (including tool calls/results) is retained; `/compress` is not a delete. Sensitive outputs should be kept out of chat or explicitly pruned.

## 8. Next Learning Topic
`todo` tool — "Manage your task list for the current session. Use for complex tasks with 3+ steps or when the task is non-trivial" (per tools-reference `todo` toolset row). Study whether Alfred/Project Zero should adopt `todo` for in-session multi-step task tracking vs. the Kanban system, and the agent-loop handling semantics. Source candidate: https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference + https://hermes-agent.nousresearch.com/docs/reference/tools-reference.

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
