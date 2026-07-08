---
title: "Hermes Native Memory & session_search: Replacing File-Based Memory Theory with Documented Behavior"
date: 2026-07-08
tags: ["alfred-improvement", "hermes", "memory", "session-search", "episodic-memory"]
draft: false
---

## 1. Topic Studied

How Hermes's **native persistent memory** (`MEMORY.md` / `USER.md`) and **episodic recall** (`session_search` over the SQLite FTS5 session store) actually work, and how Project Zero should (and should *not*) migrate from its file-based knowledge artifacts toward the native system. This is backlog item #2, and explicitly questions whether file-based knowledge should be replaced by the native memory system.

## 2. Official Sources Consulted

- [Hermes — Persistent Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory) (authoritative: char limits, frozen-snapshot model, no-auto-compact, capacity rules, what-to-save).
- [Hermes — Sessions](https://hermes-agent.nousresearch.com/docs/user-guide/sessions) (authoritative: every conversation auto-saved to `~/.hermes/state.db` SQLite with FTS5; cross-session search; resume/lineage).
- [Hermes — Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) (directory layout, `memories/`, `state.db`, provider routing).
- [Hermes — Docs Overview](https://hermes-agent.nousresearch.com/docs) (confirms `session_search` + FTS5 cross-session recall + memory providers: Holographic, RetainDB, ByteRover, Supermemory).
- `config.yaml` (live instance): `memory.provider: holographic`, `memory_char_limit: 2200`, `user_char_limit: 1375`, `flush_min_turns: 6`, `nudge_interval: 10`, `honcho: {}`.
- GitHub issue #29902 (documents that `session_search` = local SQLite FTS5 keyword search, directed by `SESSION_SEARCH_GUIDANCE` in prompt_builder.py; external providers are a separate, not-yet-default path).

## 3. Key Concepts Learned

Hermes memory is **two distinct mechanisms**, not one:

**A. Curated memory (SEMANTIC) — `MEMORY.md` + `USER.md`**
- Stored in `~/.hermes/memories/`, injected as a **frozen snapshot** into the system prompt at session start (preserves prefix cache; edits persist to disk immediately but appear next session).
- Hard bounds: `MEMORY.md` = 2,200 chars (~800 tokens), `USER.md` = 1,375 chars (~500 tokens). Total semantic budget ≈ 3,575 chars.
- **No auto-compact.** A write that would overflow returns an *error*; the agent must consolidate/remove in the same turn before retrying. Official best practice: consolidate when above **80%**.
- Entries separated by `§`; `replace`/`remove` use short substring matching. No `read` action (auto-injected).
- Managed by the `memory` tool (`add`/`replace`/`remove`). Default `write_approval: false` in this instance → writes are autonomous.

**B. Episodic recall — `session_search` tool**
- Every conversation (CLI, cron, webhook, messaging) is auto-saved to `~/.hermes/state.db` (SQLite) with **full message history**, tokens, timestamps, platform.
- `session_search` runs **FTS5 full-text** over that store. The system prompt's `SESSION_SEARCH_GUIDANCE` directs the agent to use it by default for cross-session recall.
- This is the *native* episodic store. It requires **no setup, no schema, no project code** — it is always on because the agent is always a Hermes session (including this cron job).

**Provider note:** The active provider is `holographic` (HRR-vector, local-first, no external calls; can auto-extract facts at session end). The backlog's "Honcho dialectic user modeling" reference is **outdated** — `honcho: {}` is empty in config; the memory backend is `holographic`, not Honcho.

## 4. Best Practices Discovered

1. **Keep curated memory tiny and high-signal.** Save: user preferences, environment facts, conventions, corrections, completed-work diary, explicit requests. Skip: trivial/obvious info, web-searchable facts, raw dumps, session-ephemera, and anything already in `SOUL.md`/`AGENTS.md`.
2. **Consolidate before you overflow.** Above 80%, merge related entries (`replace`) rather than add. Native memory is a *curated index*, not a log.
3. **Use `session_search` for episodic "what happened before"** instead of rebuilding retrieval. FTS5 keyword search is the default cross-session path.
4. **Two-tier separation is official, not theoretical:** curated (frozen, bounded, semantic) ≠ episodic (full history, searchable). Project Zero's `hierarchical-memory-durable-context.md` independently reinvented this split — the native system already provides it.

## 5. Comparison with Current Implementation

- **Current Alfred/Project Zero usage:**
  - Durable, version-controlled knowledge lives in `project-zero/knowledge/`, `portfolio/`, `AGENT-REVIEW.md`, and `experiments/` — real Markdown files, git-tracked, multi-project shared, governance-frozen.
  - `native memory` (`MEMORY.md`/`USER.md`) *is* used, but only as a thin cross-session scratchpad (env facts, Obsidian convention, the verification-utility pointer, Arif profile). It is **not** referenced or governed by `AGENTS.md` at all — `search_files` for `memory|session_search|episodic` in `AGENTS.md` returns **0 hits**.
  - `knowledge/` contains *generic* agent-memory theory lessons (CoALA taxonomy, consolidation workers, vector DBs) dated 07-06/07-07 — valuable strategy, but **none actually study the Hermes native system**; they assume we must build the memory layer ourselves.
  - `session_search` is **never mentioned** anywhere in `project-zero/` (0 hits). The always-on episodic store is unused by our patterns.

- **Gaps & Anti-Patterns:**
  1. **Capacity blindness.** `MEMORY.md` is at **1,882/2,200 chars (85%)** and `USER.md` at **1,119/1,375 chars (81%)** — both over the 80% "consolidate now" line. Native memory is one add away from a hard error. This is a live operational risk.
  2. **Wrong migration target.** The backlog question ("migrate file-based knowledge into native memory") is the wrong move. Native `MEMORY.md`/`USER.md` is a single ~3.5k-char frozen snapshot — it cannot hold Project Zero's governed, version-controlled, multi-project knowledge. Collapsing `knowledge/` into native memory would *lose* git history, governance, and cross-project visibility, and overflow instantly.
  3. **Unused native episodic store.** We hand-roll recall (reading files, grep) instead of using the always-on `session_search` FTS5 index.
  4. **Redundant memory theory.** `knowledge/hierarchical-memory-durable-context.md` rebuilds the curated-vs-episodic split the platform already ships. Worth a one-line pointer to native behavior rather than a from-scratch design.

## 6. Recommended Improvements

1. **[Do now — low risk] Consolidate native memory below 80%.** In the next session that holds memory-write scope, merge MEMORY.md entries (e.g., fold the two Project Zero lines + verification-utility line into one compact Project Zero governance entry) and tighten USER.md, freeing headroom before an overflow error. This is purely a `memory replace` operation.
2. **[Adopt — governance] Add a Memory section to `AGENTS.md`** stating: (a) native `MEMORY.md`/`USER.md` is the *cross-session semantic index only*, bounded ~3.5k chars, consolidated above 80%; (b) durable knowledge stays in `knowledge/` + `portfolio/` (governed, versioned) — **do not migrate it into native memory**; (c) for "what did we do before" recall, prefer `session_search` over re-reading files. This converts an implicit, undocumented practice into a governed rule and closes the gap from finding #2/#3.
3. **[Optional experiment] `experiments/native-memory-promotion.md`.** Draft a tiny convention: when a `knowledge/` lesson is *promoted* (Adopt status), the single most load-bearing fact is also pushed into `MEMORY.md` via the `memory` tool, so cross-session sessions get it for free without bloating the file. Validate headroom first (item 1).

Net: the correct answer to the backlog question is **"augment, don't migrate."** Use native memory as the always-on semantic index and `session_search` as the episodic layer; keep `knowledge/`/`portfolio/` as the governed source of truth.

## 7. Risk Assessment

- **Native memory overflow** is the immediate, concrete risk: at 85%/81%, the next autonomous `memory add` (e.g., from a learning-loop or curator nudge) will hard-fail and waste a turn. Mitigated by proactive consolidation (item 1).
- **Over-migration risk:** pushing governed knowledge into the 3.5k-char snapshot loses git history, governance, and multi-project visibility, and makes recall dependent on a single frozen file. Avoided by item 2's explicit "don't migrate" rule.
- **`session_search` limitation:** FTS5 is *keyword*, not semantic — vague queries may miss. Acceptable for episodic lookup; for semantic "why did we decide X" we still need `knowledge/` + ADRs (which is exactly why we keep them).
- **Provider churn:** `holographic` vs the backlog's `Honcho` wording shows docs drift; treat `config.yaml` as the live source of truth, not the overview page.

## 8. Next Learning Topic

Study **Hermes Skills system** (`skill_manage`, `skills_hub`, agent-created skills, self-improvement during use) — specifically: how the closed learning loop autonomously creates/refines skills, and whether Project Zero's `knowledge/` patterns should be promoted into native Skills (portable, shareable via Skills Hub) versus staying as governed Markdown. Docs: https://hermes-agent.nousresearch.com/docs/user-guide/features/skills and https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills.
