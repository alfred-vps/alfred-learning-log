---
title: "Hermes Skills System — Native Reusable Knowledge vs Project Zero's knowledge/ Markdown"
date: 2026-07-08
tags: ["alfred-improvement", "hermes", "skills"]
draft: false
status: Published
---

## 1. Topic Studied
The Hermes **Skills System**: end-to-end lifecycle, progressive disclosure (Level 0→2), the `SKILL.md` format, the `/learn` auto-generation command, skill bundles, plugin-provided skills, and the native `.no-bundled-skills` opt-out. Specifically evaluated against the open question in `limitations_backlog.md`: *should Project Zero's `knowledge/` Markdown patterns be promoted into native portable Skills vs staying as governed Markdown?*

## 2. Official Sources Consulted
- [Skills System | Hermes Agent](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) — primary reference (lifecycle, progressive disclosure, SKILL.md format, `/learn`, bundles, opt-out).
- [Working with Skills | Hermes Agent](https://hermes-agent.nousresearch.com/docs/guides/work-with-skills) — install, hub, plugin skills, config frontmatter, creating skills.
- Cross-checked against the bundled `hermes-agent-skill-authoring` skill (`~/.hermes/skills/software-development/hermes-agent-skill-authoring/SKILL.md`) already present in this environment.

## 3. Key Concepts Learned
- **Skills are on-demand knowledge documents** loaded via progressive disclosure to minimize token usage. Compatible with the [agentskills.io](https://agentskills.io/specification) open standard.
- **Progressive disclosure levels:**
  - Level 0: `skills_list()` → compact `[{name, description, category}]` (~3k tokens, loaded at session start).
  - Level 1: `skill_view(name)` → full `SKILL.md` content, loaded only when needed.
  - Level 2: `skill_view(name, path)` → a specific reference file inside the skill.
  - **Skills cost zero tokens until used** — this is the core efficiency property.
- **Source of truth:** `~/.hermes/skills/` (bundled + hub + agent-created). Project-local skills live under `<project>/.hermes/skills/` and are loaded for that project's sessions.
- **SKILL.md format:** YAML frontmatter (`name`, `description` ≤1024 chars, `version`, optional `platforms`, optional `metadata.hermes.{tags, category, fallback_for_toolsets, requires_toolsets, config}`), then Markdown body with `## When to Use`, `## Procedure`, `## Pitfalls`, `## Verification`.
- **`/learn` command:** turns reference material (a file, URL, or "how I just did X") into a standards-compliant skill without hand-writing `SKILL.md`. Agent gathers material with its own tools, then authors the skill. A **write-approval gate** applies if enabled (uses `skill_manage`). No separate ingestion engine.
- **Skill bundles:** repeated `/skill-a /skill-b` combos can be packaged as a bundle.
- **Plugin-provided skills:** namespaced `plugin:skill`, opt-in, not in `skills_list`.
- **Opt-out:** `hermes skills opt-out` (or `--no-skills` at install) writes a `.no-bundled-skills` marker; `--remove` deletes only byte-identical bundled skills. Edited/agent-written skills are always kept.

## 4. Best Practices Discovered
- **Keep descriptions trigger-focused**, not task-focused: "Use when debugging X" beats "Debug X". Descriptions are paid every turn in `skills_list`, so they must be tight.
- **Push branch-specific/bulky reference behind pointers** (`references/`, `templates/`, `scripts/`) — load Level 2 only when needed.
- **Every ordered step needs a checkable completion criterion.**
- **Prune duplication and no-op prose** — a skill should get shorter/sharper over time.
- **Don't over-promote.** The official model is: capture reusable *procedure* as a skill; keep one-off notes as plain docs. Over-promotion inflates the `skills_list` token budget.
- **Prefer extending an existing skill over creating a narrow sibling.**

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** Hybrid model already in place:
  - `knowledge/` holds **28 Markdown files** (architecture patterns, kanban theory, execution-tier guidance, research synopses).
  - `.hermes/skills/` holds **2 project-local skills**: `alfred-self-improvement` and `alfred-north-star`.
  - `project-zero-governance` skill (global) already encodes the classification ladder `Memory → Knowledge → Documentation → ADR → Skill → Automation …`, i.e. the governance model *expects* reusable procedure to graduate from Knowledge → Skill.
  - The bundled `hermes-agent-skill-authoring` skill already documents authoring conventions and the `write_file` + `git add` path for in-repo skills.
- **Gaps & Anti-Patterns:**
  1. **Procedure-shaped knowledge stranded in `knowledge/` as flat Markdown.** `execution-tier-selection.md` already has a `When to Use / NOT for / Procedure / Decision Framework` shape that maps 1:1 onto `SKILL.md` — but it is not a loadable skill, so it costs tokens only when an agent happens to read it (or AGENTS.md points at it), instead of being natively invoked by trigger.
  2. **No promotion trigger rule.** There is no documented criterion that says "when a `knowledge/` file is referenced N times or codifies trigger→procedure→pitfalls, promote it to a project-local skill." The classification ladder implies it but nothing enforces it.
  3. **`/learn` underutilized for auto-promotion.** The backlog explicitly asked whether repeated lesson patterns could be auto-promoted via `/learn`. Today, `/learn` targets `~/.hermes/skills/` via `skill_manage` and **cannot write to project-local `project-zero/.hermes/skills/`** — and in restricted cron contexts `skill_manage` is unavailable entirely (documented in `project-zero-governance` pitfall #6). So the auto-promotion path is currently blocked for the very skills we'd want to promote.
  4. **`knowledge/README.md` is stale** — it still says "No domain knowledge has been added yet," contradicting 28 real files. Minor but a governance-hygiene gap.

## 6. Recommended Improvements
1. **Create a Promotion Audit.** Classify each `knowledge/` file as **Skill-candidate** (encodes trigger→procedure→pitfalls, invoked during orchestration) vs **Keep-as-Markdown** (conceptual/theoretical reference, e.g. `cognitive-architectures.md`, `alfred-north-star-research.md`). Drafted in `experiments/skills-promotion-audit/README.md`.
2. **Promote the strongest candidate to a project-local skill:** `execution-tier-selection.md` → `project-zero/.hermes/skills/execution-tier-selection/SKILL.md` (drafted in the experiment dir as `execution-tier-selection/SKILL.md`). It already matches the Skill body shape; only frontmatter + minor reordering needed. Route via `write_file` (not `skill_manage`, which can't target project-local dirs).
3. **Add a Promotion Trigger rule to `project-zero-governance`:** "When a `knowledge/` file is referenced 3+ times in orchestration/delegation work OR codifies a When-to-Use/Procedure/Pitfalls structure, promote it to a project-local skill via `write_file`." This closes the gap between the classification ladder and actual behavior.

## 7. Risk Assessment
- **Token budget:** Each skill adds to the `skills_list()` (~3k base). Promoting ~5–8 procedure files is safe; promoting all 28 would be wasteful. Keep the bar high (trigger-shaped, frequently used).
- **Project-local skills can't be edited by `skill_manage`** in restricted cron contexts — must use `write_file`/`patch`, and the change only becomes visible in a *new* session (skill loader is initialized at session start). Mitigation: document fixes in `AGENT-REVIEW.md` and apply in an unrestricted session.
- **`/learn` limitation:** it writes to the global skills dir and may trigger the write-approval gate; it cannot target `project-zero/.hermes/skills/`. For project-local promotion, hand-authoring via `write_file` remains the reliable path.

## 8. Next Learning Topic
**Hermes persistent memory** (`MEMORY.md` / `USER.md`, 2,200/1,375 char caps, frozen-snapshot pattern, no-auto-compact, the 80%-consolidate rule, duplicate prevention). Study how Project Zero should consolidate its file-based knowledge (`knowledge/`, `AGENT-REVIEW.md`) against the native memory system. Source: https://hermes-agent.nousresearch.com/docs/user-guide/features/memory

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
