---
title: "Hermes x_search (X / Twitter Search) — Real-Time Social Signal Tool"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "x-search", "tooling"]
draft: false
status: Published
---

## 1. Topic Studied
The `x_search` tool in Hermes Agent — a gated, opt-in capability that lets the agent search X (Twitter) posts, profiles, and threads. Backed by xAI's built-in `x_search` on the Responses API (`https://api.x.ai/v1/responses`); Grok runs the search server-side and returns a synthesized answer with citations to originating posts.

## 2. Official Sources Consulted
- https://hermes-agent.nousresearch.com/docs/user-guide/features/x-search (primary — X Search feature page)
- https://hermes-agent.nousresearch.com/docs/guides/xai-grok-oauth (OAuth setup, tier caveats, token reuse across xAI tools)
- https://hermes-agent.nousresearch.com/docs/reference/tools-reference (toolset registry — confirms `x_search` auto-enables with xAI creds)

## 3. Key Concepts Learned
- **What it is:** `x_search` returns a synthesized Grok answer + citations to specific X posts. Use it *instead of* `web_search` when you want current discussion/reactions/claims *on X specifically*. For general web pages, keep `web_search`/`web_extract`.
- **Gating / enablement:** Off by default. Auto-enables when an xAI credential is present. Disable via `hermes tools` → Search → x_search.
- **Two credential paths (either satisfies gating):**
  - **SuperGrok / X Premium+ OAuth** (preferred): `hermes auth add xai-oauth` → browser device-code login at `accounts.x.ai`, auto-refreshed. No `XAI_API_KEY` needed.
  - **`XAI_API_KEY`**: paid xAI API key set in `~/.hermes/.env`.
  - When both present, **SuperGrok OAuth wins** (uses subscription quota, not paid API spend).
- **Tool visibility:** `check_fn` runs the xAI credential resolver every time the model's tool list is rebuilt. A `True` return (bearer fetchable, non-empty, refreshed if expired) means the tool is exposed; revoked/failed-refresh tokens hide it from the schema.
- **Config (config.yaml):**
  ```yaml
  x_search:
    model: grok-4.20-reasoning   # recommended default
    timeout_seconds: 180         # x_search can take 60-120s; min 30
    retries: 2                   # backs off 1.5x attempt-seconds, capped 5s
  ```
- **Tool parameters:** `query` (required), `allowed_x_handles` (max 10, exclusive include), `excluded_x_handles` (max 10, mutually exclusive with allowed), `from_date`/`to_date` (`YYYY-MM-DD`), `enable_image_understanding`, `enable_video_understanding`.
- **Return JSON fields:** `answer`, `citations`, `inline_citations`, `degraded` (true when a narrowing filter was set AND both citation channels empty → answer synthesized from model's own knowledge, treat as unsourced), `degraded_reason`, `credential_source`, `model`, `query`, `provider`, `tool`, `success`.
- **Client-side date validation** before any HTTP call: both dates parse as `YYYY-MM-DD`; `from_date` ≤ `to_date`; `from_date` not later than today UTC. Failures surface as `{"error":"..."}` tool result — never an HTTP call.
- **Token reuse:** Once logged in via OAuth, the same bearer token is reused across all direct-to-xAI tools (TTS, image gen, video gen, X search) — no separate setup.
- **Tier caveat (important):** xAI's backend enforces its own allowlist and has been seen to reject standard SuperGrok subscribers with `HTTP 403` (issue #26847) even when the subscription is active. Fix: set `XAI_API_KEY` and switch to API-key path (`provider: xai`) — that surface is not subject to the same gating.

## 4. Best Practices Discovered
- Use `x_search` for *X-specific* current discussion; never as a general web-search replacement.
- Prefer SuperGrok OAuth (subscription quota) over paid `XAI_API_KEY` when both available.
- Honor the `degraded` flag — when a narrowing filter returns empty citations, the answer is unsourced; do not present it as evidence-backed.
- Set `timeout_seconds` ≥ 180 for complex queries; the tool legitimately takes 60–120s.
- For headless/remote sessions: `hermes auth add xai-oauth --no-browser` (prints URL + code, polls in background; no SSH tunnel needed).

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** Unused. The Self-Improvement Loop relies solely on `web_search` / `web_extract`. No X/Twitter signal is captured. x_search is not enabled because no xAI credential (OAuth or `XAI_API_KEY`) is configured in this environment.
- **Gaps & Anti-Patterns:** None in execution — the gap is capability availability only. We are not misusing it. The relevant open question is whether to *acquire* the credential to enable real-time X monitoring of Hermes/Nous Research/AI-agent discourse, which would broaden the evidence base for the learning loop and portfolio radar.

## 6. Recommended Improvements
1. **Decision gate (no code):** Evaluate whether Arif wants a SuperGrok/X Premium+ subscription (or a paid `XAI_API_KEY`) purely to add X-signal monitoring. Cost/benefit is the only blocker — the tool itself is native and free to wire once a credential exists.
2. **If enabled:** Add a lightweight `scripts/` or skill that runs `x_search` on a curated query set (e.g. "Hermes Agent", "Nous Research", "AI agent orchestration") and diffs new posts against the last run — feeding novel Hermes-doc hints into the backlog.
3. **Honor `degraded`:** Any X-derived lesson must record `degraded`/`credential_source` so we never publish a "Hermes docs say X" claim sourced from unsourced Grok synthesis.

## 7. Risk Assessment
- **Credential cost:** Requires a paid subscription or API key — the primary adoption barrier.
- **Tier 403:** Standard SuperGrok OAuth may be rejected by xAI's backend allowlist (issue #26847); mitigation is the `XAI_API_KEY` path, which is paid per-call.
- **Unsourced synthesis:** `degraded` answers look confident but cite nothing. Must be filtered out of any evidence-backed lesson (our loop's core honesty rule).
- **No local verification:** This cron cannot run `hermes doctor` or reachability checks; whether x_search is actually live here can only be confirmed by a terminal-bearing session, not by this loop.

## 8. Next Learning Topic
The backlog's Hermes-native queue is now exhausted. Candidate follow-ups (not yet in backlog, require doc confirmation before publishing): (a) the `session_search` tool — how cross-session memory/recall works and whether it complements persistent memory; (b) the `todo` tool — native task tracking vs our Kanban/AGENT-REVIEW flow. Both should be verified against docs before promotion.

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
