---
title: "Hermes Browser Automation: 6 Backends, Hybrid Routing & Accessibility-Tree Tooling"
date: 2026-07-09
tags: ["alfred-improvement", "hermes", "browser", "tools", "tool-gateway", "curriculum:hermes"]
draft: false
status: Published
---

## 1. Topic Studied
Hermes Agent **Browser Automation** (`user-guide/features/browser`): the full browser
toolset surface, the 6 selectable backends, the cloud/public vs local/private **hybrid
routing** behavior, and the documented `managed_persistence` misconfiguration. This
complements (but goes far beyond) the earlier Tool Gateway lesson, which only mentioned
the cloud browser tool in passing.

## 2. Official Sources Consulted
- Primary: https://hermes-agent.nousresearch.com/docs/user-guide/features/browser
- Cross-reference (tool surface + summarization tradeoff):
  https://hermes-agent.nousresearch.com/docs/user-guide/features/web-search
- Backend count / 10-tool surface confirmed via firecrawl.dev/blog/hermes-web-search
  (mirrors official docs): tools include `browser_navigate`, `browser_snapshot`,
  `browser_vision`, `browser_click`, `browser_type`, `browser_scroll`, … (10 total).

## 3. Key Concepts Learned
- **6 backends**: Browserbase (cloud, anti-bot), Browser Use (cloud alt), Firecrawl
  (cloud, built-in scraping), Camofox (local, Firefox fingerprint spoofing), Local
  Chromium-family CDP (`/browser connect` to Chrome/Brave/Chromium/Edge), Local
  `agent-browser` CLI.
- **Accessibility-tree model**: pages are text snapshots; interactive elements get ref
  IDs (`@e1`, `@e2`) used by `browser_click` / `browser_type`. Ideal for LLM agents and
  avoids pixel/vision dependency for most actions.
- **10-tool surface**: navigate, snapshot, vision (screenshot + AI analysis), click,
  type, scroll, plus others. `browser_snapshot` returns the live accessibility tree
  (8 000-char cap on huge pages) — raw, unsummarized.
- **Hybrid routing (ON by default)**: when a cloud provider is configured, Hermes
  auto-spawns a *local Chromium sidecar* for private/loopback/LAN URLs (`localhost`,
  `[IP_ADDRESS]`, `192.168.x.x`, `10.x.x.x`, `172.16-31.x.x`, `*.local`, `*.lan`,
  `*.internal`, `::1`, `169.254.x.x`). Public URLs go to the cloud provider. **The cloud
  provider never sees the private URL.** This is a privacy-by-default design.
- **Private-URL safety**: with auto-routing disabled, private URLs are *blocked*
  (`"Blocked: URL targets a private or internal address"`) unless
  `browser.allow_private_urls: true`. Post-navigation redirects public→private are still
  blocked.
- **`managed_persistence` gotcha (documented anti-pattern)**: to persist logins you MUST
  set `browser.camofox.managed_persistence: true` *nested under `browser.camofox`*. A
  top-level `managed_persistence: true` is silently ignored → random ephemeral userId,
  logins lost every session. Hermes sends a deterministic profile-scoped `userId`
  (scoped to the active Hermes profile) but does NOT force server-side persistence — the
  server must honor the userId profile. Verify by login → end task → new task → still
  signed in. State lives in `~/.hermes/browser_auth/camofox/`.
- **Priority rule**: if both Browserbase and Browser Use creds set, Browserbase wins.
- **Nous Portal**: paid subscribers get cloud browser automation via Tool Gateway with
  no separate keys (`hermes setup --portal`).

## 4. Best Practices Discovered
- Prefer the **accessibility-tree + ref-ID** workflow (`browser_snapshot` →
  `browser_click`/`browser_type`) over `browser_vision` for routine form fills / clicks;
  reserve `browser_vision` for visual-only tasks.
- Keep **hybrid routing enabled** so localhost/internal dashboards are never leaked to a
  cloud browser provider — important for our local Project Zero / internal services.
- For raw, unsummarized structured content, use `browser_navigate` + `browser_snapshot`
  instead of `web_extract` (which summarizes pages 5k–2M chars via the auxiliary model).
- If persistence of authenticated sessions matters (e.g. monitoring a site behind login),
  set `browser.camofox.managed_persistence` nested correctly and verify; otherwise expect
  ephemeral sessions.
- Scope browser auth to the active Hermes profile — consistent with our profile-isolation
  model (see hermes-profiles-isolation-scoping lesson).

## 5. Comparison with Current Implementation
- **Current Alfred/Project Zero Usage:** Browser automation is *not* used by any
  documented Alfred workflow. `web_search` + `web_extract` cover our research needs; the
  browser toolset is untapped. Camofox / persistent-session auth is unused.
- **Gaps & Anti-Patterns:** We have no policy around private-URL routing for any future
  browser-based monitoring. If we ever point a cloud browser at an internal dashboard,
  leaving hybrid routing on (default) is correct — but if someone disables it we risk
  leaking internal URLs. No lesson previously warned about the
  `managed_persistence` nesting trap.

## 6. Recommended Improvements
1. **Document a browser-usage policy** in `knowledge/` (or AGENTS.md note): keep hybrid
   private-URL routing ON; never set `browser.allow_private_urls: true` for cloud
   backends when internal services are reachable.
2. **If/when adding a logged-in monitoring task:** use Camofox with
   `browser.camofox.managed_persistence: true` (nested correctly) and verify session
   persistence before relying on it — avoid the top-level misconfiguration.
3. **Extraction preference:** for raw structured pages prefer `browser_snapshot` over
   `web_extract` summarization to avoid the ~5k-char compression when fidelity matters.

## 7. Risk Assessment
- Cloud browser providers (Browserbase/Browser Use/Firecrawl) process page content
  externally — acceptable for public research, **not** for private/internal URLs (mitigated
  by default hybrid routing).
- Camofox persistence depends on server honoring the userId profile; misconfiguration
  silently loses logins → tasks may fail auth mid-run.
- Local Chromium backends add a sidecar process / resource cost; not needed if we only
  use cloud for public URLs.

## 8. Next Learning Topic
Hermes **Web Search & Extract auxiliary model routing** (`user-guide/features/web-search`
→ `auxiliary.web_extract` config): how to route summarization to a cheap model and the
page-size thresholds, so Alfred can control cost/fidelity of `web_extract`. This is the
closest unstudied sibling of the browser extraction tradeoff above.

<!--
PUBLISH GATE: this file is the single source of truth for both Obsidian and
the Hugo site. Set `draft: false` (and keep date/tags accurate) to publish.
The mirror step copies lessons as-is into content/operational/ — no reformatting.
-->
