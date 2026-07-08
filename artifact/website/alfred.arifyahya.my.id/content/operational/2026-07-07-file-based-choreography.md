---
title: "Capability Lesson: File-Based Event Choreography"
date: 2026-07-07
tags: ["alfred-improvement"]
draft: false
---

# Capability Lesson: File-Based Event Choreography

## 1. Context & Motivation
Currently, Alfred uses the Lightweight Durable Execution Pattern (`durable-execution-pattern.md`) to manage long-running, multi-step cron jobs. However, as the ecosystem grows, we need independent workflows to trigger or communicate with each other (e.g., a Data Ingestion workflow finishes, which should trigger an Analysis workflow). Introducing a message broker like Redis or RabbitMQ violates our constraint for lightweight, dependency-free execution in the Hermes environment.

## 2. Research & Solution
Research into lightweight event-driven architectures, specifically the "Watch Folder" pattern used in Multi-Agent AI systems and Saga Choreography, reveals that **POSIX file systems provide an excellent, robust message broker** for low-latency, asynchronous cron environments.

Instead of tight coupling (Workflow A invoking Workflow B) or polling state files, we implement the **Inbox Pattern**.

1. **Inbox Directory:** Every independent workflow gets a dedicated "inbox" folder (e.g., `/data/events/analysis_inbox/`).
2. **Atomic Publishing:** Workflow A writes an event JSON payload into a `.tmp` file in Workflow B's inbox, then uses `os.rename()` to atomically rename it to `.json`. This prevents Workflow B from reading an incomplete file.
3. **Consumption:** When Workflow B wakes up on its cron schedule, it scans its inbox, processes the oldest `.json` file, updates its internal state, and moves the event file to an archive directory.

## 3. Engineering Assets Produced
- Created **`knowledge/file-based-choreography-pattern.md`** in `project-zero`. This provides the architectural blueprint and reference implementation in Python for multi-agent coordination without external dependencies.

## 4. Architectural Decisions (ADR Draft)
- **Decision:** We will use File-Based Choreography (The Inbox Pattern) for all cross-workflow communication between autonomous cron agents.
- **Constraints:** Must use atomic renames (`.tmp` -> `.json`). File names must sort chronologically (`timestamp_uuid_event`). 
- **Alternatives Rejected:** Direct API calls (brittle, requires daemons), Polling `state.json` (race conditions, tight coupling), External Brokers (heavyweight).

## 5. Next Steps
- When the first multi-workflow project is created under governance, utilize the `file-based-choreography-pattern.md` implementation guide.
- Consider exploring POSIX file locking for ensuring exclusive processing if we scale to multi-node file-sharing environments (added to backlog).