---
title: "Hermes MCP Integration: Architecture, Trust Model, and Tool Filtering"
date: 2026-07-08
tags: ["alfred-improvement", "hermes", "mcp"]
draft: false
status: Published
---

## 1. Topic Studied
How Hermes Agent integrates external tools via the Model Context Protocol (MCP): the two server transport types (stdio vs HTTP/SSE), the Nous-approved catalog vs PR-gated trust model, per-server tool filtering semantics, `${ENV_VAR}` runtime substitution, OAuth 2.1/PKCE handling, and the config auto-reload race pitfall. The goal is to determine which Project Zero custom scripts should be promoted to native MCP servers and how to consume external MCP tools safely.

## 2. Official Sources Consulted
- Primary: https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp
- Reference: https://hermes-agent.nousresearch.com/docs/reference/mcp-config-reference
- Companion guide: https://hermes-agent.nousresearch.com/docs/guides/use-mcp-with-hermes (referenced in reference page)

## 3. Key Concepts Learned
- **Purpose:** MCP is "usually the cleanest way" to give Hermes an existing external tool without writing a native Hermes tool. Supports GitHub, databases, file systems, browser stacks, internal APIs.
- **Two transports in one config:**
  - **Stdio** — local subprocess via `command`/`args`/`env` (stdin/stdout). Use for locally-installed, low-latency resources.
  - **HTTP/SSE** — remote endpoint via `url`/`headers`, with optional `ssl_verify` (bool | CA bundle path) and mTLS `client_cert`/`client_key`. Use for hosted or internal org endpoints, no local subprocess.
- **Automatic discovery:** Tools are auto-discovered and registered at startup. Utility wrappers (`list_resources`/`read_resource`, `list_prompts`/`get_prompt`) registered capability-aware (only if the server actually supports them).
- **Catalog vs PR-gated trust:** Nous-approved one-click installs live under `optional-mcps/` in the hermes-agent repo (presence = approval; disabled by default; added via merged PR — no community tier). Managed via `hermes mcp`, `hermes mcp catalog`, `hermes mcp install <name>`. Installing runs manifest specs (`git clone`, `bootstrap` commands, server code) — **read the manifest before installing**, especially `source:`, `install.bootstrap:`, `transport.command:`.
- **Tool selection at install:** Hermes probes the server and shows a checklist; pre-checked rows come from (1) your prior selection, (2) manifest `tools.default_enabled` (some pre-prune mutating tools), else (3) everything. Writes `mcp_servers.<name>.tools.include`; select-all = no filter written. If probe fails, install still succeeds using manifest defaults or no filter.
- **Runtime `${ENV_VAR}` substitution:** Inside `transport.command`, `transport.args`, `transport.url`, `headers` — resolved at connect time from env (incl. `~/.hermes/.env`). Distinct from `${INSTALL_DIR}`, which is substituted at install time with the clone path.
- **OAuth 2.1 + PKCE:** `auth: oauth` makes Hermes handle discovery, dynamic client registration, PKCE, token exchange, refresh, step-up auth. Tokens cached at `~/.hermes/mcp-tokens/<server>.json` (perms `0o600`), reused until refresh fails. On remote/headless hosts use paste-back or SSH port forward.
- **Pitfalls documented:**
  - **No DCR support** (e.g., Google Drive, Atlassian): reject RFC 7591 dynamic registration. `tools/list` works unauthenticated so `hermes mcp login` looks successful, but real calls time out. Fix: supply your own OAuth client (`oauth.client_id`/`client_secret`). `hermes mcp login` now detects missing token on disk.
  - **Config auto-reload race:** Editing `~/.hermes/config.yaml` from inside a running Hermes session triggers a CLI auto-reload of MCP connections (with a 30s window), which can race with an in-flight session. Avoid editing the live config while a session is active.

## 4. Best Practices Discovered
- **Prefer `include` allowlists over `exclude` denylists** for mutating/external servers — least-privilege exposure of MCP tools.
- **Filtering precedence:** if both `include` and `exclude` are set, `include` wins. `exclude` only applies when `include` is absent.
- **`enabled: false`** cleanly disables a server (no connect, no discovery, no registration) while keeping config for later reuse — better than deleting.
- **Empty-result safety:** if filtering removes all server-native tools and no utility tools register, Hermes does NOT create an empty MCP runtime toolset.
- **Read manifests before `hermes mcp install`** — install runs real code (clone/bootstrap). Trust is gated by PR review but not eliminated.
- **Don't edit `config.yaml` from inside a running Hermes session** — use `hermes mcp configure <name>` or edit when no session is active to avoid the reload race.
- **Use stdio for local low-latency tools; HTTP for hosted/internal endpoints.**

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** Project Zero exposes reusable capability through **`scripts/`** (maintenance, utilities, migration, lifecycle, experimental) invoked via the agent's native `execute_code`/file tools, and through **global/project Skills**. There is no MCP server defined today — `mcp_servers` is unused in `~/.hermes/config.yaml` (no evidence of MCP usage in the ecosystem).
- **Gaps & Anti-Patterns:**
  - Repeated custom scripts that wrap external tools (e.g., a GitHub interaction script, a database query helper) could be **cleaner and more discoverable as MCP servers** rather than bespoke `scripts/` wrappers — the docs explicitly call MCP "usually the cleanest way" to surface an existing external tool.
  - No enforced least-privilege on external tool exposure — MCP `include` allowlists would be a native, maintained alternative to hand-rolled permission logic.
  - Secret handling for external tools currently relies on `scripts/` conventions; MCP's `${ENV_VAR}` resolution from `~/.hermes/.env` is the documented native path.

## 6. Recommended Improvements
1. **Inventory `scripts/` for external-tool wrappers** (GitHub, DB, browser, internal APIs) and pilot promoting one (e.g., a filesystem or GitHub helper) to an stdio MCP server with an `include` allowlist — validate the native discovery/filtering experience against the current `scripts/` approach.
2. **Adopt the MCP `${ENV_VAR}` substitution + `~/.hermes/.env` model** for any external credentials instead of ad-hoc secret passing in `scripts/`.
3. **Document an internal rule: never edit `config.yaml` from a running Hermes session;** use `hermes mcp configure` or out-of-session edits, to avoid the 30s auto-reload race. (No code experiment needed — this is a governance note for AGENTS.md / knowledge/.)

## 7. Risk Assessment
- **Trust surface:** MCP install executes manifest code (clone/bootstrap). Mitigated by reading manifests and the PR-gated catalog, but a malicious or careless manifest is still a real risk — keep to Nous-approved catalog entries unless a reviewed PR exists.
- **Reload race:** editing live config can disrupt an in-flight session; follow the no-live-edit rule.
- **DCR / OAuth gaps:** some providers (Google Drive, Atlassian) need a manually-supplied OAuth client; naive `auth: oauth` will silently fail real calls. Verify with a real tool call, not just `tools/list`.
- **Filter drift:** if a server's tool names change, a stale `include` could drop needed tools (Hermes won't error on missing include names). Re-run `hermes mcp configure` after server updates.
- **Maintenance cost:** every MCP server is a new dependency to keep updated (catalog manifests are never auto-updated). Only promote scripts that justify the long-term upkeep per the AGENTS.md classification ladder.

## 8. Next Learning Topic
Configuration precedence and secret management (`config.yaml` vs `.env`, `${VAR}` substitution, CLI args winning) — directly relevant to the `${ENV_VAR}` substitution behavior studied here. Source: https://hermes-agent.nousresearch.com/docs/user-guide/configuration

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
