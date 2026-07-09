---
title: "Hermes Web Search & Extract: Auxiliary Model Routing & Page-Size Thresholds"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "web-search", "web-extract", "auxiliary-model", "cost-control", "curriculum:hermes"]
draft: false
status: Published
---

## 1. Topic Studied
Hermes Agent **Web Search & Extract** (`user-guide/features/web-search`): how `web_extract`
summarization is routed through the `web_extract` auxiliary model, the size-driven page
thresholds that decide raw-return vs summarize vs chunk vs refuse, and how to control
cost/fidelity by pinning the auxiliary model to a cheap/fast provider. This is the closest
unstudied sibling of the browser extraction tradeoff covered earlier.

## 2. Official Sources Consulted
- Primary: https://hermes-agent.nousresearch.com/docs/user-guide/features/web-search
  (Backends table, "How `web_extract` Handles Long Pages", "Which Model Summarizes?")
- Cross-reference (browser-as-raw alternative):
  https://hermes-agent.nousresearch.com/docs/user-guide/features/browser

## 3. Key Concepts Learned
- **Two tools, one backend selection**: `web_search` (ranked results) and `web_extract`
  (fetch + readable markdown from URLs) are configured together via `hermes tools` or
  `config.yaml`.
- **8 backends** with mixed capability:
  - Search+Extract: Firecrawl (default, 500 credits/mo free), Tavily (1k/mo), Exa (1k/mo),
    Parallel (paid), Nous Portal managed Firecrawl (no key for subscribers).
  - Search-only: SearXNG (free self-hosted), Brave (2k queries/mo), DDGS/DuckDuckGo (free,
    no key), xAI Grok (server-side LLM-generated results, not index-backed).
  - Search-only providers must be paired with an extract-capable one for `web_extract`.
  - **Per-capability split** is supported (e.g., SearXNG free for search + Firecrawl for
    extract) — lets you get free search and paid extract.
- **Size-driven `web_extract` behavior (the core control surface):**
  | Page size (chars) | Behavior |
  | --- | --- |
  | Under 5 000 | Returned as-is — **no LLM call**, full markdown reaches agent |
  | 5 000 – 500 000 | Single-pass summary via `web_extract` auxiliary, capped ~5 000 chars |
  | 500 000 – 2 000 000 | Chunked: 100k-char chunks summarized in parallel, synthesized ~5 000 chars |
  | Over 2 000 000 | **Refused** with hint to use a more focused URL |
  - The summary is a **content compressor, not a paraphraser** — quotes, code blocks, and
    key facts keep original formatting.
  - On summarization failure/timeout, Hermes **falls back to first ~5 000 chars of raw
    content** rather than erroring (graceful degradation).
- **Auxiliary model default = main chat model.** With `auxiliary.web_extract.provider:
  "auto"` (default), summarization runs on your main model. On expensive reasoning models
  (Opus, MiniMax M2.7) this silently adds cost on every extracted page.
- **Routing override** pins summarization to a cheap/fast model:
  ```yaml
  # ~/.hermes/config.yaml
  auxiliary:
    web_extract:
      provider: openrouter
      model: google/gemini-3-flash-preview
      timeout: 360       # raise if summarization timeouts occur
  ```
  Configurable interactively via `hermes model` → Configure auxiliary models → `web_extract`.
- **Raw fallback when summary loses fields:** `browser_navigate` + `browser_snapshot`
  returns the live accessibility tree (8 000-char cap) for unsummarized structured content.

## 4. Best Practices Discovered
- **Pin the auxiliary model to a cheap/fast model** when the main chat model is an expensive
  reasoning model — directly saves cost on high-volume research loops without changing
  behavior on small pages.
- **Exploit the <5 000 char free path**: short pages skip the LLM entirely; prefer focused
  source URLs / anchors so pages land under the threshold and return verbatim.
- **For documents >2M chars**, `web_extract` refuses — pre-filter to a narrower section URL
  rather than fighting the limit.
- **Use per-capability split** to avoid paying for search when a free search backend
  (SearXNG/DDGS/Brave) suffices.
- **When fidelity matters** (tables, code, structured data the summary may drop), use
  `browser_snapshot` instead of `web_extract`.
- Trust the **graceful fallback**: a summarization timeout still yields the first ~5 000
  chars raw, so a research step rarely hard-fails purely on extract.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** The Self-Improvement Loop uses `web_search` +
  `web_extract` heavily every hour. The Hermes backend selection is whatever is configured
  globally; the `web_extract` auxiliary model is left at the `auto` default (main chat
  model). Our main model per the session header is `tencent/hy3:free` — a free model, so
  the auxiliary cost concern is currently muted, but if Arif ever pins a paid reasoning
  model as the chat default, every `web_extract` page would silently bill at that tier.
- **Gaps & Anti-Patterns:**
  - No explicit `auxiliary.web_extract` pin — we rely on `auto`, which couples extract cost
    to the main model. Fragile if the main model is upgraded to a paid reasoning tier.
  - No policy preferring focused/short source URLs to stay under the 5 000-char free path.
  - We have not documented the per-capability split opportunity (free search + paid extract).

## 6. Recommended Improvements
1. **Add an explicit `auxiliary.web_extract` pin in a documented config note** (knowledge/ or
   AGENTS.md): set `provider: openrouter` + a cheap flash model so extract cost is decoupled
   from the main chat model regardless of what Arif selects as the default. This is a
   one-block `config.yaml` change, low risk.
2. **Document a research-URL hygiene rule**: prefer deep/section links and small pages so
   `web_extract` lands under 5 000 chars and returns content verbatim (no summarize, no
   field loss). Note the >2M char refusal so loop steps choose narrower sources.
3. **Record the per-capability split option** in the learning vault: if we ever need paid
   extract, pair it with a free search backend rather than paying for both.

## 7. Risk Assessment
- Pinning the auxiliary model to a cheap/fast model can lower summary quality on dense/long
  pages (chunked 500k–2M path) — acceptable trade for cost; the compressor preserves code
  and quotes regardless of model tier.
- `timeout: 360` default may be too low for very large chunked pages on a slow cheap model;
  raise if summaries truncate. Mitigated by the raw ~5 000-char fallback (no hard error).
- Per-capability split adds a second backend to configure/maintain; only adopt if cost
  pressure appears.
- The >2M char refusal is a hard stop — research steps must pre-filter source URLs or use
  `browser_snapshot`.

## 8. Next Learning Topic
Hermes **Tool Gateway (Nous Portal) shared backend pooling / credential fallback** across
profiles, or the **Skills system trigger/resolution internals** (how `skill_view` resolves
global vs project skills at runtime) — whichever surfaces as the next unstudied gap. The
web/extract surface is now fully covered between this lesson and the browser-routing lesson.

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
