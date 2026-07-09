---
title: "Hermes Computer Use — Background Desktop/GUI Automation"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "computer-use", "curriculum:hermes"]
draft: false
status: Published
---

## 1. Topic Studied
How Hermes's `computer_use` toolset enables Alfred to drive a real desktop GUI — clicking, typing, scrolling, dragging — **in the background** on macOS, Windows, and Linux, without moving the user's cursor or stealing keyboard focus. Studied from the dedicated Computer Use doc plus the tools-runtime registration model. This is a Hermes-native capability not previously covered by the v2 backlog or any prior lesson.

## 2. Official Sources Consulted
- https://hermes-agent.nousresearch.com/docs/user-guide/features/computer-use
- https://hermes-agent.nousresearch.com/docs/reference/tools-reference (lists `computer_use` as a standalone tool)
- https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime (toolset dispatch model)

## 3. Key Concepts Learned
- **Background, no-foreground contract.** Hermes drives the desktop through the `computer_use` toolset, which speaks MCP over stdio to [`cua-driver`](https://github.com/trycua/cua), an open-source background computer-use driver. The agent reads the accessibility tree of any visible window and posts synthesized input events **without** bringing the window to front, switching virtual desktops, or moving the real OS cursor. A tinted "agent overlay cursor" glides to each target instead.
- **Model-agnostic.** Works with any tool-capable model (Claude, GPT, Gemini, or open models on local OpenAI-compatible endpoints) — no Anthropic-native schema required.
- **Cross-platform input dispatch:**
  - macOS: AX tree (SkyLight SPIs), `SLPSPostEventRecordTo` — pid-scoped, no cursor warp.
  - Windows: UIAutomation, `SendInput` + `PostMessage` — no focus steal.
  - Linux: AT-SPI (X11 + Wayland), XTest / virtual-keyboard.
- **Tool actions (from the documented quick example):** `computer_use(action="capture", mode="som", app="Mail")` (screenshot with numbered "set-of-marks" elements), `computer_use(action="click", element=14)`, `computer_use(action="type", text="...")`. Capture → click/type/scroll by element is the core loop.
- **Session isolation.** Each Hermes run declares its own cua-driver session id; concurrent runs/subagents get their own cursors.
- **First-triage CLI.** `hermes computer-use install`, `hermes computer-use status`, and `hermes computer-use doctor` (structured `health_report` MCP tool; exit 0 ok / 1 degraded / 2 binary missing). `doctor` reports per-check matrix (binary version, platform, TCC accessibility/screen-recording, AX/UIA/AT-SPI capability, screen-capture).
- **Enabling:** add `computer_use` to `~/.hermes/config.yaml` or run `hermes -t computer_use chat`. Requires platform prereqs (macOS Accessibility + Screen Recording TCC; Linux reachable `DISPLAY` or Wayland + AT-SPI).

## 4. Best Practices Discovered
- **Prefer `web_search`/`web_extract` and `browser_*` for retrieval** — the docs themselves say to prefer those for simple retrieval (faster, cheaper). Use `computer_use` only when the target is a **desktop GUI app** with no API/CLI/browser surface.
- **Capture with `mode="som"` first**, then act on numbered elements — keeps the model grounded on real UI targets rather than pixel coordinates.
- **Run `hermes computer-use doctor` before relying on it** in any automated context; degraded TCC/AX permissions fail silently otherwise.
- **Keep it background-only.** The value is co-working on the same machine without disrupting the user; do not attempt to foreground windows.
- **Per-session cursor isolation** lets multiple Alfred subagents drive different apps concurrently without cross-talk.

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** None. Alfred's execution is terminal-/script-bound (the entire `scripts/` automation layer, `execute_code`, `delegate_task`). There is no path today for Alfred to verify or operate a GUI app, screenshot a rendered artifact, or drive a desktop client.
- **Gaps & Anti-Patterns:**
  - Any task requiring a desktop GUI is currently impossible or force-fit into brittle shell/headless workarounds.
  - Rendered-output verification (e.g., confirming an app window, PDF viewer, or desktop notification actually displays correctly) has no native tool — Alfred can only assert on file contents, not pixels/UI.

## 6. Recommended Improvements
1. **Document the capability as available-on-demand** in Project Zero's `knowledge/` so future tasks that genuinely need GUI interaction reach for `computer_use` instead of being abandoned as "terminal-only."
2. **Add an `experiments/` spike** (non-governed): install `cua-driver` in a sandbox/desktop-capable environment, run `hermes computer-use doctor`, and drive one read-only GUI task (e.g., open an image in the system viewer and `capture` it) to confirm Alfred can invoke the toolset end-to-end. This confirms real behavior before any production reliance.
3. **Extend the Execution Tier Selection table** in AGENTS.md with a "GUI/Desktop" row noting `computer_use` as the native tier when the target lacks API/CLI/browser access — keeping it clearly separate from terminal/execute_code/delegation.

## 7. Risk Assessment
- **Environment-gated.** Requires a real display session and OS-level permissions (macOS TCC, Linux AT-SPI/XWayland). On headless servers (the typical Project Zero host) it is unavailable unless a desktop/display is provisioned — do not assume it works in cron/sandbox contexts.
- **Not a replacement for APIs.** Slower and more fragile than a CLI/API; reserve strictly for GUI-only targets.
- **Permission surface.** Grants the agent synthetic input to any visible window — scope to trusted, non-sensitive apps. Treat like terminal access in the threat model.
- **Linux Wayland** needs an XWayland bridge; pure Wayland may not expose AT-SPI cleanly.

## 8. Next Learning Topic
Hermes **Vision & `vision_analyze`** — multimodal image paste (`/paste`, base64 content blocks) routed through an auxiliary vision model, plus the `vision_analyze` standalone tool. Study https://hermes-agent.nousresearch.com/docs/user-guide/features/vision and the tools-reference standalone section, to determine how Alfred can analyze screenshots/images produced by `computer_use` or `browser_vision` (closing the perceive-act loop).

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
