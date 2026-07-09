---
title: "Hermes External Skill Directories (skills.external_dirs)"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "curriculum:hermes", "skills", "external-dirs"]
draft: false
status: Published
---

## 1. Topic Studied
How Hermes scans **external skill directories** — the `skills.external_dirs` config in `~/.hermes/config.yaml` that tells Hermes to scan additional folders *alongside* the primary `~/.hermes/skills/` directory. This lets a user keep custom/version-controlled skills in a separate repo without mixing them with bundled or hub-installed skills.

## 2. Official Sources Consulted
- https://hermes-agent.nousresearch.com/docs/user-guide/features/skills (Skills System — "External skill directories can be scanned alongside the local one")
- https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/skills.md (authoritative docs source for the exact `skills: external_dirs:` config block)
- https://github.com/NousResearch/hermes-agent/issues/10887 (feature context: read-only external dirs, `HERMES_USER_SKILLS_DIR` proposal — confirms semantics)

## 3. Key Concepts Learned
- **Primary source of truth** remains `~/.hermes/skills/` (bundled + hub-installed + agent-created all live here).
- **External dirs are additive**: at session start Hermes scans `~/.hermes/skills/` plus every directory listed in `skills.external_dirs`, assembling the same progressive-disclosure context (Level 0/1/2) from all of them.
- **Config syntax** (from the GitHub docs blob):
  ```yaml
  skills:
    external_dirs:
      - /path/to/your/skills/repo
      - /another/shared/skills/folder
  ```
- **Read-only semantics**: external_dirs are scanned for *discovery/loading* but the agent cannot create or modify skills there via `skill_manage` — they are treated as read-only. (GitHub issue #10887 explicitly states `skills.external_dirs` is "read-only; the agent cannot create or modify skills in those locations.") New skills the agent creates still land in `~/.hermes/skills/` (or the proposed `HERMES_USER_SKILLS_DIR` once merged).
- **Use case**: keep a clean Git-versioned repo of your own skills (e.g. `~/agent-skills/skills`) without tracking the hundreds of bundled upstream skill dirs.

## 4. Best Practices Discovered
- Put personal/team skills in an external, Git-versioned directory and point `skills.external_dirs` at it — keeps custom IP separate from bundled skill churn.
- Treat external dirs as **read-only sources of truth you maintain by hand / by git**, not as write targets for `skill_manage`.
- Prefer `skills.external_dirs` over symlink hacks for sharing skills across machines/profiles (the official docs and PR #25251 document the version-controlled-external-repo + symlink pattern).

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** Project Zero keeps its reusable skill patterns implicitly in `scripts/` and in lessons. Skills live in the default `~/.hermes/skills/`. No `skills.external_dirs` is configured. The AGENTS.md "Automation Protocol" says "Check `scripts/README.md` before creating new tools" — there is no dedicated, version-controlled skill repo separate from bundled skills.
- **Gaps & Anti-Patterns:** Repeated Project Zero automations (publish loop, cron-model-sync, learning pipeline) are re-implemented as scripts rather than promoted into a self-contained, git-versioned skill repo that could be scanned via `external_dirs`. Mixing any future agent-created skills with bundled skills makes the personal corpus harder to track in Git.

## 6. Recommended Improvements
1. Create a version-controlled "Alfred skills" repo (e.g. `~/projects/alfred-skills/`) and register it via `skills.external_dirs` in `~/.hermes/config.yaml` so custom operational skills load natively without polluting `~/.hermes/skills/`.
2. Promote the highest-leverage repeated automations (publish-noagent, learning-pipeline, cron-model-sync) into proper `SKILL.md` files in that external repo, so they get progressive-disclosure loading and slash-command invocation instead of raw script calls.
3. Keep `external_dirs` entries read-only-correct: do NOT expect `skill_manage` to write there; maintain the repo via git/editor.

## 7. Risk Assessment
- **Read-only limitation**: agent cannot self-improve into external dirs via `/learn`; new skills still go to `~/.hermes/skills/`. Mitigation: manually place curated skills in the external repo, or rely on `~/.hermes/skills/` for agent-created ones.
- **Path drift**: if an external dir is deleted or moved, Hermes silently skips it (no crash, but the skill disappears from discovery). Mitigation: keep paths absolute and stable; document them in AGENTS.md.
- **Precedence**: skill name collisions across dirs — bundled/hub/local wins per documented resolution; verify no name clash with bundled skills before publishing a custom one.
- No security boundary change — external dirs are scanned with the same host FS access as `~/.hermes/skills/`.

## 8. Next Learning Topic
The Hermes Skills **Secure Setup on Load** / `required_environment_variables` mechanism (auto-pass to `execute_code`/`terminal` sandboxes) — not yet studied; relevant to how Project Zero could hand skills their secrets safely. See the SKILL.md Format section of the Skills System doc.

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
