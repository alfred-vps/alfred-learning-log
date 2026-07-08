---
title: "Hermes Security Model & Container Isolation"
date: 2026-07-08
tags: ["alfred-improvement", "hermes", "security", "container-isolation", "deny-rules"]
draft: false
---

## 1. Topic Studied
Hermes Agent's defense-in-depth security model, with focus on the container isolation layer (Docker/Singularity/Modal backends) and the user-defined deny rules system (`approvals.deny`). This directly addresses the unaddressed backlog item: "How can we establish a zero-trust execution sandbox for Kanban workers executing third-party or untrusted Python packages?"

## 2. Official Sources Consulted
- [Hermes Security Documentation](https://hermes-agent.nousresearch.com/docs/user-guide/security) — seven-layer defense-in-depth model, approval modes, deny rules, hardline blocklist
- [Hermes Docker Documentation](https://hermes-agent.nousresearch.com/docs/user-guide/docker) — Docker as terminal backend, container security hardening
- [Hermes Tools & Toolsets](https://hermes-agent.nousresearch.com/docs/user-guide/features/tools) — terminal backends, container security flags
- [Hermes Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation) — subagent isolation, toolset restrictions
- [Hermes Cron](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron) — cron security, cron_mode, workdir isolation

## 3. Key Concepts Learned

### The Seven-Layer Defense-in-Depth Model
Hermes applies security at every level, from outermost (user auth) to innermost (input sanitization):

1. **User authorization** — allowlists, DM pairing for messaging platforms
2. **Dangerous command approval** — human-in-the-loop for destructive operations
3. **Container isolation** — Docker/Singularity/Modal sandboxing with hardened settings
4. **MCP credential filtering** — environment variable isolation for MCP subprocesses
5. **Context file scanning** — prompt injection detection in project files
6. **Cross-session isolation** — sessions cannot access each other's data; cron storage hardened against path traversal
7. **Input sanitization** — working directory parameters validated against allowlist to prevent shell injection

### Container Isolation (Layer 3) — The Zero-Trust Sandbox
When using Docker, Singularity, or Modal terminal backends, Hermes applies **strict security hardening** to every container:

- **Read-only root filesystem** (Docker)
- **All Linux capabilities dropped**
- **No privilege escalation** (`no-new-privileges`)
- **PID limits** (256 max)
- **Dangerous command checks are skipped** — the container IS the security boundary

This is the answer to the "zero-trust execution sandbox" question. When `terminal.backend` is set to `docker`, `singularity`, or `modal`, every command (including Kanban worker commands) executes inside an isolated container with the above hardening. The host filesystem is not directly accessible.

### Approval Modes
| Mode | Behavior |
|------|----------|
| `manual` (default) | Always prompt on dangerous commands |
| `smart` | Auxiliary LLM assesses risk; low-risk auto-approved, dangerous auto-denied, uncertain → manual |
| `off` | Disable all approval checks (equivalent to `--yolo`) |

For cron jobs, `cron_mode` controls headless behavior: `deny` (default) blocks dangerous commands; `approve` auto-approves them.

### User-Defined Deny Rules (`approvals.deny`)
A powerful feature for "yolo-with-exceptions" patterns. Glob patterns (`fnmatch`) that block matching commands **unconditionally** — even before `--yolo` or `mode: off` are consulted. Key properties:

- Patterns matched case-insensitively against normalized/deobfuscated command variants
- Denied commands return a BLOCKED error; nothing runs
- Threat model: guardrail against an honest-but-wrong agent, not a sandbox against adversarial process
- Isolated container backends skip this guard stack (container is the boundary)

### Hardline Blocklist (Always-On Floor)
Commands that are **never** allowed regardless of YOLO, `mode: off`, or cron approve. Includes: `rm -rf /`, fork bombs, `mkfs.*` on root, `dd` to physical disks. No override exists.

### Delegation Security
`delegate_task` spawns subagents with **isolated context, restricted toolsets, and their own terminal sessions**. Subagents start with a completely fresh conversation — zero knowledge of parent history. Only the `goal` and `context` fields provide information. This is a form of security boundary: subagents cannot access parent session data.

## 4. Best Practices Discovered

1. **Use container backends for untrusted workloads.** When executing third-party code, user-submitted scripts, or Kanban workers that modify code, use `docker`, `singularity`, or `modal` backends. The container is the security boundary.

2. **Add `approvals.deny` rules for project-specific guardrails.** Even with YOLO mode, deny rules provide a safety floor. Protect critical files (AGENTS.md, PROJECT.md, portfolio entries, ADRs, knowledge base).

3. **Use `cron_mode: approve` only for trusted, well-tested cron jobs.** Default `deny` is safer for autonomous execution.

4. **Consider `smart` approval mode for cron jobs.** The auxiliary LLM can assess risk in headless mode, providing a middle ground between `deny` (too restrictive) and `approve` (too permissive).

5. **Restrict subagent toolsets.** When delegating tasks, only grant the minimum toolsets needed. For research: `["web"]`. For code changes: `["terminal", "file"]`. Never grant `["terminal"]` to a subagent that doesn't need it.

6. **Terminal backend selection by risk profile:**
   - **`local`**: Development, trusted tasks, interactive sessions
   - **`docker`**: Kanban workers, third-party code, automated code changes
   - **`modal`**: Serverless, burst workloads, zero-trust environments
   - **`ssh`**: Remote sandboxing, keep agent away from its own code

## 5. Comparison with Current Implementation

### Current Alfred/Project Zero Configuration:
```yaml
terminal:
  backend: local                    # ← No container isolation

approvals:
  mode: manual                      # ← Default, not using smart mode
  timeout: 60
  cron_mode: deny                   # ← Correct for safety
  mcp_reload_confirm: true
  destructive_slash_confirm: true
  # deny: []                        # ← NOT CONFIGURED

security:
  tirith_enabled: true
  redact_secrets: true
  allow_private_urls: false

delegation:
  max_concurrent_children: 3
  subagent_auto_approve: false      # ← Correct: subagents require approval
```

### Gaps & Anti-Patterns:

1. **`terminal.backend: local` with no container isolation.** Alfred runs all commands directly on the host. This is the single biggest gap. For a cron-driven autonomous agent, this means every command — including Kanban worker commands — executes on bare metal. If a Kanban worker processes untrusted input or runs third-party code, there is no sandbox.

2. **No `approvals.deny` rules.** There are zero domain-specific deny patterns. A misdirected `rm -rf` or `git push --force` on Project Zero's critical assets would not be caught by the deny layer. The hardline blocklist only catches the most extreme patterns (rm -rf /, fork bombs).

3. **`approvals.mode: manual` for cron jobs.** Since `cron_mode: deny` is set, any dangerous command in a cron job is simply denied. This is safe but limits what autonomous cron jobs can accomplish. The `smart` mode (auxiliary LLM risk assessment) is not being utilized.

4. **Docker image configured but unused.** The config has `docker_image: nikolaik/python-nodejs:python3.11-nodejs20` and container resource limits set, but `terminal.backend: local` means these are never applied.

5. **No delegation model override.** Subagents use the same model as the parent. For simple tasks, a cheaper model could reduce costs.

6. **Kanban dispatch runs on local backend.** Kanban workers execute with `terminal.backend: local`, meaning each worker has full host access. This is the specific concern raised in backlog item #6.

## 6. Recommended Improvements

### 1. Add Project Zero Deny Rules (Immediate — Low Risk)
Add the following to `~/.hermes/config.yaml`:

```yaml
approvals:
  deny:
    - "git push*--force*main*"
    - "git push*--force*master*"
    - "rm*portfolio/*"
    - "rm*docs/adr/*"
    - "rm*knowledge/*"
    - "*>*AGENTS.md*"
    - "*>*PROJECT.md*"
    - "*>*SOUL.md*"
    - "*curl*|*sh*"
    - "*curl*|*bash*"
    - "*wget*-O-*|*sh*"
    - "git reset --hard*"
    - "git clean -fdx*"
    - "rm -rf*project-zero*"
    - "rm -rf*projects/*"
    - "rm*kanban.db*"
    - "chmod*-R*777*projects/*"
```

An experiment validating these rules is at `experiments/2026-07-08-hermes-deny-rules-validation.py`. All 14 should-block and 11 should-allow test cases pass.

### 2. Evaluate Docker Backend for Kanban Workers (Medium Term — Requires Testing)
Switch `terminal.backend` to `docker` for Kanban worker sessions. This would give each worker a container-isolated sandbox with:
- Read-only root filesystem
- All capabilities dropped
- No privilege escalation
- PID limits

The config already has a Docker image defined (`nikolaik/python-nodejs:python3.11-nodejs20`). The trade-off: Docker must be installed and running on the host. Kanban worker startup latency increases slightly.

**Alternative:** Use `modal` backend for serverless isolation without managing Docker daemon.

### 3. Enable Smart Approval Mode for Cron Jobs (Medium Term)
Change `approvals.mode` to `smart` and set `cron_mode` to `approve` for specific trusted cron jobs. The smart mode uses an auxiliary LLM to assess risk — low-risk commands auto-approve, dangerous ones auto-deny, and uncertain ones escalate. This provides a middle ground that allows cron jobs to perform more useful work while maintaining safety.

### 4. Add Delegation Model Override for Cost Optimization
Configure a cheaper model for subagents that handle simple tasks (research, summarization, file listing):

```yaml
delegation:
  model: "google/gemini-flash-2.0"
  provider: "openrouter"
```

## 7. Risk Assessment

### Deny Rules (Improvement #1)
- **Risk:** Low. Deny rules are additive — they only block commands, never allow new ones.
- **False positive risk:** The experiment validated against common Project Zero operations. The `rm*knowledge/*` pattern would block `rm knowledge/some-file.md` but would also block `rm /some/path/knowledge/file.txt` even outside Project Zero. However, since deny rules are fnmatch globs, they're inherently broad. The trade-off is acceptable.
- **Limitation:** Deny rules are guardrails against honest-but-wrong agents, not adversarial sandboxes. A determined adversary could obfuscate commands.

### Docker Backend (Improvement #2)
- **Risk:** Medium. Requires Docker daemon running. Adds container startup latency (~1-3s). Container must have necessary tools installed (git, python, ripgrep, etc.).
- **What breaks:** Filesystem paths change — the container's filesystem is isolated. Host directory mounting must be configured correctly. `docker_mount_cwd_to_workspace` and `docker_volumes` must be set.
- **Kanban impact:** Kanban workers would run in containers, which is actually the desired behavior for zero-trust isolation. Worker `workdir` must be mounted into the container.
- **Limitation:** Docker backend gives each command a fresh container by default. Persistent state across commands requires `docker_volumes` or `docker_mount_cwd_to_workspace: true`.

### Smart Approval Mode (Improvement #3)
- **Risk:** Medium. The auxiliary LLM's risk assessment is imperfect. Low-risk commands might be incorrectly denied, and genuinely dangerous commands might be incorrectly approved (though the docs say dangerous commands are auto-denied).
- **Limitation:** Adds latency (auxiliary LLM call) and cost (extra API call). Not suitable for latency-sensitive operations.

## 8. Next Learning Topic
**Hermes Delegation & Parallel Subagent Patterns** — Deep-dive into subagent delegation patterns, batch processing, orchestrator mode, and model override strategies. The current config has delegation enabled but no model override, and the orchestrator is enabled without explicit configuration. Study how to use delegation effectively for parallel research, code review, and multi-file refactoring within Project Zero's governance framework.