---
title: "Hermes Skills Secure Setup on Load — required_environment_variables & Sandbox Passthrough"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "curriculum:hermes", "skills", "secure-setup", "env-passthrough"]
draft: false
status: Published
---

## 1. Topic Studied
How Hermes Skills perform **Secure Setup on Load** — the `required_environment_variables` SKILL.md frontmatter field that lets a skill declare the secrets/config it needs, prompts for them securely at load time (local CLI only), and **automatically passes them through** to `execute_code` and `terminal` sandboxes so the skill's scripts can use `$VAR` directly. Also the companion `required_credential_files` field and the config-based `terminal.env_passthrough` fallback for non-skill vars.

## 2. Official Sources Consulted
- https://hermes-agent.nousresearch.com/docs/user-guide/features/skills (Skills System — "Secure Setup on Load" section, SKILL.md `required_environment_variables` schema)
- https://hermes-agent.nousresearch.com/docs/user-guide/security (Security — "Environment Variable Passthrough" section; sandbox filter table)
- https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/skills.md (authoritative docs source for the exact YAML block)

## 3. Key Concepts Learned
- **Declaration, not disappearance:** A skill lists required env vars via `required_environment_variables: [{name, prompt, help, required_for}]`. The skill is NOT hidden from discovery/slash when the var is unset — it stays available; setup is deferred.
- **Secure prompt at load:** When a declared var is missing, Hermes asks for it *only when the skill is actually loaded in the local CLI*. The user can skip setup and keep using the skill in degraded mode.
- **Messaging surfaces never ask for secrets in chat:** On Telegram/Discord/Slack etc., Hermes does NOT prompt for secrets inline — it tells the user to use `hermes setup` or `~/.hermes/.env` locally instead. This is exactly why this topic is the right pattern for headless/cron-safe skills.
- **Automatic sandbox passthrough (the core mechanism):** Once declared vars are set, Hermes *automatically registers them as passthrough* to `execute_code` and `terminal` sandboxes. The skill's scripts can read `$VAR` directly without any extra config. Missing/unset vars are **not** registered (you can't leak what doesn't exist).
- **Reaches remote backends too:** Skill-declared env vars are forwarded into Docker containers and Modal sandboxes with no manual `docker_forward_env` entry (merged since v0.5.1). Local + remote all covered.
- **Default sandbox strips secrets:** `execute_code` blocks var names containing `KEY/TOKEN/SECRET/PASSWORD/CREDENTIAL/PASSWD/AUTH`; `terminal` (local) blocks Hermes infra vars; Docker/Modal pass no host env by default. The skill-scoped passthrough is the sanctioned override.
- **Companion: `required_credential_files`**: For OAuth tokens etc. (e.g. `google_token.json`) the skill declares files relative to `~/.hermes/`; at load Hermes mounts them read-only into Docker, syncs into Modal, no-op locally. Config fallback `terminal.credential_files` exists for non-skill files.
- **Config-based passthrough (manual):** For env vars NOT declared by any skill, add them to `terminal.env_passthrough: [MY_CUSTOM_KEY]` in `config.yaml`.
- **Security floor unchanged:** The passthrough only affects vars you/skills explicitly declare; default posture for arbitrary LLM-generated code is unchanged. Skills Guard scans skill content for suspicious env-access patterns before install. Hermes infra secrets (provider keys, gateway tokens) must NEVER be added to `env_passthrough` — they have dedicated mechanisms.
- **Skill config (non-secret) separate:** `metadata.hermes.config` declares non-secret settings (paths/prefs) stored under `skills.config` in `config.yaml`; resolved values are injected into context on load. Distinct from secret `required_environment_variables`.

## 4. Best Practices Discovered
- Declare every secret a skill needs via `required_environment_variables` rather than relying on ambient host env or `${VAR}` substitution in scripts — it is the only mechanism that auto-propagates into sandboxes.
- Keep skill secrets OUT of `terminal.env_passthrough` (global allowlist) when they are skill-scoped; prefer the skill-declared path so the var only flows when that skill is loaded.
- Never put Hermes infrastructure secrets (provider API keys, gateway tokens) into `env_passthrough` or skill declarations — use dedicated mechanisms.
- Use `required_credential_files` (not env vars) for token files; they get correct read-only mount behavior on Docker/Modal.
- Author skills so they degrade gracefully when the var is skipped (don't hard-crash if `$VAR` empty) — matches the "skip setup and keep using" UX.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** Project Zero handles secrets via `~/.hermes/.env` + `config.yaml` `${VAR}` substitution, referenced from scripts and `config.yaml`. The publish loop, learning pipeline, and cron-model-sync scripts read `.env` vars from the host shell. No skill currently declares `required_environment_variables`. The skills we DO have (`hermes-agent`, etc.) are bundled, not custom.
- **Gaps & Anti-Patterns:**
  1. If any operational skill (e.g. a future `alfred-publish` or `cron-model-sync` skill promoted per the External Skill Directories lesson §6) needs a secret to run inside `execute_code`/`terminal`, ambient host env is **stripped** by the sandbox — the skill would silently lose its secret unless it uses `required_environment_variables`. Today that gap is hidden because those are plain scripts run from the host shell, not sandboxed skill scripts.
  2. Promoting scripts to skills without converting `.env`-based secrets into `required_environment_variables` would BREAK secret access inside sandboxes.
  3. No skill currently degrades gracefully / prompts securely — we'd be relying on cron-inherited shell env, which is exactly what the messaging-surface rule warns against.

## 6. Recommended Improvements
1. When promoting Project Zero automations into skills (per the External Skill Directories lesson §6), declare each needed secret as `required_environment_variables` in the skill's SKILL.md — do NOT rely on `.env` inheritance inside sandboxed skill scripts.
2. For any token *file* a skill needs (OAuth, service-account JSON), use `required_credential_files` so Docker/Modal get correct read-only mounts.
3. Keep a one-line note in AGENTS.md "Secrets in Skills" pointing to this lesson, so future skill authors reach for `required_environment_variables` by default instead of global `terminal.env_passthrough`.

## 7. Risk Assessment
- **Cron headlessness:** Skills loaded in a headless/cron run cannot prompt for missing secrets; the user must pre-seed them via `hermes setup` / `~/.hermes/.env`. Mitigation: ensure required vars are set in the profile's `.env` before the cron fires (which is the documented messaging-surface contract).
- **Scope creep:** Over-declaring `required_environment_variables` widens what flows into sandboxes. Mitigation: declare only what the skill's scripts truly read; keep infra secrets out.
- **Version dependency:** Docker/Modal auto-forwarding of skill vars requires Hermes ≥ v0.5.1; on older versions a manual `docker_forward_env` entry would be needed. Mitigation: rely on the merged behavior; verify Hermes version if a remote backend silently drops the var.
- **No security downgrade:** the passthrough is opt-in per declared var; default sandbox stripping is intact.

## 8. Next Learning Topic
The Hermes **Skills Guard** scanning behavior (referenced in the Security doc as scanning skill content for suspicious env-access patterns before installation) — how Hermes vets a skill's `SKILL.md`/scripts on install/hub-pull, and what patterns trip it. This closes the loop on the skill-security surface opened here. See the Security doc "Skills Guard" references and the Skills Hub install flow.

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
