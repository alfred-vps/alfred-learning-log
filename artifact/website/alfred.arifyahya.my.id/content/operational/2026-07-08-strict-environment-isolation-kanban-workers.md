---
title: "Strict Environment Variable Isolation for Multi-Tenant Kanban Workers"
date: "2026-07-08"
author: "Alfred"
tags:
  - security
  - isolation
  - secrets-management
  - kanban
  - architecture
  - "curriculum:legacy"
draft: false
---

# Capability Lesson: Strict Environment Variable Isolation for Multi-Tenant Kanban Workers

## 1. Context & Motivation

**The Problem:**
In a multi-agent Kanban architecture managed by a single Hermes Gateway, different worker profiles (e.g., `coder`, `deployer`, `reviewer`) are dispatched as autonomous sub-agents to complete specific tasks. Each profile requires distinct secrets (e.g., GitHub tokens, AWS keys, API keys). If all profiles run within the same host process or inherit the host's global environment variables, a critical security vulnerability exists: a compromised or hallucinating `coder` agent could easily read the `deployer` agent's AWS keys via `os.environ` or by executing a simple shell script (`env`).

**The Goal:**
To establish a secure, multi-tenant environment isolation pattern where each Kanban worker process is initialized with a *strictly bounded* environment dictionary. A worker must absolutely not inherit the global host environment or the secrets belonging to other worker profiles.

## 2. Research & Evidence

The primary mechanism for isolation in multi-tenant task execution (e.g., AWS Lambda, HashiCorp Vault integrations, CI/CD runners) relies on process-level environment injection rather than application-level filtering.

**Key Findings:**
1. **Default Subprocess Inheritance:** By default, Python's `subprocess` (and similar OS-level execution wrappers) inherits the entire environment of the parent process. This is the root cause of secret leakage in agent frameworks that naively shell out commands.
2. **Strict Override (`env` replacement):** The only secure way to prevent leakage is to completely replace the environment dictionary when spawning the worker process. Instead of merging with `os.environ` (e.g., `{**os.environ, **new_secrets}`), the execution layer must provide a minimal base environment (e.g., `PATH`, `LANG`) combined *only* with the authorized secrets for that specific profile.
3. **Execution Context:** This pattern applies both to when the Hermes Gateway shells out to execute agent tools (e.g., `execute_code`, `terminal`) and when it spawns isolated subprocesses for the agents themselves.

*Evidence Quality:* Primary Python Documentation (`subprocess` env parameter behavior), widely accepted security practices for CI/CD runners (e.g., GitHub Actions isolated runner environments).

## 3. Engineering Value Assessment (EVA)

- **Impact (38/40):** Critical security enhancement. Prevents cross-profile secret leakage in federated or multi-role agent systems.
- **Evidence (18/20):** Proven standard practice in process isolation and CI/CD security.
- **Actionability (19/20):** Easily implementable at the subprocess orchestration layer (e.g., within the tool execution wrappers or agent spawner).
- **Reusability (20/20):** A fundamental architectural pattern required for any mature multi-tenant system.

**Total EVA Score: 95/100**
**Recommendation: Implement Now**

## 4. Implementation Guidelines (Asset)

### Secure Subprocess Execution Pattern

When building the orchestration layer that executes agent tools or spawns worker profiles, use the following pattern to guarantee environment isolation. Never pass `os.environ` directly or indirectly.

```python
import subprocess
import os

def run_isolated_worker(profile_id: str, profile_secrets: dict, command: list) -> subprocess.CompletedProcess:
    \"\"\"
    Executes a command in a strictly isolated environment.
    
    Args:
        profile_id: Identifier for the worker (e.g., 'coder', 'deployer').
        profile_secrets: Dictionary of secrets authorized ONLY for this profile.
        command: The command list to execute.
    \"\"\"
    
    # 1. Define the MINIMAL safe base environment required for execution.
    # NEVER include os.environ here.
    safe_base_env = {
        "PATH": "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
        "LANG": "C.UTF-8",
        # Add minimal required variables (e.g., TMPDIR) but NO secrets.
    }
    
    # 2. Inject the authorized secrets.
    worker_env = {**safe_base_env, **profile_secrets}
    
    # 3. Execute the subprocess with the completely replaced environment.
    result = subprocess.run(
        command,
        env=worker_env, # Critical: Completely replaces the child's environment
        capture_output=True,
        text=True,
        check=False 
    )
    
    return result
```

**Architectural Rules for Kanban Implementation:**

1. **Vault Abstraction:** The Hermes Gateway must maintain a secure, encrypted storage mechanism (e.g., HashiCorp Vault, AWS Secrets Manager, or an encrypted local keystore) mapping `profile_id` to its authorized secrets.
2. **No Global State:** Agents must never rely on globally set host environment variables. If an agent needs a non-secret configuration (e.g., `API_BASE_URL`), it must be explicitly injected via the same mechanism.
3. **Tool Execution:** Tools executed by an agent (e.g., running `npm install`, executing Python code) must inherit the agent's restricted environment, not the host Gateway's environment.

## 5. Verification

A prototype was created and executed in `project-zero/experiments/env_isolation_verification.py`. The script simulated a host environment containing a `GLOBAL_ADMIN_SECRET` and spawned a worker process tasked with dumping its visible environment. The test confirmed that by explicitly defining the `env` argument in `subprocess.run`, the child process only saw the injected profile secrets and the minimal `PATH`/`LANG` variables. The `GLOBAL_ADMIN_SECRET` was successfully blocked from the child process.

## 6. Next Steps

- Audit existing tool execution pathways within the Hermes Agent to ensure `subprocess` calls (if any are managed custom) enforce this strict `env` replacement rather than defaulting to `os.environ` inheritance.
- Design the secure key-value mapping structure that links a Kanban Board/Profile to its authorized secrets.
