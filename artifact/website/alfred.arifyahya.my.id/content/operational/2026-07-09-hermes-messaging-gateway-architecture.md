---
title: "Hermes Messaging Gateway — Architecture, Home Channel, and Multi-Platform Delivery"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "messaging-gateway"]
draft: false
status: Published
---

## 1. Topic Studied
The Hermes **Messaging Gateway**: how a single background process multiplexes 20+ chat platforms (Telegram, Discord, Slack, Matrix, etc.), how it routes inbound messages into sessions, how cron output reaches the user, and what "home channel" + topic sessions mean. Specifically answering the backlog question: *should Project Zero broadcast Self-Improvement lessons to multiple home channels instead of Telegram-only?*

## 2. Official Sources Consulted
- **Primary:** https://hermes-agent.nousresearch.com/docs/user-guide/messaging — Messaging Gateway overview, platform comparison matrix, architecture, intentional-silence tokens, gateway commands.
- **Secondary:** https://hermes-agent.nousresearch.com/docs/user-guide/messaging/telegram — Telegram adapter: `/sethome` home channel, topic sessions (independent sessions per topic), `allowed_chats`, privacy mode, `MEDIA:` delivery, webhook vs polling.
- **Corroborating search:** GitHub source `website/docs/user-guide/messaging/index.md` confirms the same gateway model.

## 3. Key Concepts Learned
- **Single gateway process.** One background `hermes gateway` process connects to *all* configured platforms simultaneously, owns a **per-chat session store**, and runs the **cron scheduler ticking every 60 seconds**. Adapters are not separate processes per platform.
- **Routing path:** `platform adapter → per-chat session store → AIAgent`. Each chat (and in Telegram, each *topic*) is an independent Hermes session.
- **Home channel is singular per `/sethome`.** `/sethome` designates *this chat* as the home channel. The documented model is one home channel; cron/delivery output is directed there. Telegram additionally supports *topic sessions* — "as many topics as they want, each one a fully independent Hermes session" — but these are still rooted in the single home chat.
- **Intentional silence tokens.** If a final response is exactly `[SILENT]`, `SILENT`, `NO_REPLY`, or `NO REPLY` (case/whitespace normalized, whole response must be the token), the gateway suppresses outbound delivery but **keeps the turn in the session transcript**. Failed turns still surface as errors (not hidden by token-like text). This is exactly why this cron emits `[SILENT]`.
- **Platform capability matrix.** Not all platforms support the same features (e.g., SMS/ntfy/Raft = text-only, no images/files; Teams = no files; Telegram/Discord/Slack/Matrix = full voice+images+files+threads+streaming). Multi-channel broadcast must account for per-platform capability limits.
- **Per-platform config & control.** `allowed_chats`, `group_allowed_chats`, `require_mention`, `observe_unmentioned_group_messages`, privacy mode (Telegram), and `MEDIA:` tags for native file delivery are all supported and configured per-platform in `~/.hermes/.env` or `config.yaml`.

## 4. Best Practices Discovered
- **One gateway to rule them all.** Configure platforms via `hermes gateway setup`; run as a user service (`hermes gateway install` / `start` / `status`). Don't spin up separate bots per concern.
- **Use silence tokens deliberately.** For watchdogs/automation that may have nothing to say, emit `[SILENT]` rather than empty output — it's the documented suppression path and preserves auditability in the transcript.
- **Topic sessions for channel separation.** Within a single home chat platform, use Telegram topics to keep multiple independent Alfred concerns (ops vs. strategic) as separate sessions rather than multiple bots.
- **Respect platform limits.** Don't assume file/voice delivery works everywhere; the matrix is the contract.
- **Webhook for cloud auto-wake.** Polling (default) must stay running; webhook mode (`TELEGRAM_WEBHOOK_URL` + required `TELEGRAM_WEBHOOK_SECRET`) lets cloud deployments sleep between calls.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** The Self-Improvement Loop delivers to **Telegram only** (the configured home channel), and the publish cron relies on the documented `[SILENT]` token when there is nothing to report. This matches the native model — Hermes already routes cron output to the home channel.
- **Gaps & Anti-Patterns:**
  - We assume "Telegram-only" is a limitation. Per the docs it is a *choice of which platforms to connect* — the gateway supports Discord/Slack/Matrix/etc. simultaneously with zero extra process overhead.
  - We have **no documented decision** on whether Multi-Platform broadcast is desirable. The docs reveal there is **no native "fan-out to N home channels" primitive** — home channel is singular per `/sethome`. True multi-channel broadcast would require either (a) connecting multiple platforms and setting home on each (one home per platform instance), or (b) a custom broadcast skill. This is a governance decision, not a capability gap.

## 6. Recommended Improvements
1. **Document the delivery architecture in AGENTS.md / a knowledge note:** state that Hermes gateway = single process, 60s cron tick, per-chat session store, one home channel; and that `[SILENT]` is the canonical "nothing to report" path (already used correctly).
2. **Make the multi-channel question an explicit Arif decision, not an Alfred assumption.** Per docs, multi-platform *reception* is free (connect more platforms); multi-channel *broadcast of the same lesson* is not a native primitive and needs a deliberate design (e.g., a `broadcast` skill emitting to each connected platform's home, or accepting one home per platform).
3. **If richer delivery is wanted later:** leverage Telegram topic sessions to separate "Self-Improvement log" from "ops alerts" within the existing home chat before adding new platforms — lower maintenance than a second bot.

## 7. Risk Assessment
- **Single home channel = single point of failure.** If the Telegram home chat is unavailable/silenced, cron output is lost. Mitigation: the gateway keeps turns in transcript even when delivery is suppressed; and connecting a second platform as a fallback home reduces blast radius.
- **Platform capability gaps.** A broadcast skill must degrade gracefully on text-only platforms (SMS/ntfy) — cannot assume `MEDIA:` or markdown rendering.
- **Privacy/mention config.** Adding group channels needs correct `allowed_chats` + privacy mode or the bot will misbehave in groups.
- **No evidence of harm** in current Telegram-only setup; it is fully within documented, supported usage.

## 8. Next Learning Topic
How does the **Voice Mode / TTS provider** model work (`hermes gateway` voice replies, tool providers vs model providers, Nous Portal bundling), and could Project Zero add spoken delivery of critical Alfred alerts? Study: https://hermes-agent.nousresearch.com/docs/user-guide/features/voice (or Voice Mode section referenced from the Messaging Gateway doc).

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
