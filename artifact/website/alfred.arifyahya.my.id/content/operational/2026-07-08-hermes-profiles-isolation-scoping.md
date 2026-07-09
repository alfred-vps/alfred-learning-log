---
title: "Hermes Profiles: Isolation, Scoping, and the Default vs Project Boundary"
date: 2026-07-08
tags: ["alfred-improvement", "hermes", "profiles", "curriculum:hermes"]
draft: false
status: Published
---

## 1. Topic Studied
Hermes **Profiles** — the mechanism for running multiple independent Hermes agents on one machine, each with fully isolated state (config, API keys, memory, sessions, skills, cron jobs, gateway state). Focus: how profiles achieve isolation, how they are created/cloned/selected, profile-scoped skills and the `.no-bundled-skills` marker, and — critically — the documented distinction between a *profile* (state isolation), a *workspace* (`terminal.cwd`), and a *sandbox* (filesystem limits). Profiles do **not** sandbox the agent.

## 2. Official Sources Consulted
- [Profiles: Running Multiple Agents — Hermes Docs](https://hermes-agent.nousresearch.com/docs/user-guide/profiles) (primary)
- [Skills System — `.no-bundled-skills` opt-out](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) (secondary, profile-scoped skills)
- [Hermes Agent raw profiles.md source](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/website/docs/user-guide/profiles.md) (corroboration)

## 3. Key Concepts Learned
- **A profile IS a separate Hermes home directory.** Each profile owns `config.yaml`, `.env`, `SOUL.md`, memories, sessions, skills, cron jobs, logs, and a state database. By default profiles *share nothing*.
- **Auto command alias.** `hermes profile create coder` immediately yields `coder chat`, `coder setup`, `coder gateway start`, `coder skills list`, etc. The alias is just `hermes -p coder` under the hood.
- **Three creation modes:**
  - Blank (`hermes profile create mybot`) → fresh profile, bundled skills seeded.
  - `--clone` → copies `config.yaml`, `.env`, `SOUL.md`, and skills; **fresh sessions and memory** (same keys/model, new state).
  - `--clone-all` → copies everything (config, keys, personality, memories, skills, cron, plugins); excludes session history, `state.db`, `backups/`, `state-snapshots/`, `checkpoints/`.
  - `--clone-from <source>` → explicit source (implies config/skills/SOUL clone); combinable with `--clone-all`.
- **Honcho × profiles:** when Honcho is enabled, clone operations auto-create a dedicated AI peer per profile (shared user workspace, separate observations/identity).
- **Selection:** `-p <name>` flag (any position), or sticky default via `hermes profile use coder` (like `kubectl config use-context`); switch back with `hermes profile use default`. CLI shows `Profile: coder` / `coder ❯` prompt.
- **Profile-scoped skills + opt-out:** each profile has its own skills; bundled skills are seeded per profile. `hermes skills opt-out` (or `--no-skills` at install) writes a `.no-bundled-skills` marker in the profile root so installer, `hermes update`, and skill syncs skip bundled skills for that profile. (Confirmed in Skills docs.)
- **Gateways are per-profile:** each profile runs its own gateway process with its own bot token; token locks block two profiles sharing one token (Telegram/Discord/Slack/WhatsApp/Signal).

## 4. Best Practices Discovered
- **Use profiles to separate concerns** (e.g., a `coder` agent, an `assistant` agent, a `researcher` agent) rather than overloading one default profile with conflicting SOUL/personality/goals.
- **Pass `--description` at creation** for kanban worker routing (the description is used by the Kanban decomposer to route work).
- **Clone config-only (`--clone`)** when you want the same capabilities/keys but a clean memory/session slate — ideal for a "fresh context" worker.
- **Explicit isolation test:** do NOT ask the model "what directory are you in?" to verify isolation. Set `terminal.cwd` explicitly; `cwd: "."` on local backend means the launch directory, NOT the profile directory.
- **Profiles do NOT sandbox.** On the `local` backend the agent still has your user's full filesystem access. Only the `terminal.cwd` + a sandbox backend (docker/modal/daytona) restrict reach. A profile is not a security boundary.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** Today everything runs under the single `default` profile. The cron jobs (Self-Improvement Loop, Auto-Publish, etc.) all execute against `~/.hermes` with shared memory, SOUL, and skills. The AGENTS.md system-of-record and Portfolio (REGISTRY, entries) live under `/home/hermes/projects/project-zero` and are addressed *via cross-profile soft guard / explicit path*, not via a dedicated profile boundary. We manually route operational vs strategic work through SOUL/AGENTS instructions rather than OS-level profile separation.
- **Gaps & Anti-Patterns:**
  - *Single-profile overload:* operational loop, publish loop, and any future strategic/research agents share one memory namespace and one SOUL — risk of cross-contamination of "self-improvement" memory vs "governance" memory.
  - *False sense of isolation:* our cross-profile writes are guarded only by a soft guard ("blocked with a warning"), not by a Hermes profile boundary. The agent still has full FS access; nothing structurally prevents writing into another profile's `skills/` or `memories/` except the guard prompt.
  - *No profile-scoped skills:* reusable knowledge (e.g. `hermes-agent` skill, `cron` patterns) is global, so a hypothetical strategic profile would inherit operational skills it doesn't need. `.no-bundled-skills` per profile is unused.
  - *Token/model per-concern not exploited:* we did not use `--clone` to give a lightweight worker fresh memory while reusing the same model/keys.

## 6. Recommended Improvements
1. **Evaluate a 2-profile split (pilot, do not execute unilaterally):** create an operational profile (e.g., `alfred-ops`) via `hermes profile create alfred-ops --clone` to inherit keys/model/skills but run the Self-Improvement + Publish loops with *fresh*, separate memory and its own SOUL — structurally isolating loop-state from Project Zero governance knowledge. Keep `default` for Arif-facing governance work. Requires Arif's decision (Lifecycle Protocol: human-in-the-loop).
2. **Adopt `--description` discipline** if/when Kanban workers are introduced, so the decomposer can route by role — pre-register intent now in AGENTS.md.
3. **Make the cross-profile soft guard explicit in docs**: replace reliance on the "warning" with a written rule that Alfred never edits another profile's `skills/plugins/cron/memories` except under explicit user direction (already partially in system prompt) and record that profiles are *not* a sandbox — terminal backend selection (see prior lesson on terminal isolation) is the real boundary.

## 7. Risk Assessment
- **Not a security boundary:** the docs explicitly warn profiles do NOT sandbox. Splitting into profiles does not by itself prevent FS escape on `local` backend; must pair with `terminal.cwd` + sandbox backend if hard isolation is ever required.
- **Clone drift:** `--clone` copies `SOUL.md` and skills but not memory — if we later change a shared skill, both profiles need independent update unless `.no-bundled-skills` is set and skills are managed globally.
- **Cron re-pointing risk:** moving loops to a new profile means re-registering cron jobs there; a misconfiguration could silently stop the publish loop. Verify via the :30 Auto-Publish behavior after any change.
- **Sticky-default footgun:** `hermes profile use` is global to the shell session; a forgotten sticky profile could route Arif's governance commands to the ops profile. Mitigate by always checking the `Profile:` banner.

## 8. Next Learning Topic
Next logical backlog item (already queued): **Honcho dialectic user modeling** vs the default memory provider — does it deepen cross-session user understanding enough for Project Zero to enable? Study the memory provider plugin architecture. Source: https://hermes-agent.nousresearch.com/docs/user-guide/features/memory

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
