---
title: "Hermes Configuration Precedence & Secret Management"
date: 2026-07-08
tags: ["alfred-improvement", "hermes", "configuration", "secrets"]
draft: false
status: Published
---

## 1. Topic Studied
How Hermes Agent resolves configuration precedence and secret management: the `config.yaml` vs `.env` split, `${VAR}` environment-variable substitution, and CLI-arguments-wins ordering. The goal is to verify whether Project Zero's current secret handling aligns with the documented rule of thumb and to capture the native mechanism so we stop hand-rolling secret plumbing.

## 2. Official Sources Consulted
- [Configuration | Hermes Agent](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) — precedence ladder, rule of thumb, `${VAR}` substitution, `hermes config set` auto-routing, provider timeouts.

## 3. Key Concepts Learned
- **Single source of truth location:** all settings live in `~/.hermes/`. Two files matter here: `config.yaml` (model, terminal, TTS, compression, non-secret settings) and `.env` (API keys + secrets, **required** for secrets).
- **Precedence ladder (highest → lowest):**
  1. CLI arguments (e.g. `hermes chat --model anthropic/claude-sonnet-4`)
  2. `~/.hermes/config.yaml`
  3. `~/.hermes/.env` (required for secrets)
  4. Built-in defaults
- **Rule of thumb:** Secrets go in `.env`. Everything else goes in `config.yaml`. When both are set, `config.yaml` wins for non-secret settings. Org deployments can pin values via a system-level Managed Scope.
- **`hermes config set KEY VAL` auto-routes:** API keys are saved to `.env`, everything else to `config.yaml`. You should NOT hand-edit the YAML for secrets.
- **`${VAR}` substitution:** reference env vars inside `config.yaml` with `${VAR_NAME}` (e.g. `auxiliary.vision.api_key: ${GOOGLE_API_KEY}`). Supports multiple refs like `"${HOST}:${PORT}"`. Undefined vars stay verbatim as `${UNDEFINED_VAR}`. Only `${VAR}` is expanded — bare `$VAR` is NOT.
- **Provider timeouts** are configured in `config.yaml` via `providers.<id>.request_timeout_seconds` (provider-wide) and per-model `timeout_seconds` overrides, plus `stale_timeout_seconds`. Not wired for AWS Bedrock.
- **Logs auto-redact secrets** in `~/.hermes/logs/` (errors.log, gateway.log).

## 4. Best Practices Discovered
- Keep secrets exclusively in `.env`; never commit them to `config.yaml` or any tracked project file.
- Use `hermes config set` instead of editing YAML by hand — it routes secrets correctly.
- For upstream service keys consumed by a `config.yaml` value, prefer `${VAR}` substitution so the secret stays in `.env` and the config stays portable.
- Prefer CLI args for one-off overrides (highest precedence) rather than mutating files.
- Org/pinned deployments: use Managed Scope for fleet-pinned values.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** Project Zero is a governance/markdown repo (`AGENTS.md`, `PROJECT.md`, `knowledge/`). It holds no project-local `.env` — secrets are delegated to the global Hermes `~/.hermes/.env` / `config.yaml`, which is fully consistent with the rule of thumb. ADR-003 lists "secrets management" as a *guiding principle* but codifies no actual policy or Operational Rule.
- **Gaps & Anti-Patterns:**
  - No documented Alfred-side rule stating "secrets live in `~/.hermes/.env`, never in `config.yaml` or project files" — the principle exists in ADR-003 but is unoperationalized.
  - No captured guidance that `hermes config set` auto-routes secrets, so future maintainers might hand-edit YAML and risk committing a key.
  - No documented `${VAR}` substitution convention for any Project Zero tool that bridges to external services.

## 6. Recommended Improvements
1. **Operationalize the ADR-003 principle:** add a short "Secrets Management" rule to `AGENTS.md` (or `PROJECT.md`) stating secrets belong ONLY in `~/.hermes/.env`/`config.yaml`, never in tracked project files, and that `hermes config set` is the canonical write path.
2. **Document `${VAR}` substitution** as the approved bridge pattern for any future MCP server or external-service tool config (links to the MCP lesson already captured).
3. **Lower-priority hygiene:** confirm no historical commit in `project-zero` ever introduced a literal key (grep the repo for `sk-`/`AKIA`/`ghp_`) — a one-line `scripts/maintenance/` check could be added, but only if a need is demonstrated.

## 7. Risk Assessment
- **Substitution fragility:** a typo'd `${VAR}` silently stays verbatim (`${UNDEFINED_VAR}`) with no error — a misconfigured external service fails opaquely at call time, not at load. Mitigate by validating required vars before use.
- **Cross-profile scope:** `~/.hermes/.env` is global to the profile; per-project/per-cron secret isolation is NOT provided by this mechanism (relevant to the separate Kanban worker-isolation question in the backlog). Switching to MCP/`docker_forward_env` is the documented path for scoping.
- **Bedrock caveat:** provider timeouts are not wired for AWS Bedrock — irrelevant unless we adopt Bedrock.

## 8. Next Learning Topic
How do the 6 terminal backends differ (local, docker, ssh, modal, daytona, singularity) and when should Project Zero switch the Self-Improvement / publish loops to an isolated backend (e.g. docker) for safer execution? Study `home_mode`, `container_persistent`, and the persistent-sandbox-VM behavior. Source: https://hermes-agent.nousresearch.com/docs/user-guide/features/tools

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
