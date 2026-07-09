---
title: "Hermes Context Compression & Long-Running Agent Safety"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "context-compression", "durable-execution", "curriculum:hermes"]
draft: false
status: Published
---

## 1. Topic Studied
Hermes Agent's **dual-layer context compression system** — how and when it fires, the configurable parameters in `config.yaml`, what is always protected from summarization, and how this affects long-running / durable Alfred agents (Kanban tasks, multi-session cron loops) that rely on loaded governance context (AGENTS.md, PROJECT.md, portfolio entries, knowledge/). Evaluated against the backlog question: *how should long-running Project Zero agents avoid losing critical governance context near token limits?*

## 2. Official Sources Consulted
- [Context Compression and Caching | Hermes Agent](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching) — primary reference (dual compression, config keys, protect_first_n, plugin engines).
- [Configuration | Hermes Agent](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) — `config.yaml` location, `compression` settings group, `auxiliary.compression` model override.

## 3. Key Concepts Learned
Hermes runs **two independent compression layers**, not one:

1. **Gateway Session Hygiene** (safety net, pre-agent): fixed at **85%** of model context length. Uses actual API-reported tokens when available, else a rough char-based estimate. Fires only when `len(history) >= 4` and compression is enabled. Purpose: catch sessions that escaped the agent's own compressor. Deliberately set higher than the agent compressor because 50% caused premature compression every turn in long gateway sessions.
2. **Agent ContextCompressor** (primary, in-loop): fires at **`threshold` = 0.50** of context length by default, using accurate API-reported token counts inside the tool loop.

**Pluggable engine** (`ContextEngine` ABC in `agent/context_engine.py`): built-in `compressor` (lossy summarization) is default. Plugin engines (e.g. `lcm` lossless) are **never auto-activated** — you must explicitly set `context.engine: lcm`. Resolution order: `plugins/context_engine/<name>/` → general plugin registry → built-in `ContextCompressor`.

**What is always protected from summarization:**
- `protect_first_n` is **hardcoded to 3** = system prompt + first exchange are always preserved.
- `protect_last_n` (default 20) = minimum number of recent messages always kept.
- `target_ratio` (default 0.20) = fraction of threshold tokens kept as the protected tail budget.
- Summarization model is configurable under `auxiliary.compression` (model/provider/base_url); defaults to auto-detect.

## 4. Best Practices Discovered
- **Keep governance in the system prompt.** Because `protect_first_n=3` hardcodes preservation of the system prompt, AGENTS.md / PROJECT.md (injected into the system prompt) are never summarized away. This is the safe home for non-negotiable governance.
- **Don't rely on mid-session-loaded context for durability.** Portfolio entries, knowledge/ files, and project-specific references loaded *after* the first exchange sit in the compressible body and can be summarized as the agent approaches the threshold.
- **Tune `threshold`/`target_ratio` per model window.** The 0.50 default is static regardless of context-window size (documented as a known sharp edge in the setup wizard); large-window models may tolerate a higher threshold, small-window models may need a lower one to avoid runaway compression loops.
- **Front-load critical references into the first exchange** if they must survive compression, or re-inject them explicitly in long durable tasks.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** `AGENTS.md` and `PROJECT.md` are loaded as project context into the system prompt — correctly placed in the always-protected zone. The Self-Improvement Loop, publish cron, and Kanban durable tasks run as long-lived agents but depend on `knowledge/` and `portfolio/` content loaded mid-run; those are *not* guaranteed protected.
- **Gaps & Anti-Patterns:**
  - Durable Kanban tasks that accumulate governance context over many turns (e.g. multi-step research) may silently lose earlier portfolio/knowledge references once the 50% threshold is crossed, even though `protect_last_n=20` preserves only the most recent tail.
  - No explicit `compression` tuning in any profile's `config.yaml` — relying on the static 0.50 default across models with very different context windows.
  - Risk of a compression loop on small-window models: the gateway hygiene layer (85%) and agent compressor (50%) can both fire repeatedly in a long session, producing churn.

## 6. Recommended Improvements
1. **Audit profile `config.yaml` for a `compression:` block.** Add explicit, model-appropriate values (e.g. raise `threshold` for large-window models used by durable Kanban workers; lower it for small-window models) to avoid premature/runaway compression. Use `hermes config set compression.threshold 0.65` etc.
2. **Re-inject governance in long durable tasks.** For Kanban tasks expected to exceed ~40% of context, add an explicit "context refresh" step that re-reads the critical portfolio/knowledge refs (via `read_file`/`search_files`) so they re-enter the protected recent tail rather than relying on them surviving from turn 1.
3. **Keep non-negotiable governance in AGENTS.md/PROJECT.md (system prompt), not in mid-session messages.** This is already the case for Project Zero — preserve that invariant when adding new standards.

## 7. Risk Assessment
- **Plugin engines never auto-activate:** switching to a lossless engine (`lcm`) requires an explicit `context.engine` set and the plugin installed; don't assume it's on.
- **`protect_first_n` is hardcoded (3)** — you cannot protect more than the system prompt + first exchange via config; anything else depends on `protect_last_n` (recent only) or `target_ratio` (tail budget).
- **Compression is lossy by default.** Summarized body context is not recoverable; treat any fact needed later as either system-prompt-resident or re-fetched.
- **Changing `compression` settings** takes effect per session/config reload; verify in a fresh session after editing `config.yaml`.

## 8. Next Learning Topic
How does the messaging gateway multiplex 20+ platforms (Telegram, Discord, Slack, etc.) and should Project Zero broadcast Self-Improvement lessons to multiple home channels instead of Telegram-only? Study delivery routing and topic sessions. Source: https://hermes-agent.nousresearch.com/docs/user-guide/messaging

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
