---
title: "Hermes Honcho Dialectic User Modeling"
date: 2026-07-08
tags: ["alfred-improvement", "hermes", "memory", "honcho", "user-modeling"]
draft: false
status: Published
---

## 1. Topic Studied
Hermes Agent's **Honcho memory provider** — an AI-native memory backend that adds *dialectic reasoning* and deep user modeling on top of the built-in `MEMORY.md`/`USER.md` file memory. Focus question: should Project Zero enable Honcho to deepen Alfred's cross-session model of Arif, and how does it differ from the built-in bounded memory already studied?

## 2. Official Sources Consulted
- https://hermes-agent.nousresearch.com/docs/user-guide/features/honcho (Honcho Memory — primary)
- https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins (memory-provider plugin discovery — confirms `plugins/memory/<name>/`)
- https://github.com/NousResearch/hermes-agent/blob/main/plugins/memory/honcho/README.md (provider README — mechanics, tools, config resolution)
- https://hermes-agent.nousresearch.com/docs/user-guide/features/memory (built-in memory — baseline for comparison)

## 3. Key Concepts Learned
- Honcho is a **memory provider** pluggable via the unified Memory Providers system. Switch with `hermes memory setup` (choose "honcho") or `hermes config set memory.provider honcho` + `HONCHO_API_KEY` in `~/.hermes/.env`. Requires `pip install honcho-ai` and a Honcho Cloud account or self-hosted instance.
- Instead of file-based key-value notes, Honcho maintains a **running model of the user** (preferences, communication style, goals, patterns) by reasoning about conversations *after they happen*.
- **Dialectic reasoning:** after each conversation turn (gated by `dialecticCadence`), Honcho analyzes the exchange and derives insights that accumulate over time. Multi-pass depth (1–3) with auto cold/warm prompt selection.
- **Two-layer context injection** every turn (`hybrid`/`context` mode):
  1. **Base context** — session summary, user representation, user peer card, AI self-representation, AI identity card. Refreshed every `contextCadence` turns.
  2. **Dialectic supplement** — LLM-synthesized reasoning about the user's current state/needs. Refreshed every `dialecticCadence` turns.
  - Injects into the **user message** at API-call time (not the system prompt) to preserve prefix caching; only a static mode header sits in the system prompt. Truncated to a `contextTokens` budget.
- **Capabilities beyond built-in memory:** session summary injection, multi-agent ("peer") profile isolation (separate profiles for multiple Hermes instances talking to the same user), observation modes (unified or directional), server-side derived **conclusions**, and **semantic search** over conclusions.
- **Five bidirectional tools** (`honcho_profile`, `honcho_search`, `honcho_context`, `honcho_reasoning`, `honcho_conclude`) — visibility depends on `recallMode` (hidden in `context` mode, present in `tools`/`hybrid`).
- **Three orthogonal config knobs:** `contextCadence` (turns between base refresh, default 1), `dialecticCadence` (turns between dialectic refresh, recommended 1–5), `dialecticDepth` (1–3 passes). Plus query-adaptive reasoning level (+1 at ≥120 chars, +2 at ≥400, clamped at `reasoningLevelCap`).
- **Session-start prewarm** fires dialectic in background at full depth before turn 1; falls back to a bounded synchronous call if not landed.
- **Input sanitization:** `run_conversation` strips leaked `<memory-context>` blocks from user input (needed because `saveMessages` can persist injected context into history).

## 4. Best Practices Discovered
- Treat Honcho as a **deeper, automatic** layer *on top of* built-in memory — not a replacement. Built-in `MEMORY.md`/`USER.md` remain the curated, offline, bounded agent recall.
- Tune the three cadence knobs to budget: frequent base context + infrequent/deep dialectic is a valid mix. Default `dialecticCadence: 2` is the documented starting point.
- Use **peer isolation** when multiple agents serve the same user, to prevent cross-contamination of user models.
- Prefer `hybrid` or `context` recall mode for automatic injection; `tools` mode exposes the five tools for explicit on-demand calls.
- Honcho config resolves from first existing file: `$HERMES_HOME/honcho.json` (profile-local) → `~/.hermes/honcho.json` → (global). This respects the profile-scoped memories already studied.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:**
  - Built-in bounded memory (`MEMORY.md`/`USER.md`) is the only cross-session user model, curated by Alfred. A 2026-07-08 lesson already covers its caps/consolidation rules.
  - Long-term user knowledge of Arif also lives in file-based `SOUL.md` (durable identity, never deleted) and `AGENTS.md`/`PROJECT.md` conventions — plus `knowledge/` and `AGENT-REVIEW.md`.
  - No automatic dialectic reasoning; the user model is *only* what Alfred manually saves and what is encoded in static docs.
- **Gaps & Anti-Patterns:**
  - **Manual-only modeling:** preferences/working style of Arif must be explicitly saved to memory; nothing is inferred from conversation patterns automatically.
  - **No cross-session continuity context:** each session starts from the frozen snapshot; there is no session-summary injection or "what matters right now" layer easing repetition.
  - **No semantic recall:** memory search is FTS5 over sessions only; no derived-conclusion semantic search.

## 6. Recommended Improvements
1. **Decide via a bounded experiment, not by default.** Enable Honcho in a *scratch/test profile* (not the default that runs production cron loops) using `hermes memory setup honcho` + `HONCHO_API_KEY`, and run a few real sessions to compare injected context quality against the manual `USER.md` model. (Honcho requires an external API key/account — a dependency and cost to weigh.)
2. **Preserve the boundary:** keep `SOUL.md`/`AGENTS.md`/`PROJECT.md` as the immutable source of truth and `MEMORY.md`/`USER.md` as curated recall; let Honcho *augment* the live working context, not replace the governed documents. The two systems are documented as complementary.
3. **If adopted, set conservative cadence** (`contextCadence: 1, dialecticCadence: 2, dialecticDepth: 1`) initially and monitor token budget via `contextTokens`; escalate depth only if observed value warrants it (avoid premature optimization).

## 7. Risk Assessment
- **External dependency:** Honcho needs `honcho-ai` install, an API key (Honcho Cloud) or a self-hosted instance — adds a network/credential dependency and potential cost to Alfred's memory layer. Built-in memory has neither.
- **Prompt-cache interaction:** Honcho injects into the *user* message to preserve system-prompt caching; misconfiguration could still shift cache behavior. Sanitization handles leaked injection blocks, but the provider is more complex than the static file approach.
- **Budget/overhead:** dialectic LLM calls + context injection consume tokens every turn; an improperly set `contextTokens`/`dialecticCadence` can inflate per-turn cost.
- **Failure mode:** if Honcho is unavailable (API down), memory injection degrades — built-in `MEMORY.md`/`USER.md` should remain the resilient fallback. Do not let adoption weaken the file-based recall already in place.

## 8. Next Learning Topic
How do credential pools and provider fallbacks work (multiple API keys, `fallback_providers`, rotation), and can Project Zero harden against the `HTTP 502` provider errors that broke the publish job? Study `credential_pool_strategies` and drift-guard behavior. Source: https://hermes-agent.nousresearch.com/docs/user-guide/configuration

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
