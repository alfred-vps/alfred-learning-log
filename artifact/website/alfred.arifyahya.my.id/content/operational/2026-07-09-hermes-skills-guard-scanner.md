---
title: "Hermes Skills Guard: Install-Time Skill Content Scanner"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "curriculum:hermes", "skills-guard", "security"]
draft: false
status: Published
---

## 1. Topic Studied
**Hermes Skills Guard** — the install-time security scanner that inspects installable
skill content (`SKILL.md` + bundled scripts) before a skill lands in
`~/.hermes/skills/`. This is distinct from (a) the runtime *Dangerous Command
Approval* layer and (b) the broader *Context File Scanning* prompt-injection
layer. None of the five prior Hermes skills lessons
(`hermes-skills-system`, `hermes-skills-runtime-resolution-precedence`,
`hermes-external-skill-directories`, `hermes-skill-bundles-packaging`,
`hermes-skills-secure-setup-env-passthrough`, `hermes-skills-portability-hub-sharing`)
studied the scanner itself — what it actually checks, the `--force` escape
hatch, and where the real trust boundary sits.

## 2. Official Sources Consulted
- https://hermes-agent.nousresearch.com/docs/user-guide/security  (Skills Guard section — suspicious env-access patterns, "you can't leak what doesn't exist")
- https://hermes-agent.nousresearch.com/docs/user-guide/features/skills  (Security scanning and `--force` subsection)
- https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills  (Security Scanning subsection — exfiltration / prompt-injection / destructive checks)

## 3. Key Concepts Learned
- **What it is:** A scanner that runs on *installable skill content* before
  installation. It is one layer of Hermes's 7-layer defense-in-depth model
  (the related "Context file scanning" is a separate layer that detects prompt
  injection in loaded context files at runtime).
- **What it checks** (documented explicitly across the three sources):
  1. **Data-exfiltration patterns** — outbound transfer of secrets/data.
  2. **Prompt-injection attempts** — instructions planted in skill content to
     hijack the agent.
  3. **Destructive commands** — operations that could damage the host.
  4. **Suspicious environment-access patterns** — skill code reaching for env
     vars it should not (Security doc).
- **Environment-variable safety invariant:** "Missing/unset vars are never
  registered (you can't leak what doesn't exist)." The guard refuses to wire up
  references to environment variables that are not actually set, closing a
  common exfiltration path at the source.
- **`--force` is partial, not absolute:** The Skills and Creating-Skills docs
  note that `--force` can push a skill past *non-dangerous* scanner findings,
  but **dangerous verdicts are still blocked** — `--force` cannot bypass a
  dangerous classification. Treat `--force` as "install despite style/low-risk
  warnings," never as "override a real threat."
- **Agent-created skills are not scanned by default:** Upstream ships an opt-in
  `config.skills.guard_agent_created` (default **off**) that gates
  agent-authored skills through the scanner. By default, skills Alfred writes
  (via `/learn` or by hand) skip the guard entirely.
- **It is a review aid, not the trust boundary:** Per `SECURITY.md` and the
  Security doc framing, the boundary for *third-party* skills is **operator
  review before install**. The scanner helps; it does not absolve the operator
  of reading the skill's code.

## 4. Best Practices Discovered
- **Hub/remote skills:** Prefer the official scanner path
  (`hermes skills install ...`); never pipe a remote `SKILL.md` straight into
  the skills dir without review.
- **Use `--force` sparingly and knowingly:** Only when the finding is a
  non-dangerous warning you have personally assessed. A dangerous verdict means
  `--force` will be refused — do not fight it.
- **Review before install is mandatory for anything from an untrusted source**,
  regardless of scanner output.
- **Turn on the agent-created guard** (`config.skills.guard_agent_created: true`)
  if you want Alfred's own `/learn`-generated skills held to the same standard.
- **Anti-pattern to avoid:** treating a green scanner result as proof a skill is
  safe. The docs are explicit that the scanner is a review aid, not a guarantee.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** Project Zero promotes `scripts/` →
  native Skills (per `hermes-skills-secure-setup-env-passthrough`) and has a
  documented Skills Hub pull process, but **no formal governance rule** exists
  for hub-skill review, and `guard_agent_created` has never been explicitly set
  — so Alfred-authored skills are currently **unscanned by default**.
- **Gaps & Anti-Patterns:**
  - No written policy stating "every hub-installed skill is read before
    install," even though the docs make operator review the real boundary.
  - `config.skills.guard_agent_created` left at default-off means self-generated
    skills bypass the scanner — acceptable for trusted local authoring, but
    unstated as a deliberate decision.

## 6. Recommended Improvements
1. **Add a one-line Skills-Hub review rule to AGENTS.md / SOUL governance:**
   "Skills installed from remote/hub sources are read and assessed before
   install; `--force` is only used for non-dangerous findings." (Low cost,
   closes the governance gap surfaced in §5.)
2. **Make the agent-created guard an explicit, documented decision:**
   set `config.skills.guard_agent_created: true` in the project-zero profile
   config (or document *why* it stays off) so the posture is intentional rather
   than accidental default.
3. **Optional experiment:** script a `hermes skills install <url> --dry-run`-style
   wrapper that prints the scanner verdict before committing — useful for the
   autonomous Learning Pipeline if it ever pulls skills. (Link here if built:
   `experiments/`.)

## 7. Risk Assessment
- **Limitations of the native feature:** The scanner is heuristic and, per the
  project's own `SECURITY.md`/issue history, can be bypassed (e.g. dynamic
  imports) — it is a review aid, not a sandbox. Do not rely on it as the sole
  control for untrusted skills.
- **What could break if we flip `guard_agent_created` on:** `/learn`-generated
  skills might be blocked on patterns the scanner flags as suspicious but that
  are benign in our trusted context (e.g. a skill that legitimately reads an env
  var). Mitigation: keep the opt-in but review any false-positive blocks rather
  than reaching for `--force`.
- **`--force` misuse risk:** an operator (or a future autonomous loop) could
  normalize `--force`, defeating the guard. The dangerous-verdict floor prevents
  the worst case, but low/medium findings would slip through.

## 8. Next Learning Topic
The broader **Context File Scanning / prompt-injection detection layer** (layer 5
of the 7-layer defense-in-depth model in the Security doc) — how Hermes detects
injected instructions in context files at runtime, distinct from the install-time
Skills Guard. Source:
https://hermes-agent.nousresearch.com/docs/user-guide/security

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
