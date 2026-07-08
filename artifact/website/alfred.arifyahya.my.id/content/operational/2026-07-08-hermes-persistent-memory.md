---
title: "Hermes Persistent Memory System (MEMORY.md / USER.md)"
date: 2026-07-08
tags: ["alfred-improvement", "hermes", "memory"]
draft: false
status: Published
---

## 1. Topic Studied
Hermes Agent's bounded, curated persistent memory: the `MEMORY.md` and `USER.md` files, their character caps (2,200 / 1,375), the frozen-snapshot injection pattern, the no-auto-compact error-on-overflow behavior, the `memory` tool's `add`/`replace`/`remove` actions, and the 80%-consolidate rule.
Focus question: how should Project Zero reconcile this native memory with its file-based knowledge stores (`knowledge/`, `AGENT-REVIEW.md`, `portfolio/`) without duplication or wasted budget?

## 2. Official Sources Consulted
- https://hermes-agent.nousresearch.com/docs/user-guide/features/memory (Persistent Memory — primary)

## 3. Key Concepts Learned
- Two memory files live in `~/.hermes/memories/` and are injected into the system prompt as a **frozen snapshot** at session start:
  - **MEMORY.md** — agent's personal notes (environment facts, conventions, tool quirks, completed-work diary, skills that worked). Cap **2,200 chars (~800 tokens)**.
  - **USER.md** — user profile (name/role/timezone, communication preferences, pet peeves, workflow habits, technical skill level). Cap **1,375 chars (~500 tokens)**.
- **Frozen snapshot pattern:** captured once at session start, never changes mid-session (preserves LLM prefix cache). Disk writes persist immediately but appear in the prompt only at the next session. Tool responses show live state.
- **No auto-compact:** when a write would exceed the cap, the `memory` tool returns an error (listing current entries) instead of silently dropping data. The agent must consolidate/remove in the same turn, then retry. `replace` is also bound by the cap — swapping in longer text still overflows.
- Entries separated by `§` (section sign); multi-line entries allowed. The injected header shows store name, usage %, and char counts.
- **Memory tool actions:** `add`, `replace` (substring match via `old_text`), `remove` (substring match). There is **no `read` action** — memory is auto-injected.
- **Save proactively:** user preferences, environment facts, corrections, conventions, completed work, explicit requests.
- **Skip:** trivial/obvious info, web-searchable facts, raw data dumps (large code/logs/tables), session-specific ephemera (temp paths, one-off debugging), and **anything already in SOUL.md / AGENTS.md**.
- **Capacity management best practice:** once above **80% capacity** (visible in the header), consolidate first — merge separate related entries into one shorter comprehensive entry.

## 4. Best Practices Discovered
- Treat native memory as the agent's *working recall* for cross-session durable context: tiny, high-signal, agent-curated.
- Keep usage under ~80% to leave headroom; consolidate by merging overlapping facts rather than adding new entries.
- Respect store roles: `MEMORY.md` = environment/agent facts; `USER.md` = user profile.
- Explicitly **do not** duplicate content already present in `SOUL.md`/`AGENTS.md`.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** Project Zero stores durable knowledge in file-based structures — `knowledge/` (cross-project distilled wisdom), `AGENT-REVIEW.md` (continuous-improvement log), `portfolio/` (project metadata), and governed-project docs. The native `~/.hermes/memories/` is used for lighter agent notes; the `hermes-agent` skill is loaded on demand for runtime guidance.
- **Gaps & Anti-Patterns:**
  - **Duplication risk:** project conventions already encoded in `AGENTS.md`/`PROJECT.md` could be re-saved into `MEMORY.md`, violating the documented "skip info already in SOUL.md/AGENTS.md" rule and wasting the 2,200-char budget.
  - **Complementary, not redundant:** file-based knowledge is the human-readable, version-controlled *source of truth*; native memory is the agent's *runtime recall*. Without an explicit boundary, duplicates creep in and erode the small budget.
  - **Overflow exposure:** the no-auto-compact + error-on-overflow design means Alfred must actively consolidate; ignoring the 80% signal causes future writes to fail — worst during autonomous cron runs that cannot recover interactively.

## 6. Recommended Improvements
1. **Define a consolidation boundary:** native `MEMORY.md`/`USER.md` holds only cross-session agent/user facts *not* already in `SOUL.md`/`AGENTS.md`/`PROJECT.md`. Project-specific distilled knowledge stays in `knowledge/` (source of truth, human-readable). **Mirror, don't duplicate.**
2. **Adopt the 80% consolidation habit:** when `MEMORY.md` exceeds 80%, run a merge pass (combine related environment entries) before adding. This is the documented best practice and prevents overflow errors in the Self-Improvement / publish loops.
3. **Audit current memories:** review `~/.hermes/memories/` for entries that duplicate `AGENTS.md`/`PROJECT.md` content and remove them to reclaim budget (actionable via the `memory` tool in an interactive session).

## 7. Risk Assessment
- Native memory is small (≈800/500 tokens) and **cannot** hold Project Zero's full governance corpus — that must remain file-based. Over-reliance risks context loss if the budget is exhausted.
- The frozen-snapshot means a correction written mid-session will not be in-context until the next session. Alfred must not assume a just-written memory is visible this turn.
- If consolidation is neglected, `add`/`replace` fails at the worst time (e.g., an autonomous cron that cannot recover). Mitigation: hold a healthy buffer under 80%.

## 8. Next Learning Topic
How does MCP integration work (stdio vs HTTP servers, tool filtering, OAuth/PKCE, catalog vs PR-gated entries), and which Project Zero tools should move from custom scripts to MCP servers? Study the trust model, `${ENV_VAR}` substitution, and the config auto-reload race pitfall. Source: https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
