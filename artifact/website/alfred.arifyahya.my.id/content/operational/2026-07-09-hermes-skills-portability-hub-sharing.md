---
title: "Hermes Skills Portability — agentskills.io Standard & Skills Hub Sharing"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "skills", "portability", "curriculum:hermes"]
draft: false
status: Draft
---

## 1. Topic Studied
The **portability and shareability** of Hermes skills — specifically: (a) that Hermes `SKILL.md` is compatible with the **agentskills.io open standard**, so a skill authored inside Hermes is consumable by 42+ other agentic clients, and (b) the **Skills Hub** distribution mechanism (`/skills browse|search|install`) that lets Alfred pull optional official skills and arbitrary remote `SKILL.md` URLs into its registry. This closes the gap left by the two prior skills lessons (2026-07-08-hermes-skills-system.md, 2026-07-09-hermes-skills-runtime-resolution-precedence.md), which covered lifecycle + runtime precedence but never the **standardization/sharing/cross-agent reuse** strategic angle.

## 2. Official Sources Consulted
- [Skills System | Hermes Agent](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) — "Compatible with agentskills.io open standard"; Skill Output & Media Delivery; Skill Bundles.
- [Working with Skills | Hermes Agent](https://hermes-agent.nousresearch.com/docs/guides/work-with-skills) — Skills Hub: `/skills browse`, `/skills search`, `hermes skills install official/...` and `hermes skills install <url> --name`.
- [Agent Skills Specification | agentskills.io](https://agentskills.io/specification) — the open standard: directory structure, `SKILL.md` frontmatter (`name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools`), progressive-disclosure loading, and the 42-client compatibility list.
- [Agent Skills Overview | agentskills.io](https://agentskills.io) — standard originated by Anthropic; "build once, use across any skills-compatible agent."

## 3. Key Concepts Learned
- **Hermes is agentskills.io-compliant.** A Hermes skill directory (`SKILL.md` + optional `scripts/`, `references/`, `assets/`) is structurally identical to an agentskills.io skill. The standard mandates: `name` (≤64 chars, lowercase `a-z0-9` + hyphens, must match parent dir, no leading/trailing/double hyphens) and `description` (≤1024 chars, "what + when"). Hermes adds an optional `metadata.hermes.*` block (tags, category, `fallback_for_toolsets`, `requires_toolsets`, `config`) that other clients ignore — so a Hermes-authored skill degrades gracefully on non-Hermes clients.
- **Cross-agent reuse is a first-class property, not a side effect.** A skill built once in Hermes runs in Claude, Cursor, VS Code, GitHub Copilot, OpenAI Codex, Goose, Letta, and 35+ other clients. Conversely, any agentskills.io skill can be installed into Hermes. Knowledge written as a skill compounds across the whole toolchain instead of being locked to one product.
- **The Skills Hub is the distribution layer.** Official optional (heavier/niche) skills ship with Hermes but are inactive by default. Install them explicitly:
  - `hermes skills install official/research/arxiv`
  - `hermes skills install https://sharethis.chat/SKILL.md`
  - `hermes skills install https://example.com/SKILL.md --name my-skill`
  - Discovery: `/skills browse`, `/skills search <term>`.
  - Result: the skill dir is copied to `~/.hermes/skills/`, appears in `skills_list`, and becomes a slash command. Takes effect in **new sessions** (`/reset` or `--now` to invalidate prompt cache immediately).
- **Install-from-URL means Alfred can absorb external procedural knowledge with one command** — no manual `write_file` + frontmatter wrangling. Conversely, Alfred can publish its `alfred-*` skills as a URL for reuse elsewhere.
- **Progressive disclosure is standardized** (metadata ~100 tokens at startup → full body <5k tokens on activation → resources on demand), so portability does not cost token efficiency.

## 4. Best Practices Discovered
- **Author skills to the lowest common denominator when cross-agent reuse is intended.** Keep the `name`/`description` block agentskills.io-clean (lowercase-hyphen `name` matching the dir; trigger-focused `description`). Put Hermes-specific wiring (toolset fallbacks, config prompts) in `metadata.hermes.*` so non-Hermes clients ignore it.
- **Treat the Skills Hub as a vetting source, not an auto-load.** Hub skills are opt-in by design — install explicitly, don't enable the whole catalog (token + attack-surface cost).
- **Prefer `hermes skills install <url>` over hand-copying external skill repos.** Single command, lands in the canonical `~/.hermes/skills/` registry, becomes a slash command automatically.
- **Standardize `description` as "what + when"** (the agentskills.io spec's explicit guidance) so the skill both triggers correctly in Hermes and is discoverable by other clients' matchers.
- **Use `license` + `compatibility` fields** when publishing a skill externally (agentskills.io optional fields) — signals reuse terms and environment needs to downstream consumers.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:**
  - Two internal skills live in the registry: `alfred-north-star` (global, symlinked from `project-zero/.hermes/skills/`) and `alfred-self-improvement` (project-local dir + a stray loose `.md` in the global registry).
  - `knowledge/` holds ~28 Markdown files; the classification ladder says reusable *procedure* should graduate Knowledge → Skill.
  - All skill authoring is **manual `write_file`**; we have never used the Skills Hub, never installed from a URL, and never published a skill externally.
- **Gaps & Anti-Patterns:**
  1. **Our `alfred-*` skills are non-portable by default.** They carry no `license`, no `compatibility`, and the `name`/`description` may not satisfy the agentskills.io `name` regex (e.g. stray uppercase or underscores would break on other clients). They are effectively locked to Hermes and undocumented for external reuse.
  2. **We never consume the Hub.** Repeated procedural knowledge we hand-author (e.g. git workflows, packaging) may already exist as an official Hub skill we could `hermes skills install` in one command — wasted effort re-deriving it.
  3. **No external-ingest path.** When a useful agentskills.io skill appears elsewhere (community, another team's repo), we have no documented habit of `hermes skills install <url>`; we'd hand-copy or ignore it.
  4. **`alfred-self-improvement` dual representation** (loose `.md` + skill dir) persists from the prior runtime lesson and is now a portability hazard too — a stray top-level `.md` is not a valid agentskills.io skill dir.

## 6. Recommended Improvements
1. **Audit `alfred-*` skills for agentskills.io compliance** (name regex, description ≤1024 "what+when", add `license` + optional `compatibility`). Draft in `experiments/skills-portability-audit/`. This makes our governance skills reusable outside Hermes at zero token cost.
2. **Add a one-line governance rule to `project-zero-governance`:** "Before hand-authoring a procedural skill, check `hermes skills install official/...` / `/skills search`; prefer `hermes skills install <url>` over manual copy when ingesting external skills." Codifies Hub-as-source.
3. **Consolidate `alfred-self-improvement` to a single canonical skill dir** (per the prior runtime lesson) — also required for it to be a valid portable agentskills.io skill.
4. **(Optional, experimental) Publish one internal skill as a shareable URL** to validate the outbound path: author a clean `alfred-north-star`, serve its `SKILL.md` over a URL, and `hermes skills install <url> --name` it into a throwaway profile to confirm round-trip portability.

## 7. Risk Assessment
- **Prompt-cache invalidation cost:** installed skills only take effect in new sessions; `--now` invalidates the cache (more tokens next turn). Mitigation: batch skill installs, don't install mid-critical-turn.
- **Hub supply-chain surface:** installing from arbitrary URLs trusts the remote `SKILL.md`. Mitigation: only install from known/owned URLs; review the file before install (the URL is readable first).
- **Cross-client `metadata.hermes.*` is silently dropped** by non-Hermes clients — if a skill's *core* logic depends on Hermes-only wiring, it may no-op elsewhere. Mitigation: keep the body self-contained; use `metadata.hermes.*` only for optional enhancement.
- **`name` regex strictness:** a non-compliant `name` works in Hermes but breaks on other clients. Mitigation: enforce the lowercase-hyphen ≤64 rule at authoring time ( Improvement #1).

## 8. Next Learning Topic
Hermes **Skill Bundles** (packaging repeated `/skill-a /skill-b` stacks into a single bundle, referenced from the Skills System doc) — when Project Zero should promote its common multi-skill orchestration patterns (e.g. `alfred-self-improvement` + `hermes-agent` invocation chains) into a named bundle. Source: https://hermes-agent.nousresearch.com/docs/user-guide/features/skills

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
