---
title: "Hermes Terminal Backends — Selecting Isolation for the Learning & Publish Loops"
date: 2026-07-08
tags: ["alfred-improvement", "hermes", "terminal", "backends", "cron", "isolation", "curriculum:hermes"]
draft: false
status: Published
---

## 1. Topic Studied
How Hermes' `terminal` tool can run commands across six backends (`local`, `docker`, `ssh`, `singularity`, `modal`, `daytona`), the documented persistence/isolation semantics of each, and which backend (if any) should host the Self-Improvement and Alfred Learning-Log publish cron loops for safer execution. Also: the native `safe` toolset as a read-only alternative directly relevant to terminal-less crons.

## 2. Official Sources Consulted
- Primary: Hermes Docs — Tools & Toolsets, "Terminal Backends": https://hermes-agent.nousresearch.com/docs/user-guide/features/tools
- Secondary: Toolsets Reference (lists `safe` composite, `code_execution`, `debugging`, platform presets): https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference
- Tertiary: Built-in Tools Reference (3 terminal tools: `terminal`, `process`, `read_terminal`): https://hermes-agent.nousresearch.com/docs/reference/tools-reference

## 3. Key Concepts Learned
- **Six backends, one tool.** The `terminal` tool executes in a configurable environment; the backend is set in `~/.hermes/config.yaml` under `terminal.backend` (default `local`).
- **`local`** — runs on the host machine; intended for development and trusted tasks.
- **`docker`** — isolated containers for security/reproducibility. Crucially documented as **one persistent container shared across the whole Hermes process**: Hermes runs a single long-lived container (`docker run -d ... sleep 2h`) and routes every `terminal`, `file`, and `execute_code` call through `docker exec` into it. Working-dir changes, installed packages, and files under `/workspace` carry over across calls, `/new`, `/reset`, and `delegate_task` subagents for the process lifetime. It "behaves like a persistent sandbox VM, not a fresh container per command." The `container_persistent` flag controls whether `/workspace` and `/root` survive across process restarts.
- **`ssh`** — remote server; documented use case is *sandboxing / keeping the agent away from its own code*.
- **`singularity`** — HPC containers; cluster computing, rootless.
- **`modal`** — cloud execution; serverless, scale.
- **`daytona`** — cloud sandbox workspace; persistent remote dev environments.
- **Base config** exposes `backend`, `cwd`, and `timeout` (seconds) keys.
- **Native `safe` toolset** exists: `safe` = `image_generate`, `vision_analyze`, `web_extract`, `web_search` (read-only research + media gen). No file writes, no terminal, no code exec. This is the closest native "research-only" bundle — directly comparable to the `web,file,search` toolsets the learning loops currently run with.

## 4. Best Practices Discovered
- Match backend to trust/isolation need: `local` for trusted dev, `docker`/`ssh` when you want the agent separated from host state, `modal`/`daytona` for scale or persistence-in-the-cloud, `singularity` for HPC.
- The `docker` backend's **persistent container** is the key lever: it gives reproducibility AND state carry-over, making it suitable for loops that need to install deps once or keep scratch state across calls within a process.
- For pure research/verification crons, prefer a read-only toolset (`safe`, or `web`+`file`+`search`) so the job structurally cannot mutate host state — defense-in-depth over relying on prompt-only honesty clauses.
- Be aware the `docker` container is a **single shared sandbox per Hermes process**: state written by one tool call is visible to later calls and to subagents; clean it (`/reset`) deliberately if isolation between tasks matters.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** The Self-Improvement and Alfred Learning-Log publish crons run with `enabled_toolsets: ["web","file","search"]` — i.e. `local` backend, terminal/execute_code explicitly absent. The "terminal-less" design is intentional (honesty-clause lesson, 2026-07-08). Backend is whatever the default profile's `terminal.backend` is (almost certainly `local`).
- **Gaps & Anti-Patterns:** (1) The loops assume `local` backend with no isolation — a misbehaving prompt or injected instruction could still write host files via `file` tools. (2) We have not consciously chosen a backend; it is the inherited default. (3) There is a native `safe` toolset that is the documented "read-only research" bundle, yet the loops hand-pick `web,file,search` instead — `file` includes write tools (`write_file`, `patch`), so the loop is NOT structurally read-only even though it lacks a terminal.

## 6. Recommended Improvements
1. **Decide and document the backend explicitly.** For the two learning loops, `local` is acceptable *only if* they must write lesson files to the vault. If a future hardened variant runs, prefer the `safe` toolset (true read-only) for any job that only reads/searches.
2. **Swap `file` → narrower toolsets where writes are not needed.** The loops need `write_file`/`patch` to publish lessons, so `file` is required — but the publish-*verification* step (the :30 mirror job) could run read-only with `safe` + `search` to structurally prevent mutation during verification.
3. **If ever migrating a loop to an isolated backend**, prefer `docker` (persistent sandbox, reproducible image) over `ssh`/`modal`/`daytona` for the lower operational cost; set `container_persistent` deliberately and remember state carries across the process.

## 7. Risk Assessment
- Switching the writing loops to `docker` adds image-build/lifecycle complexity and a per-process persistent container that must be managed; not worth it while the loops only touch the vault via `file` tools on the host.
- `safe` toolset cannot write lesson files, so it cannot host the publish step itself — only verification/audit steps.
- `container_persistent` persistence means a crashed process can leave `/workspace` state behind; only relevant if we ever adopt `docker`.
- No native backend prevents a prompt-injected `write_file` — isolation is a toolset concern (`safe`/`web,search`), not a backend concern. Keep the honesty + search_files self-check regardless of backend.

## 8. Next Learning Topic
How the cron system delivers output across platforms and chains jobs (`context_from`, `script` + `no_agent` for pure watchdog outputs), per-job model overrides, delivery targets, and the 3-minute hard interrupt. Source: https://hermes-agent.nousresearch.com/docs/user-guide/features/cron

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
