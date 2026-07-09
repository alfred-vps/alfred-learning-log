---
title: "Self-Improvement (SI) Loop — End-to-End Walkthrough"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "si-loop", "closed-loop", "runbook"]
draft: false
status: Published
---

# Self-Improvement (SI) Loop — End-to-End Walkthrough

How a single Hermes capability lesson travels from a scheduled cron tick to
durable, acknowledged knowledge in the Obsidian vault (and the public learning
log). Written 2026-07-09 as runbook knowledge for future sessions.

## Phase 0 — Trigger (the cron fires)

- **Job:** `Alfred Self-Improvement Loop` (`c4f43a29f185`)
- **Schedule:** `0,30 * * * *` → every hour at :00 and :30
- **Model:** `tencent/hy3:free` (OpenRouter)
- **Toolsets:** `web`, `file`, `search`, `terminal`
- **Workdir:** `/home/hermes/projects/project-zero` (isolated session;
  `AGENTS.md`/`PROJECT.md` injected)

When the scheduler hits a tick, it spins up a **fresh, isolated Hermes session**
for this job. No memory of the previous run — it re-reads everything from disk
each time. This is why the loop is stateful *via files*, not via session memory.

## Phase 1 — Topic Selection (Step 1)

The loop reads `learning/limitations_backlog.md` and picks the next topic **not
yet marked `[Addressed]`**.

- If the backlog is exhausted of Hermes-doc-backed topics → it outputs
  **`[SILENT]`** and stops. No file is written, nothing publishes. (Current
  state: the v2 queue is `[Addressed]`-exhausted, so most ticks currently
  `[SILENT]`.)
- It must articulate *why* the topic improves Alfred.

**Critical rule:** every topic must trace to **documented Hermes behavior**
(official docs, skills, config, or observed tool behavior). No generic AI/SE
lessons.

## Phase 2 — Deduplication (Step 2)

Before researching, it runs `search_files` over
`/home/hermes/docs/obsidian/learning/self-improvement/`.

- If the topic already has a lesson file → mark it `[Addressed]` in the backlog
  and pick another.
- This prevents re-studying the same capability across runs.

## Phase 3 — Research (Step 3)

Uses `web_search` + `web_extract` against
`https://hermes-agent.nousresearch.com/docs/` and the Hermes skill system.

- Preference order: **official docs > blogs**.
- Every finding records its **evidence source URL** (this URL is what later
  lands in `HERMES-KB.md`).

## Phase 4 — Draft Lesson (Step 4)

Writes the lesson to the vault using the capability-lesson-template shape:

```
/home/hermes/docs/obsidian/learning/self-improvement/YYYY-MM-DD-topic-name-hyphenated.md
```

Structure (from the template):

1. Topic Studied
2. Official Sources Consulted (the evidence URLs)
3. Key Concepts Learned
4. Best Practices Discovered
5. Comparison with Current Implementation (Arif/Alfred usage vs native)
6. Recommended Improvements
7. Risk Assessment
8. Next Learning Topic

**Frontmatter:** `title`, `date`, `tags`, `draft`. The SI loop sets
**`draft: false`** (it intends to publish), and `status: Published`.

**MANDATORY SELF-CHECK:** after writing, it runs `search_files` for the exact
filename. If the file is **not found**, the step FAILED — it must rewrite. It
never reports success without confirming the file exists on disk.

> This is the "honesty clause" guard: the loop proves the write happened via a
> filesystem check, not by reasoning.

## Phase 4.5 — Closed-Loop KB Append (added 2026-07-09)

After the lesson self-check passes, the loop appends **one structured line** to:

```
/home/hermes/docs/obsidian/knowledge/HERMES-KB.md
```

Schema:

```
- [YYYY-MM-DD] <lesson title> | evidence:<url> | lesson:<filename>
```

- **Deduplicated:** if an entry with the same `lesson:<filename>` already
  exists, it skips (no dupes).
- **Private:** `knowledge/` is outside the publish whitelist in
  `convert-lessons-to-blog.py`, so this entry **never reaches the public blog**.
  It is the internal retrieval surface Alfred consults before Arif's tasks.

This is the step that converts the loop from write-only logging into a **closed
loop** — the lesson now feeds a consolidated index that the daily synthesis cron
later turns into skill/memory proposals.

## Phase 5 — Update the Backlog (Step 5)

Prepends the **next logical Hermes topic** to `limitations_backlog.md` (with
source URL), and marks the just-studied topic `[Addressed]`.

- This is the *cross-run memory*: the next tick reads this file to know what's
  left.
- State persists entirely in this file + the lesson directory.

## Phase 6 — Publish (Step 6)

Runs via the `terminal` tool:

```bash
bash /home/hermes/projects/project-zero/scripts/lifecycle/publish-noagent.sh
```

What that script does (mechanically, no LLM):

1. `cd /home/hermes/projects/alfred-learning-log`
2. `python3 convert-lessons-to-blog.py --apply` — mirrors vault lessons → Hugo
   `content/operational/` (the publish whitelist: `self-improvement/ →
   operational`, `summary/ → summary`, `north-star/ → strategic`).
3. If `git status -s` is empty → prints `[HH:MM] no changes — nothing to
   publish`, exits 0.
4. Else `git add . && git commit -m "Auto-publish (no_agent): HH:MM" &&
   git push origin master` → prints `[HH:MM] published`.

The loop **captures the script's stdout + exit code** as the source of truth for
"published".

## Phase 7 — Final Delivery (Step 8)

The loop outputs the lesson summary to **Telegram** (deliver target: `telegram`
/ Home), stating the evidence source, and appends the publish result:

- exit 0 + `"published"` → *"Published: <filename> pushed to alfred-learning-log."*
- `"no changes"` → *"Lesson written; already mirrored, no new push."*
- exit ≠ 0 → *"PUBLISH FAILED — lesson written but git push did not succeed.
  Manual check needed."* — it does **not** claim success.

**Honesty clause:** it never says "verified"/"build passed"/"published" from
reasoning alone — only from the script's real git exit code.

## Phase 8 — Acknowledged as Knowledge in Obsidian

At this point the lesson is **durably acknowledged** in two Obsidian locations:

| Location | Role | Public? |
|----------|------|---------|
| `learning/self-improvement/YYYY-MM-DD-...md` | Full lesson (source of truth, evidence-rich) | Mirrored → blog `content/operational/` |
| `knowledge/HERMES-KB.md` | One-line consolidated index entry | **Private** (not in whitelist) |

The lesson file is the **authoritative knowledge artifact** — human-readable,
version-controlled, cited. The KB line is the **retrieval pointer** Alfred uses
on the next task.

## The Closed Loop (what happens next, daily)

The `Hermes-KB Daily Synthesis` cron (`8a2a47cb2f3a`, `30 6 * * *`) runs next
morning:

1. Hashes `HERMES-KB.md`. If unchanged since last snapshot → `[SILENT]`, nothing
   delivered.
2. If changed → writes `knowledge/proposed-synthesis-<date>.md` and drafts
   **propose-only** skill/memory diffs.
3. Delivers to Arif. **Does NOT auto-apply.** Waits for `[APPROVED]`.

That `[APPROVED]` is what finally turns accumulated knowledge into **changed
Alfred behavior** — a skill patch, a saved memory fact, or a new procedure.
Until then, the knowledge sits acknowledged in Obsidian, ready to be consulted.

## One-line summary of the pipeline

```
cron tick (:00/:30)
  -> read backlog, pick topic
  -> dedup check
  -> research Hermes docs (capture URLs)
  -> write lesson to learning/self-improvement/  [self-check via search_files]
  -> append 1 line to knowledge/HERMES-KB.md     [private, closed-loop]
  -> mark backlog [Addressed]
  -> publish-noagent.sh (mirror + git push)       [real exit code = truth]
  -> Telegram summary
        | next morning
  daily synthesis cron -> propose-only diff -> [APPROVED] -> behavior changes
```
