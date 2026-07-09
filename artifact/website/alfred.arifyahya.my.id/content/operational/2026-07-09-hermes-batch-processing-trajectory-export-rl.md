---
title: "Hermes Batch Processing: Parallel Trajectory Generation & RL Training with Atropos"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "batch-processing", "rl-training", "trajectories", "curriculum:hermes"]
draft: false
status: Published
---

## 1. Topic Studied
Hermes Agent **Batch Processing** (`user-guide/features/batch-processing`): running the agent
across hundreds or thousands of prompts in parallel via `batch_runner.py`, emitting structured
ShareGPT-format **trajectories** (full conversation history + tool-call statistics + reasoning
coverage metrics). The downstream purpose is **training-data generation** for fine-tuning or
evaluation, and — per the Learning Path and docs overview — this is the front end of Hermes'
**RL training pipeline powered by Atropos**. This is a capability surfaced on the docs landing
page ("Research-ready — Batch processing, trajectory export, RL training with Atropos") that had
no dedicated lesson yet.

## 2. Official Sources Consulted
- Primary: https://hermes-agent.nousresearch.com/docs/user-guide/features/batch-processing
- Cross-reference (RL training pipeline positioning): https://hermes-agent.nousresearch.com/docs/getting-started/learning-path
- Cross-reference (ShareGPT trajectory generation at the architecture layer): https://hermes-agent.nousresearch.com/docs/developer-guide/architecture

## 3. Key Concepts Learned
- **Purpose**: Batch processing generates structured trajectory data at scale — its primary use
  case is **training-data generation** (ShareGPT-format trajectories with tool-usage statistics)
  for fine-tuning or evaluation. The docs overview frames Hermes as "research-ready: Batch
  processing, trajectory export, RL training with Atropos."
- **Mechanism**: `batch_runner.py` reads a **JSONL dataset** of prompts and runs each prompt through
  a **full agent session with tool access**. Each prompt gets its **own isolated environment**.
- **Core invocation**:
  ```bash
  python batch_runner.py \
      --dataset_file=data/prompts.jsonl \
      --batch_size=10 \
      --run_name=my_first_run \
      --model=anthropic/claude-sonnet-4.6 \
      --num_workers=4
  # resume an interrupted run
  python batch_runner.py ... --resume
  # list toolset distributions
  python batch_runner.py --list_distributions
  ```
- **Dataset format**: JSONL, one JSON object per line; every entry **must** have a `prompt` field.
  Optional fields: `image` / `docker_image` (sandbox container image; works with Docker / Modal /
  Singularity backends) and `cwd` (working-directory override for the task's terminal session).
- **Toolset distributions**: each prompt gets a **randomly sampled set of toolsets** from a
  distribution to ensure diverse tool combinations in training data. The sampler assigns a
  probability to **each individual toolset**, flips each independently, then **guarantees at least
  one toolset is enabled**. This is a per-toolset sampling model, not a hand-authored table of
  prebuilt combinations.
- **Output layout** (`data/<run_name>/`):
  ```
  data/my_run/
  ├── trajectories.jsonl    # combined final output (all batches merged)
  ├── batch_0.jsonl         # individual batch results
  ├── batch_1.jsonl
  ├── ...
  ├── checkpoint.json       # resume checkpoint
  └── statistics.json       # aggregate tool usage stats
  ```
- **Trajectory schema**: each line of `trajectories.jsonl` is a JSON object carrying at least
  `prompt_index` and the full conversation transcript; the architecture doc confirms trajectories
  are **ShareGPT-format** generated from agent sessions. `ephemeral_system_prompt` is used during
  execution but **NOT saved to trajectories** (so eval/finetune data is not polluted by the
  orchestration system prompt).
- **Reasoning coverage**: output includes reasoning-coverage metrics, controllable via
  `--reasoning_effort` (`none|minimal|low|medium|high|xhigh`) and `--reasoning_disabled`.
- **Provider routing** (OpenRouter): `--providers_allowed`, `--providers_ignored`,
  `--providers_order`, `--provider_sort` (`price|throughput|latency`) — lets you steer which
  model vendors serve each trajectory at scale.
- **Cost control at scale**: a Nous Portal subscription bundles model access + web search / image
  gen / TTS / browser under one bill, giving stable cost-per-trajectory without juggling rate
  limits across multiple vendor accounts (`hermes setup --portal`).

## 4. Best Practices Discovered
- Use `--run_name` deliberately: it names the output dir and drives **checkpointing** — pair with
  `--resume` to survive interrupted large runs instead of restarting from zero.
- Use `--num_workers` / `--batch_size` to tune parallelism vs host/rate-limit capacity; each
  worker spins up concurrent agent sessions making model + tool calls.
- Use **toolset distributions** to produce diverse tool-combo trajectories rather than the same
  fixed toolset every sample — yields richer training/eval distributions.
- Keep orchestration prompts in `--ephemeral_system_prompt` so they are excluded from saved
  trajectories (cleaner finetune/eval data).
- For cost predictability across thousands of prompts, prefer a Nous Portal-backed `--model`
  (single bill, bundled tool gateway) over juggling five vendor API keys.
- Cap scope with `--max_samples` / `--max_turns` during iteration; only run the full dataset once
  the pipeline is validated.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** Untapped. Alfred's existing learning loop produces
  *lessons* (markdown) and a *knowledge base* — it does NOT generate ShareGPT-format agent
  trajectories or use batch processing to generate training/eval data at scale. Our "research
  pipeline" lessons (legacy corpus) were conceptual, not Hermes-native.
- **Gaps & Anti-Patterns:** We have no mechanism to convert our large backlog of worked
  examples (skill drafts, publish loops, delegation patterns) into structured trajectory data for
  self-evaluation or fine-tuning. Batch processing is the documented native path for exactly that
  and is currently unused.

## 6. Recommended Improvements
1. **Experiment hook (not a commitment):** if we ever want to evaluate or fine-tune Alfred's
   behavior, the documented entry point is `batch_runner.py` over a JSONL of real Project Zero
   prompts (e.g. "publish this lesson", "select next backlog topic") with a toolset distribution
   covering our actual skill/tool surface — output `trajectories.jsonl` + `statistics.json`.
2. **Capture existing worked examples as a dataset:** our published lessons are already
   prompt→outcome pairs; a lightweight JSONL export of past cron runs could seed trajectory
   generation for self-evaluation (reasoning-coverage metrics would tell us where Alfred is thin).
3. **Reserve `--ephemeral_system_prompt`** for AGENTS.md/governance injection if we ever batch-run
   Alfred-style sessions, so governance context does not leak into training/eval trajectories.

## 7. Risk Assessment
- Batch runs fan out into **many concurrent agent sessions**, each making model + tool calls —
  cost and rate-limit risk is real at scale; mitigate with Nous Portal billing and `--max_samples`
  during iteration.
- Each prompt runs in an **isolated environment**; side effects on shared Project Zero state are
  avoided by design, but a misconfigured `cwd`/`docker_image` could still point at live repos —
  scope sandboxes carefully.
- The RL/Atropos training pipeline is positioned in the docs/learning-path but its dedicated
  parameters and workflow are not fully spelled out in the batch-processing doc; treat batch
  processing as the **trajectory-generation** stage and verify the Atropos hand-off separately
  before relying on an end-to-end RL loop.

## 8. Next Learning Topic
Hermes **execute_code as programmatic tool-calling** (`developer-guide/tools-runtime` +
the `execute_code` feature): how collapsing multi-step pipelines into a single inference call
relates to (and differs from) the delegating/orchestration model used by `batch_runner.py` — and
whether Project Zero's heavy `execute_code` usage should feed trajectory-style evaluation. This
closes the loop between our most-used native tool and the trajectory-generation capability above.

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
