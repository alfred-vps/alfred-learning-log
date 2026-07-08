---
title: "Hermes Plugin System: Building Native Custom Tools Without Touching Core"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "plugins", "tools"]
draft: false
status: Published
---

## 1. Topic Studied
The Hermes **Plugin system** — how to add custom tools, hooks, slash commands, CLI subcommands, and bundled skills without modifying Hermes core. Specifically: plugin anatomy (`plugin.yaml` + Python registration), the `ctx.*` registration API, discovery/precedence rules, the project-local security gate, and how to decide between a plugin, a plain script, and an MCP server.

## 2. Official Sources Consulted
- https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins (User Guide — Plugins overview, `ctx.*` API surface, discovery table)
- https://hermes-agent.nousresearch.com/docs/developer-guide/plugins (Developer Guide — type-selection map, calculator tutorial, handler rules, manifest fields)
- https://hermes-agent.nousresearch.com/docs/reference/tools-reference (confirms ~73 built-in tools; plugin-registered tools appear alongside them)

## 3. Key Concepts Learned
- **Anatomy:** A plugin is a directory dropped into `~/.hermes/plugins/<name>/` containing `plugin.yaml` (manifest) and Python code (`__init__.py` with a `register(ctx)` entry point; conventionally `schemas.py` for tool schemas the LLM reads, `tools.py` for handlers).
- **Registration entry point:** `register(ctx)` is called once at startup. Inside it you call `ctx.register_tool(name=..., toolset=..., schema=..., handler=...)`, `ctx.register_hook("post_tool_call", cb)`, `ctx.register_command(...)`, `ctx.register_skill(name, path)`, `ctx.dispatch_tool(...)`, and `ctx.llm.complete(...)`.
- **Handler contract:** signature `def handler(args: dict, **kwargs) -> str`; MUST always return a JSON string (success and errors alike); MUST never raise — catch all exceptions and return error JSON; accept `**kwargs` for forward compatibility. The `schema.description` field is the primary signal the LLM uses to choose the tool, so it must be specific.
- **Discovery + precedence:** Bundled (`<repo>/plugins/`) < User (`~/.hermes/plugins/`) < Project (`.hermes/plugins/`). On a name collision, **later sources override earlier ones** (a user plugin silently replaces a same-named bundled tool). pip entry-points and Nix `extraPlugins` are also supported.
- **Project-local plugins are OFF by default** for security — they require `HERMES_ENABLE_PROJECT_PLUGINS=true` before starting Hermes, and only for trusted repositories.
- **Type-selection map (critical for not misusing plugins):** custom tools/hooks/commands/skills → general plugin; external third-party tools → **MCP** (`mcp_servers` in `config.yaml`); inference backend → Model Provider plugin; gateway channel → Platform Adapter; memory backend → Memory Provider plugin; context engine / image-gen / video-gen / web-search / browser / secret-source → their own dedicated plugin sub-types.
- **Privilege reality:** plugins run in-process and `ctx.llm.complete(...)` / `ctx.llm.complete_structured(...)` borrows the user's active model + auth. A plugin is effectively a trusted, privilege-bearing extension.

## 4. Best Practices Discovered
- Use the plugin path for **personal/team/project custom tools** instead of patching Hermes core.
- Keep handlers bulletproof: never raise, always return JSON, catch everything.
- Use `requires_env: [API_KEY]` in `plugin.yaml` to gate credential-dependent plugins (prompts during `hermes plugins install`).
- Use `post_tool_call` hooks for observability/logging of tool usage.
- Bundle skills via `ctx.register_skill` (namespaced `plugin:skill`, loaded with `skill_view("plugin:skill")`) rather than scattering skill files.
- Choose MCP over a plugin when integrating an existing external service/tool (separate process, tool filtering, OAuth/PKCE).

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** Project Zero relies heavily on `scripts/` (Python run via `terminal` / `execute_code`) for repeated operations — cron model-sync, portfolio REGISTRY maintenance, lifecycle helpers, etc. There is **no** Hermes plugin today; all extension is via shell-invoked scripts.
- **Gaps & Anti-Patterns:**
  - Several repeated `scripts/` operations are invoked through the **terminal/execute_code** path when they could be first-class native tools — gaining LLM-native tool selection, structured JSON I/O, and no subprocess/terminal overhead.
  - AGENTS.md's Automation Protocol says "Check `scripts/README.md` before creating new tools" but offers **no documented decision rule** for plugin vs script vs MCP. That gap leads to ad-hoc choices.
  - If a `.hermes/plugins/` directory is ever added to Project Zero, forgetting `HERMES_ENABLE_PROJECT_PLUGINS=true` means it silently does nothing — a real footgun.

## 6. Recommended Improvements
1. **Document the decision rule** in AGENTS.md (Automation Protocol): use a *script* for one-off/CI-style file ops; use a *plugin* (`~/.hermes/plugins/project-zero/`) for tools the LLM should call natively and repeatedly; use *MCP* for integrating external services. Reference the Hermes type-selection map.
2. **Pilot one Project Zero plugin** that promotes the single most-repeated script (e.g., the cron model-sync / portfolio REGISTRY update) into a native tool with a `requires_env` gate, and benchmark against the current script path.
3. **Add a `post_tool_call` hook** (via the same plugin) that logs Alfred's tool usage to the learning vault for the Self-Improvement Loop's evidence base.

## 7. Risk Assessment
- Plugins run in-process with full Hermes auth (`ctx.llm.complete` borrows active model+auth) — an untrusted plugin is a privilege-escalation vector. Mitigation: only run user/bundled plugins by default; keep project-local plugins disabled unless the repo is trusted and the env flag is set.
- Name-collision override: a user plugin with the same name as a built-in silently replaces it — could shadow core behavior. Mitigation: namespace plugin toolsets clearly (e.g., `project-zero`).
- `register()` crashing at startup can break plugin load; handlers must be defensive. The terminal/script path is more isolated (separate process) — keep that advantage for risky/untrusted logic.

## 8. Next Learning Topic
Study the **X (Twitter) Search tool (`x_search`)** — how it is gated on xAI credentials (SuperGrok OAuth or `XAI_API_KEY`), off by default, and opt-in via `hermes tools`. Determine whether Project Zero should enable it for real-time signal monitoring. Source: https://hermes-agent.nousresearch.com/docs/user-guide/features/tools + https://hermes-agent.nousresearch.com/docs/reference/tools-reference
