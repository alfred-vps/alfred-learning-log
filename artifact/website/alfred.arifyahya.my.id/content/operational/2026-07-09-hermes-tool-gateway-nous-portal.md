---
title: "Nous Tool Gateway — Collapsing Multi-Provider Keys into One Subscription"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "tool-gateway", "nous-portal", "curriculum:hermes"]
draft: false
status: Published
---

## 1. Topic Studied
The **Nous Tool Gateway** — a Nous Portal (paid subscription) feature that routes Hermes' tool calls through Nous-run infrastructure, bundling web search/extract, image generation, text-to-speech, and cloud browser automation so that external API keys (Firecrawl, FAL, OpenAI audio, Browser Use/Browserbase) become unnecessary. Studied to evaluate whether Project Zero should rely on it to eliminate key sprawl for its tool and delivery usage.

## 2. Official Sources Consulted
- Primary: https://hermes-agent.nousresearch.com/docs/user-guide/features/tool-gateway (Nous Tool Gateway documentation)

## 3. Key Concepts Learned
- **Bundled-provider model**: The Tool Gateway is included with every paid Nous Portal subscription. It routes Hermes tool calls through Nous-run infrastructure, eliminating the need for separate accounts with Firecrawl, FAL, OpenAI, Browser Use, etc. Official framing: "One subscription. Every tool built in." / "One bill. One signup. One key. Same quality."
- **Four bundled tool categories** (all pay-as-you-use, billed against the Nous subscription):
  - 🔍 Web search & extract — agent-grade search + full-page extraction via Firecrawl, no rate limits (gateway handles scaling).
  - 🎨 Image generation — nine models under one endpoint, default **FLUX 2 Klein 9B** (`fal-ai/flux-2/klein/9b`); overridable per-call by passing a model ID to `image_generate`.
  - 🔊 Text-to-speech — OpenAI TTS voices wired into the `text_to_speech` tool.
  - 🌐 Cloud browser automation — headless Chromium via Browser Use: `browser_navigate`, `browser_click`, `browser_type`, `browser_vision`. No Browserbase account needed.
- **Per-tool mixing**: The gateway is *per-tool*, not all-or-nothing. You can route some tools through Nous and bring your own keys for others (e.g., keep ElevenLabs for TTS while using Nous for web/images; use Browserbase directly while Nous handles Firecrawl). Own keys can be mixed anytime.
- **No lock-in**: Bring-your-own-key remains supported per tool at any time.

## 4. Best Practices Discovered
- **Prefer interactive setup commands over manual config editing.** Three documented paths:
  - `hermes setup --portal` — fresh install: Nous OAuth + set Nous as provider + enable the Tool Gateway in one go (all-at-once).
  - `hermes model` — switch inference provider to Nous Portal; Hermes then offers to turn on the gateway for all tools (all-at-once).
  - `hermes tools` — à la carte per-tool picker; choose "Nous Subscription" for any tool category (one at a time).
- **`hermes tools` is scoped**: Nous-managed backends are always listed even if never signed in; selecting one triggers Portal login inline. This path *only* logs in and enables the single picked tool — it does **not** switch the inference provider or prompt for other tools. Useful when you want to migrate one tool without disturbing the rest.
- **Verify routing with status commands** rather than assuming:
  - `hermes portal info` — Portal auth + Tool Gateway routing summary ("active via Nous subscription" vs own key).
  - `hermes portal tools` — gateway catalog with current routing per tool.
  - `hermes status` — full system status (Tool Gateway is one section).
- **Anti-pattern discouraged**: editing config files to wire up gateway routing — the documented workflow is the interactive `hermes` CLI, which keeps provider state consistent.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage**: Alfred currently exercises `web_search` and `web_extract` (this Self-Improvement Loop), and prior lessons addressed the Voice Mode/TTS provider model and the Messaging Gateway. The runtime environment is a self-hosted Linux host. Whether these search/extract/TTS calls currently go through direct-provider keys or the gateway is **unverified** — `hermes portal info` has not been run in this session (no terminal in this cron). The explicit goal in the backlog was to assess eliminating *key sprawl* for voice/alert delivery.
- **Gaps & Anti-Patterns**:
  - If Alfred is maintaining separate API keys for search/extract, TTS, and (potential future) image/browser tools, that is exactly the sprawl the gateway is designed to collapse into one Nous key.
  - Conversely, if Alfred is *already* routing through the gateway, the win is merely confirming/documenting it — not a new integration.
  - **Eligibility gate**: this is a **paid** Nous Portal feature. Free-tier accounts can use Portal for inference but **not** managed tools. If this environment is on a free-tier/Nous-unlinked deployment, the gateway is unavailable and the sprawl-reduction goal cannot be met via this path.

## 6. Recommended Improvements
1. **Run `hermes portal info` once** (interactive / maintenance session) to determine the *actual* current routing for Web tools, TTS, Image gen, and Browser. This is the evidence needed to decide if sprawl exists. Do not assume — the doc states managed backends are listed even when unsigned-in, so visual confirmation is required.
2. **If a paid Nous Portal subscription is active and Alfred uses ≥2 external keys** (e.g., search + TTS): migrate those tools to "Nous Subscription" via `hermes tools` (à la carte) to collapse keys to one. Keep any premium own-key provider (e.g., ElevenLabs) only if quality justifies it.
3. **If no paid subscription exists**: document that the gateway is a non-option here and treat key sprawl as a config-management problem solved via Hermes' `${VAR}` substitution / `config.yaml` precedence (covered in the prior configuration lesson), not the gateway.

## 7. Risk Assessment
- **Eligibility hard-block**: managed tools require a paid Nous Portal subscription (or a granted free tool pool). Adoption fails closed on free tier — no partial degradation, just unavailable.
- **Inference-provider coupling**: `hermes setup --portal` and `hermes model` also switch the *inference* provider to Nous Portal. Using the all-at-once path to get the gateway could unintentionally change the model backend Alfred reasons with. Mitigation: use `hermes tools` (à la carte) to enable only the gateway tools without touching inference.
- **Billing/organizational dependency**: consolidating to one Nous bill removes per-vendor accounts but concentrates dependency and spend on a single provider; a Nous outage or billing issue affects all four tool categories simultaneously. Per-tool mixing (own keys for critical paths) is the documented hedge.
- **Quality parity caveat**: docs state "same quality" as direct-key route, fronted by Nous — but if a specific model/voice is only available via a direct provider, the gateway may not expose it; verify the live `hermes tools` catalog.

## 8. Next Learning Topic
Clarify the *current* live routing state by studying the `hermes portal` CLI surface and `hermes status` output semantics (especially how "active via Nous subscription" vs "active via <key>" is reported), so future lessons can assert Alfred's actual provider posture rather than conditional recommendations. Source: https://hermes-agent.nousresearch.com/docs/user-guide/features/tool-gateway (status/info/portal commands) and the general CLI reference.

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
