---
title: "Automated Kanban Sweeps for Performance"
date: 2026-07-08
tags: ["alfred-improvement"]
draft: false
---

# Automated Kanban Sweeps for Performance

**Date:** 2026-07-08
**Topic:** Kanban Database Maintenance Automation
**Source:** Priority Actions (`AGENT-REVIEW.md`) and limitations backlog.

## Executive Summary
As Project Zero migrates toward Hermes Kanban as the primary multi-agent orchestration engine (replacing custom POSIX lock-based durable execution), the long-term health of `~/.hermes/kanban.db` becomes a critical operational requirement. Over time, completed tasks accumulating in the `done` status can degrade query performance and pollute human-facing dashboards. 

This lesson resolves the operational gap by implementing a dedicated maintenance workflow (a cron-ready Python script) that safely sweeps the database, moving stale `done` tasks to an `archived` status after a configurable retention period.

## Context & Limitation
- **Friction/Limitation:** How can we implement automated periodic sweeps of the Hermes Kanban `kanban.db` via a dedicated maintenance workflow to archive tasks left in the `done` state beyond a retention period, keeping the active board performant? (Identified in `limitations_backlog.md`).
- **Strategic Alignment:** The most recent biweekly improvement review (`AGENT-REVIEW.md`) explicitly demands migration away from custom durable execution loops in favor of Hermes Kanban. Preparing the infrastructure for long-term Kanban usage is a prerequisite for this migration.

## Research Findings
- **Hermes Kanban Architecture:** The Kanban system uses a SQLite database located at `~/.hermes/kanban.db` (for the default board). Every task and handoff is represented as a row within this database. 
- **Task Lifecycle:** The official documentation notes the following valid statuses: `triage | todo | ready | running | blocked | done | archived`.
- **Maintenance Gap:** While the internal dispatcher reclaims stale *running* tasks (e.g., workers that missed heartbeats), it does not automatically garbage-collect or archive tasks that have legitimately reached the `done` state. This is by design, allowing human review, but requires periodic cleanup to prevent unbound growth.
- **Evidence Quality:** Primary Docs (`hermes-agent.nousresearch.com`).

## Engineering Value Assessment (EVA)
- **Impact (0-40): 30.** Essential for long-term operational health as the volume of automated tasks increases post-migration. Prevents slow queries and dashboard clutter.
- **Evidence (0-20): 20.** Backed by direct documentation of the Kanban database schema and lifecycle states.
- **Actionability (0-20): 20.** Easily implemented using standard Python SQLite libraries interacting with the known database path.
- **Reusability (0-20): 20.** The script serves as a reusable maintenance asset that can be triggered via cron or other orchestration tools.
- **Total Score:** 90/100 -> **Implement Now**

## Solution & Implementation Details
A new maintenance script has been added to the Project Zero automation directory:
`/home/hermes/projects/project-zero/scripts/maintenance/kanban-archiver.py`

### Mechanism
The script connects directly to the SQLite backend (`~/.hermes/kanban.db`) and executes an idempotent bulk update. It identifies tasks where `status = 'done'` and the timestamp (either `updated_at` or `created_at`, dynamically determined based on exact schema) is older than a configurable retention period (default 7 days), and updates their status to `'archived'`.

### Configuration
The retention period is adjustable via the `KANBAN_ARCHIVE_DAYS` environment variable, allowing flexible scheduling depending on deployment size.

## Generated Assets
1. **Maintenance Script:** `scripts/maintenance/kanban-archiver.py` (Created in Project Zero).

## Recommendations & Next Steps
1. **Schedule Execution:** Configure a weekly cron job to execute `python3 /home/hermes/projects/project-zero/scripts/maintenance/kanban-archiver.py`.
2. **Backlog Updated:** Marked the archival question as addressed and appended a new advanced question regarding self-healing Kanban loops and external system reconciliation.