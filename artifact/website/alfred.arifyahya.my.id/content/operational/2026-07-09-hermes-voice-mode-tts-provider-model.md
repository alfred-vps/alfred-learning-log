---
title: "Hermes Voice Mode & TTS Provider Model"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "voice", "tts"]
draft: false
status: Published
---

## 1. Topic Studied
Hermes Agent's Voice Mode and the underlying Text-to-Speech (TTS) / Speech-to-Text (STT) provider model: how voice interaction works across CLI and messaging platforms, which providers are available, what API keys (if any) each requires, how the Nous Portal bundles voice tools, and what this means for adding a spoken delivery channel to Project Zero's Alfred alerts.

## 2. Official Sources Consulted
- Voice Mode overview & prerequisites: https://hermes-agent.nousresearch.com/docs/user-guide/features/voice-mode
- Voice & TTS providers, config, platform delivery: https://hermes-agent.nousresearch.com/docs/user-guide/features/tts
- Messaging Gateway (voice delivery per platform, Nous Portal bundling): https://hermes-agent.nousresearch.com/docs/user-guide/messaging

## 3. Key Concepts Learned
- **Three voice interaction modes** exist:
  - *Interactive Voice (CLI)* — press the record key (default Ctrl+B) to capture mic audio; a two-stage silence detector (RMS threshold + 3.0s continuous silence) auto-stops recording; transcription is sent to the agent; if TTS is on, the reply is spoken aloud. Recording auto-restarts for continuous conversation.
  - *Auto Voice Reply (Telegram, Discord)* — the agent sends spoken audio **alongside** its text response.
  - *Voice Channel (Discord)* — the bot joins a voice channel, listens to speakers, and speaks replies back.
- **STT providers**: local `faster-whisper` (zero API key, ~150 MB model auto-downloads), Groq Whisper (free tier, `GROQ_API_KEY`), OpenAI Whisper (`VOICE_TOOLS_OPENAI_KEY`). Local STT means voice input works with no cloud cost.
- **TTS providers (10 total)**: Edge TTS (default, free, no key, 322 voices / 74 languages), ElevenLabs (paid, `ELEVENLABS_API_KEY`), OpenAI TTS (`VOICE_TOOLS_OPENAI_KEY`), MiniMax, Mistral Voxtral, Google Gemini (free tier), xAI, plus fully-local free options NeuTTS, KittenTTS, and Piper. So TTS can also run with **zero API key** via Edge TTS or local engines.
- **Nous Portal bundling**: a paid Nous Portal subscription supplies the LLM **and** OpenAI TTS via the Tool Gateway with no separate OpenAI key. `hermes setup --portal` wires both up on fresh install; existing installs pick "Nous Subscription" for TTS via `hermes model` / `hermes tools`.
- **Configuration** lives in `~/.hermes/config.yaml` under a structured `tts:` block with a global `provider` + `speed`, and per-provider overrides (voice, model, voice_id, speed, pitch, etc.). Global `tts.speed` defaults to 1.0; provider-specific `speed` overrides it.
- **Platform delivery formats**: Telegram → inline voice bubble (Opus `.ogg`); Discord → voice bubble (Opus/OGG) with file-attachment fallback; WhatsApp → MP3 attachment; CLI → saved under `~/.hermes/audio_cache/` as MP3.

## 4. Best Practices Discovered
- Verify base text setup works (`hermes`) **before** enabling voice — voice is an add-on layer, not a replacement.
- Prefer keyless providers (Edge TTS, local NeuTTS/KittenTTS/Piper) for zero-cost, privacy-preserving spoken output.
- Keep STT local (`faster-whisper`) when no cloud dependency is desired.
- One config surface (`~/.hermes/config.yaml`) governs all TTS behavior; avoid ad-hoc per-script TTS calls when a single provider choice suffices.
- Use `hermes setup --portal` for a one-command LLM+TTS bundle rather than wiring separate keys.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** Self-Improvement lessons and cron outputs are delivered as **text only** (Telegram text messages, and the vault→site mirror). There is no spoken delivery anywhere. Voice/TTS is entirely unused.
- **Gaps & Anti-Patterns:** None violate best practice (we simply don't use voice). The gap is *opportunity*, not *violation*: critical alerts (publish failures, cron hard-interrupts, provider 502s) could add an audible channel with **zero new API keys** via Edge TTS on Telegram/Discord. We are also missing awareness that voice replies are *additive* to text — adopting them does not break the existing text delivery path.

## 6. Recommended Improvements
1. **Pilot a spoken-critical-alert channel**: enable `tts.provider: edge` in `~/.hermes/config.yaml` and route a *subset* of high-severity alerts (publish-job failures, cron timeout) to the Telegram home channel so they arrive as inline voice bubbles. Edge TTS needs no key — zero marginal cost. (Experiment candidate under `experiments/`.)
2. **Document the voice delivery contract** in AGENTS.md / a knowledge note: spoken replies are additive to text; do not assume voice replaces the text record that the Auto-Publish cron mirrors.
3. **Hold for Nous Portal evaluation**: if Arif later subscribes, `hermes setup --portal` gives OpenAI TTS quality via Tool Gateway without key sprawl — revisit provider choice then.

## 7. Risk Assessment
- **Extra surface area**: voice adds system dependencies (PortAudio, ffmpeg, opus) only if CLI/Discord-VC voice is used; *Telegram auto voice reply via Edge TTS needs none of these* — it is pure TTS generation, so the alert-channel pilot carries minimal infra risk.
- **Latency**: TTS generation adds milliseconds–seconds per alert; acceptable for low-frequency critical alerts, not for high-volume logs.
- **No guaranteed delivery**: voice bubble is a delivery format, not a guarantee; text record in vault remains the source of truth.
- **Local provider limits**: Edge TTS requires network egress to Microsoft endpoints; fully-offline needs local NeuTTS/KittenTTS/Piper instead.

## 8. Next Learning Topic
Study the **Tool Gateway (Nous Portal)** bundled web/search/image/tts/browser capability — it determines when external API keys are unnecessary and is the mechanism behind the voice-bundling described above. Source: https://hermes-agent.nousresearch.com/docs/user-guide/features/tool-gateway

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
