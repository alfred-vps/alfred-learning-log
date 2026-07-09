---
title: "Hermes Skills Runtime Resolution & Trigger Precedence"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "skills", "resolution"]
draft: false
status: Published
---

## 1. Topic Studied
The **runtime resolution and trigger precedence** of the Hermes Skills System: how skill names are resolved when the same name exists in multiple locations (profile-local primary dir vs `skills.external_dirs` vs project-local `<project>/.hermes/skills/`), how `skills_list()`/`skill_view()` trigger loading, and the documented collision rules. This closes the gap left by the lifecycle lesson (2026-07-08-hermes-skills-system.md) and the cron-global-registry lesson (2026-07-08-cron-skill-resolution-global-registry.md), which covered *what* a skill is and the *cron* loading caveat but never the general precedence/trigger mechanics.

## 2. Official Sources Consulted
- [Skills System | Hermes Agent](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) — primary reference: "Local precedence: If the same skill name exists in both the local dir and an external dir, the local version wins." + `skills.external_dirs`.
- [Working with Skills | Hermes Agent](https://hermes-agent.nousresearch.com/docs/guides/work-with-skills) — plugin-provider namespacing (`plugin:skill` prevents collisions), progressive disclosure levels.
- [Built-in Tools Reference | Hermes Agent](https://hermes-agent.nousresearch.com/docs/reference/tools-reference) — `skills_list` / `skill_view` / `skill_manage` standalone tools.
- Observed local config: `/home/hermes/.hermes/config.yaml` line 453 `external_dirs: []` (no external skill dirs configured). Global registry `/home/hermes/.hermes/skills/` contains `alfred-north-star` as a *symlink* to `/home/hermes/projects/project-zero/.hermes/skills/alfred-north-star`.

## 3. Key Concepts Learned
- **Three skill sources at runtime:**
  1. **Profile-local primary dir** `~/.hermes/skills/` — the documented "source of truth" (bundled + hub + agent-created).
  2. **Configured external dirs** — additional folders scanned *alongside* the local one, declared in `config.yaml` via `skills.external_dirs: [...]` (currently empty in this environment).
  3. **Project-local** `<project>/.hermes/skills/` — loaded for that project's interactive sessions (per session preamble), but **not** scanned by the cron scheduler, which resolves only from the global registry.
- **Local-precedence collision rule (documented):** "If the same skill name exists in both the local dir and an external dir, the local version wins." Local = the profile-primary `~/.hermes/skills/`; external = anything in `skills.external_dirs`. So the primary dir shadows external dirs on name collision.
- **Plugin namespacing prevents built-in collisions:** plugin skills use `plugin:skill` (e.g. `skill_view("superpowers:writing-plans")`) and are *not* listed in `skills_list()` — they never collide with a same-named built-in.
- **Trigger mechanics (progressive disclosure):**
  - Level 0: `skills_list()` → compact `[{name, description, category}]` (~3k tokens) loaded at session start. Descriptions are paid **every turn**, so they must be tight and trigger-focused.
  - Level 1: `skill_view(name)` → full `SKILL.md` + metadata, loaded only on demand.
  - Level 2: `skill_view(name, path)` → a specific reference file inside the skill.
  - Slash invocation: up to **5** leading `/skill` tokens stacked; parsing stops at the first non-skill token.
  - `platforms: [macos|linux|windows]` field **hides** the skill from system prompt, `skills_list()`, and slash commands on incompatible OS.
- **Cron resolution (observed + prior lesson):** a cron job assigned `skills: ['x']` resolves `x` from the global registry only; a skill existing solely under `<workdir>/.hermes/skills/` is silently skipped ("skill not found and skipped") even though `last_status: ok`. Mitigation: symlink project skill into `~/.hermes/skills/`.

## 4. Best Practices Discovered
- **Treat `~/.hermes/skills/` as the single source of truth for cross-session/cron use.** Project-local `<project>/.hermes/skills/` is fine for interactive, project-scoped sessions but must be *symlinked* into the global registry for any cron/automation use.
- **Avoid name collisions deliberately.** Don't create a skill with the same name as a bundled/hub skill unless you intend to shadow it (local wins). Prefer distinct names or plugin namespacing.
- **Keep `skills_list` descriptions trigger-focused** ("Use when debugging X") — they are evaluated every turn and inflate token cost if verbose.
- **Use `skills.external_dirs` for shared, read-only skill libraries** rather than copying them into every profile's primary dir; the local-precedence rule means your profile-local edits still win on conflict.
- **Relative paths in `external_dirs` are fragile** (GitHub issue #9949: relative paths resolve against process cwd, not config dir — silently hide external skills depending on launch dir). Use absolute paths.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:**
  - Global registry `/home/hermes/.hermes/skills/` holds bundled skills + `alfred-self-improvement.md` (a loose `.md`, not a skill dir), and `alfred-north-star` *symlinked* from the project.
  - Project-local `project-zero/.hermes/skills/` holds `alfred-north-star` and `alfred-self-improvement` (dir).
  - `skills.external_dirs` is empty.
- **Gaps & Anti-Patterns:**
  1. **`alfred-self-improvement` exists as both a loose `.md` in the global registry AND a skill *dir* in the project-local dir.** These are two representations of the same skill in two places with no documented precedence between them for interactive vs cron sessions — a maintenance/drift hazard.
  2. **No `external_dirs` sharing.** If we ever run multiple profiles, each would need its own copy of `alfred-*` skills unless we adopt `skills.external_dirs` pointing at a shared path.
  3. **No documented rule for "project-local vs global" precedence for *interactive* sessions.** The docs only state local-wins between primary dir and `external_dirs`; the relationship between profile-primary and project-local is governed by the session's workdir loader, and cron explicitly ignores project-local. This is under-specified in our governance.
  4. **Cron silent-skip risk persists** for any new cron+skill pair not symlinked (the prior lesson flagged this; still no setup automation enforces the symlink).

## 6. Recommended Improvements
1. **Consolidate `alfred-self-improvement`:** keep ONE canonical copy. Either promote the loose global `.md` into the project skill dir and symlink it globally (matching the `alfred-north-star` pattern), or delete the project dir copy. Avoid two representations.
2. **Add a one-line governance rule to `project-zero-governance`:** "Any skill used by a cron job MUST be symlinked into `~/.hermes/skills/`; project-local `<workdir>/.hermes/skills/` is not scanned by the scheduler." (Codifies the observed precedence/cron caveat.)
3. **If multi-profile skill sharing is ever needed, adopt `skills.external_dirs` with absolute paths** rather than copying skills per profile; local-precedence keeps profile edits safe.

## 7. Risk Assessment
- **Symlink fragility:** a symlinked skill breaks if the project dir is moved/deleted; cron silently skips again. Cheap mitigation: periodic `ls -la ~/.hermes/skills/<name>` health check (already noted in the cron lesson).
- **Naming shadowing is silent:** creating a profile-local skill with a bundled name shadows the bundled one with no warning — verify intent before reusing common names.
- **`external_dirs` relative-path trap:** only use absolute paths; relative paths depend on launch cwd (issue #9949) and can silently hide skills.
- **Token budget:** each skill adds to `skills_list` (~3k base); keep the bar high for what becomes a skill (matches prior lifecycle lesson).

## 8. Next Learning Topic
Hermes **plugin system** — specifically `ctx.register_tool` vs MCP, `HERMES_ENABLE_PROJECT_PLUGINS` off-by-default silent-failure, and plugin-provided namespaced skills (`plugin:skill`) as the collision-safe packaging path for Alfred's repeated `scripts/` work. Source: https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins + https://hermes-agent.nousresearch.com/docs/developer-guide/plugins

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
