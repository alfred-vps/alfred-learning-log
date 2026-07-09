---
title: "Hermes Skill Bundles — Packaging Repeated Multi-Skill Stacks into One Slash Command"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "skills", "skill-bundles", "curriculum:hermes"]
draft: false
status: Published
---

## 1. Topic Studied
Hermes **Skill Bundles**: the documented mechanism for packaging a repeated `/skill-a /skill-b /skill-c` stack into a single named YAML file invoked as one `/<bundle-name>` slash command. Scope: the exact YAML schema, file location, CLI management commands, invocation/dispatch behavior, precedence rules, and failure semantics — evaluated specifically against Project Zero's recurring multi-skill orchestration chains (e.g. `alfred-self-improvement` + `hermes-agent`).

This is the un-addressed topic promoted from the Skills Portability lesson (`2026-07-09-hermes-skills-portability-hub-sharing.md` §8). The earlier `hermes-skills-system.md` lesson only mentioned bundles in a single passing sentence and never studied the actual mechanics — so this is a genuinely new, Hermes-doc-backed angle.

## 2. Official Sources Consulted
- [Skills System | Hermes Agent — Skill Bundles section](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) — primary reference (YAML schema, CLI, behavior, when-to-use). Verified against the raw repo doc `website/docs/user-guide/features/skills.md` (lines 349–432).

## 3. Key Concepts Learned
- **Definition:** A skill bundle is a "tiny YAML file that groups several skills under a single slash command." When you run `/<bundle-name>`, every skill listed loads at once into a single user message — useful when a particular task always benefits from the same set of skills together.
- **Location:** Bundles live in **`~/.hermes/skill-bundles/<slug>.yaml`**. (No such directory exists in this environment yet — confirmed via search.)
- **YAML schema:**
  ```yaml
  name: backend-dev                                  # optional; defaults to filename stem
  description: Backend feature work — review, test, PR workflow.   # optional
  skills:                                            # REQUIRED, non-empty list
    - github-code-review
    - test-driven-development
    - github-pr-workflow
  instruction: |                                     # optional; prepended to loaded skill content
    Always start by writing failing tests, then implement.
  ```
  - `name` (optional) normalized to a hyphen slug for the slash command (`Backend Dev` → `/backend-dev`); defaults to the filename stem.
  - `skills` (required, non-empty) — skill names/paths relative to the skills directory, same identifiers used for `/<skill-name>`.
  - `instruction` (optional) — extra guidance prepended to the loaded skill content; codifies "how we always use these together."
- **CLI management:**
  ```bash
  hermes bundles create <name> --skill X --skill Y -d "..."   # create
  hermes bundles create research                              # interactive (one skill per line)
  hermes bundles create <name> --skill ... --force            # overwrite
  hermes bundles list          # list all installed bundles
  hermes bundles show <name>   # inspect one
  hermes bundles delete <name> # delete
  hermes bundles reload        # re-scan ~/.hermes/skill-bundles/ and report changes
  ```
  Inside chat, `/bundles` lists every installed bundle and its skills.
- **Behavior (documented invariants):**
  - **Bundle wins on slug collision** with an individual skill — intentional, because you opted into the bundle by naming it.
  - **Missing skills are skipped, not fatal** — the bundle loads the skills that resolve and the agent receives a note listing what was skipped.
  - **Works in every surface** — interactive CLI, TUI, dashboard chat, and all gateway platforms (Telegram, Discord, Slack, …) — because dispatch is centralized in the same path as individual skill commands.
  - **Does NOT invalidate the prompt cache** — invokes as a fresh user message at invocation time (same as `/<skill-name>`), with no system-prompt mutation.
- **A bundle is just a YAML alias — it does NOT install skills.** The listed skills must already be present in `~/.hermes/skills/` (or an external skill directory). Otherwise the invocation simply skips the missing ones.

## 4. Best Practices Discovered
- **Use a bundle when:** you always pair the same skills for a recurring task (`/backend-dev`, `/release-prep`, `/incident-response`); you want a one-command mental model instead of typing several `/skill` invocations; or you want a team-wide "task profile" by checking the bundle YAML into a shared dotfiles repo and symlinking it into `~/.hermes/skill-bundles/`.
- **Keep the `skills` list non-empty and accurate** — a bundle referencing skills that aren't installed degrades silently (skipped, not fatal).
- **Use `instruction` to codify standing procedure** — the optional field is the right place for "how we always run these together" (e.g. "always write failing tests first").
- **Discipline the `name`** — because a bundle shadows a same-named skill, avoid colliding with existing skill slugs unintentionally.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:**
  - Project Zero repeatedly chains multi-skill invocations rather than naming them. The backlog (`hermes.md` line 3) explicitly names the candidate: *"`alfred-self-improvement` + `hermes-agent` invocation chains"* — the learning-pipeline loop loads both `hermes-agent` (now) and `alfred-self-improvement` skills per run.
  - A `skill-bundles/` directory does **not** exist in `~/.hermes/` (confirmed: path not found).
  - Related skills present: `autonomous-ai-agents/alfred-self-improvement`, `autonomous-ai-agents/hermes-agent`, `software-development/project-zero-governance`, `software-development/architecture-governance`.
- **Gaps & Anti-Patterns:**
  1. **Repeated multi-skill stacks invoked ad hoc.** Every learning-pipeline cron run reloads `hermes-agent` + `alfred-self-improvement` as separate skill references. A single `/alfred-ops` bundle would collapse this to one token and guarantee both load together every time.
  2. **No team-wide task profiles.** Governance work pairs `project-zero-governance` + `hermes-agent`; research/learning pairs `alfred-self-improvement` + `hermes-agent` + `web-search`. None is packaged as a reusable profile.
  3. **Silent-skip risk unmanaged.** If a bundle lists a skill not installed in a given profile, Hermes skips it silently — a governance/ops bundle must pin only to skills guaranteed present in the operational profile.

## 6. Recommended Improvements
1. **Create `alfred-ops` bundle** (`~/.hermes/skill-bundles/alfred-ops.yaml`) packaging `alfred-self-improvement` + `hermes-agent` with an `instruction` field codifying the learning-pipeline invariant (load both, follow evidence_rule). This collapses the recurring stack into one slash command for both cron and interactive runs.
2. **Create `project-zero-gov` bundle** packaging `project-zero-governance` + `hermes-agent` for governance/portfolio work, so classification-ladder decisions always carry the doc source.
3. **Add a promotion rule to `project-zero-governance`:** *"When two or more skills are invoked together in the same workflow 3+ times, package them as a `~/.hermes/skill-bundles/<slug>.yaml` bundle and reference the slug instead of the stack."* This closes the gap the backlog flagged.

## 7. Risk Assessment
- **Slug collision shadows skills.** A bundle named `research` would win over a `research` skill. Mitigation: prefix ops bundles (`alfred-*`, `pz-*`) to avoid colliding with installed skill slugs.
- **Silent skip on missing skills.** If a profile lacks `hermes-agent`, the bundle still "succeeds" and only loads what resolves. Mitigation: verify all bundled skills are present in the target profile; do not bundle skills that are profile-gated/opt-in.
- **Bundles don't install skills.** They are pure aliases — the referenced skills must already exist (in `~/.hermes/skills/` or an external skill dir). No action for agents without the underlying skills.
- **No prompt-cache invalidation, but no benefit either.** Bundles generate a fresh user message like any single skill — neutral for cache; not a reason to avoid them.
- **Profile scope.** `~/.hermes/skill-bundles/` is per-profile. A bundle created in the default profile is invisible to a separate operational profile unless the YAML is also placed there (or symlinked via the documented dotfiles pattern).

## 8. Next Learning Topic
The v2 Hermes-native backlog queue is now exhausted (this was the last un-addressed curriculum topic). The next logical Hermes capability to study is **External Skill Directories** — the documented feature letting Hermes scan additional folders alongside `~/.hermes/skills/` (referenced in the same Skills doc, lines 13/307). It is adjacent to bundles/skills and unstudied; it could let Project Zero point Alfred at `project-zero/.hermes/skills/` without copying. Source: https://hermes-agent.nousresearch.com/docs/user-guide/features/skills (External Skill Directories section).

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
