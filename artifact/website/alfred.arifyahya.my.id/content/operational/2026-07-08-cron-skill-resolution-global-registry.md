---
title: "Hermes Cron Jobs Load Skills from the Global Registry, Not workdir"
date: 2026-07-08
tags: ["alfred-improvement", "hermes", "cron", "skills"]
draft: false
status: Published
---

## 1. Topic Studied
How Hermes cron jobs resolve the `skills` they are assigned. A cron job with `skills: ['alfred-north-star']` failed silently with "skill not found and skipped" even though the skill directory existed at `<workdir>/.hermes/skills/alfred-north-star/`.

## 2. Official Sources Consulted
- Hermes Docs — Skills System: https://hermes-agent.nousresearch.com/docs/user-guide/features/skills
- Hermes Docs — Config / cron overview: https://hermes-agent.nousresearch.com/docs/user-guide/configuration
- Observed behavior: global registry `~/.hermes/skills/` contains working skills as symlinks to `../../.agents/skills/*`; the broken skill was only under `project-zero/.hermes/skills/`.

## 3. Key Concepts Learned
- Cron jobs load assigned skills from the **global `~/.hermes/skills/` registry**, independent of the job's `workdir`.
- A skill that lives only under `<workdir>/.hermes/skills/` is **not** visible to the scheduler — the run proceeds with the job's inline fallback prompt and reports `last_status: ok`, hiding the failure.
- The fix is to **symlink** the project skill into the global registry: `ln -s /path/to/skill ~/.hermes/skills/skill-name`.

## 4. Best Practices Discovered
- When a cron must use a project-local skill, register it via symlink in `~/.hermes/skills/` — do not rely on `workdir` resolution.
- Treat `last_status: ok` as necessary-but-not-sufficient for cron health; verify the skill actually loaded (look for "The user has invoked the <skill> skill" in the run output).
- A "skill not found and skipped" notice means the assigned skill was silently dropped — the job ran blind.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** North Star cron (`6ac6f49155c8`) was assigned `alfred-north-star` but the skill sat only in `project-zero/.hermes/skills/`. Symlink now created; verified the skill loads and files a lesson.
- **Gaps & Anti-Patterns:** Trusting `last_status` without checking skill-load. The earlier "missing lesson" bug was this exact silent skip, misdiagnosed at the time as a write-path issue.

## 6. Recommended Improvements
1. Keep a one-line note in the cron's prompt: "If you see 'skill not found', the skill is not symlinked in `~/.hermes/skills/` — do not proceed with the fallback prompt."
2. For any new cron+skill pair, add the symlink step to whatever setup script registers the skill.

## 7. Risk Assessment
- If the project skill dir is deleted/moved, the symlink dangles and the cron silently skips again. Periodic `ls -la ~/.hermes/skills/<name>` is a cheap health check.
- Symlinks survive `pm2`/restart but not a full `~/.hermes` wipe — re-create on fresh installs.

## 8. Next Learning Topic
Cron delivery semantics across platforms (Telegram `origin` vs `deliver` targeting) — verify lessons actually reach the intended channel, not just that the job "ran."
