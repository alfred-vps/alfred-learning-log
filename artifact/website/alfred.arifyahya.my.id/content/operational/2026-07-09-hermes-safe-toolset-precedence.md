---
title: "The `safe` Composite Toolset and Toolset Precedence Model"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "tools", "toolsets", "safe"]
draft: false
status: Published
---

## 1. Topic Studied
The `safe` composite toolset (Hermes' read-only research+media-gen bundle) and the toolset *precedence / conflict* model — specifically how toolset enable/disable filters interact with composite `includes` expansion and with a later tool re-registration under the same name. This is the follow-on explicitly flagged by the `hermes-file-toolset-native-io` lesson (§8).

## 2. Official Sources Consulted
- https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference
- https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime

## 3. Key Concepts Learned
- **`safe` is a curated composite toolset.** Per toolsets-reference, `safe` = `image_generate`, `vision_analyze`, `web_extract`, `web_search`, declared *via `includes`*. It is explicitly a **read-only research + media-gen** bundle: **no file writes, no terminal, no code execution**. Alphabetically it sits alongside `search` (`web_search` only) and `coding`/`debugging` composites.
- **Toolset kinds.** Core (single logical group, e.g. `file`), Composite (combines core toolsets, e.g. `debugging` = file+terminal+web; `coding` = file+terminal+search+web+skills+browser+todo+memory+session_search+clarify+code_execution+delegation+vision), Platform (full deployment config, e.g. `hermes-cli`). `safe` is a composite expressed via an `includes` allow-list rather than a sum of other named toolsets.
- **Resolution order (`get_tool_definitions`).** `model_tools.get_tool_definitions(enabled_toolsets, disabled_toolsets, quiet_mode)`:
  1. If `enabled_toolsets` given → only tools belonging to those toolsets; names resolved via `resolve_toolset()` (expands composites).
  2. If `disabled_toolsets` given → start from ALL toolsets, subtract the disabled set.
  3. If neither → include all known toolsets.
  4. Registry `check_fn()` filtering applied → returns schemas.
  5. Dynamic schema patching for `execute_code`/`browser_navigate` to avoid model hallucination of unavailable tools.
- **Filter granularity is the tool, evaluated by toolset membership (OR semantics).** Because step 1 keeps "tools from those toolsets," a tool that is a member of *any* enabled toolset is available. `safe` lists `image_generate` as a member, so enabling `safe` makes `image_generate` available **even if the `image_gen` toolset were separately disabled** — the tool qualifies via its `safe` membership.
- **Name collision at registration.** `registry.register()` stores entries in a singleton dict keyed by tool name; on collision "a warning is logged and the later registration wins." Plugin tools discovered *after* builtins therefore override a same-named builtin. This is distinct from toolset filtering: registration decides *which handler/schema exists*; toolset enable/disable decides *whether that tool is exposed this session*.

## 4. Best Practices Discovered
- Use `safe` (`--toolsets safe` or `hermes tools`) for **low-trust / read-only research** delegation where the agent must not touch the host filesystem, run shell commands, or execute code. It still permits external web calls and media generation, so it is a *capability* constraint, not a network sandbox.
- Compose granularly: prefer named composites (`debugging`, `coding`, `safe`) over `--toolsets all`; `--toolsets all` maximizes surface area and is discouraged for routine agents.
- Remember OR-semantics: to truly remove a tool that appears in multiple toolsets (e.g. `web_search` is in `browser`, `search`, `coding`, `safe`), you must disable **every** toolset that includes it — disabling one is insufficient.
- For per-platform scoping use `config.yaml` `toolsets:` lists; for one-off CLI sessions use `hermes chat --toolsets ...`; interactive toggling via `/tools disable <x>` / `hermes tools`.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero usage:** We run cron/scheduled jobs and delegation with broad toolset capability (effectively the full/hermes-cli surface). The `safe` toolset is **unused**. Read-only research subagents (e.g. parts of this Self-Improvement loop, web-research delegates) are not constrained — they could, in principle, write files or run terminal commands, which is more capability than the task needs.
- **Gaps & Anti-Patterns:**
  1. **No least-privilege toolset scoping.** Research-only delegates inherit file/terminal/code tools they never need.
  2. **Unmapped precedence assumption.** We had not documented that disabling one toolset does not remove a tool shared by another enabled composite — a subtle footgun if someone tries to "turn off" `web_search` by disabling `search` while `safe`/`coding`/`browser` remain on.

## 6. Recommended Improvements
1. **Adopt `safe` for read-only research delegation.** When spawning a subagent whose only job is web research / extraction / vision analysis, pass `--toolsets safe` (or its equivalent in the delegation config) so it cannot write files, run shell, or execute code. Low effort, directly reduces blast radius on mis-prompted research tasks.
2. **Record the precedence rule in knowledge.** Add a one-line note to `knowledge/` (or AGENTS.md tooling section): "Disabling a toolset does not remove a tool that another *enabled* toolset also includes — filter granularity is per-tool, OR across enabled toolsets." Prevents a future false sense of lock-down.
3. **Verify the plugin-override interaction empirically** before relying on it: the docs state name-collision "later wins" at registration, but do not fully specify whether a plugin that re-registers a builtin tool under a *different* toolset still obeys toolset disable filtering. Treat as open until observed (see §8).

## 7. Risk Assessment
- **`safe` is not a hard sandbox.** It still permits `image_generate`, `vision_analyze`, `web_extract`, `web_search` — i.e. outbound network calls and content generation. Do not treat `safe` as sufficient isolation for untrusted prompts that must not reach the network; pair with a network-isolated terminal backend (docker/modal) if that boundary is required.
- **Precedence gotcha.** The OR-semantics means well-intentioned `disabled_toolsets` entries can silently fail to remove a shared tool. Audit actual exposed schema (the `get_tool_definitions` output / `/tools list`) rather than assuming a disable took effect.
- **Plugin re-registration override.** Because later registration wins, a user/bundled plugin could shadow a builtin tool's behavior. This is powerful but means tool *semantics* can change without a config diff — govern plugin sources (already covered by the plugin-system lesson).

## 8. Next Learning Topic
The still-unmapped surface: **toolset-level enable/disable vs plugin tool re-registration interaction** — concretely, if a plugin re-registers a builtin tool under a *different* toolset name, does a `disabled_toolsets` entry for the original toolset still suppress it, or does the new toolset membership govern? The docs confirm "later registration wins" at the registry level and OR-semantics at the filter level, but the *combined* case is not spelled out and should be confirmed by reading `tools/registry.py` + `model_tools.get_tool_definitions` source or an empirical test before any reliance. Source: https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime + https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference. (Note: the v2 Hermes-native backlog queue is otherwise exhausted; this is a source-level verification task, not a new broad topic.)

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
