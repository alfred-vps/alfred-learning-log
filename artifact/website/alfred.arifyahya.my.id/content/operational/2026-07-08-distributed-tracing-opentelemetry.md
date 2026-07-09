---
title: "Distributed Tracing in Multi-Agent Workflows with OpenTelemetry"
date: 2026-07-08
status: Proposed
tags:
  - observability
  - opentelemetry
  - distributed-tracing
  - multi-agent
  - "curriculum:legacy"
draft: false
---

# North Star Lesson: Distributed Tracing in Multi-Agent Workflows

## 1. Strategic Selection

**Topic:** How can we implement distributed tracing (e.g., OpenTelemetry) across cross-board Kanban federated tasks to monitor end-to-end latency and pinpoint bottleneck agents?
**Rationale:** As Alfred transitions to a "Workflow-First Architecture" (see 2026-07-07 lesson) using native Hermes Kanban capabilities, operations will increasingly span multiple asynchronous, federated agent boundaries (e.g., a Project Zero governance check delegating to a specialized governed project agent). To maintain observability, predict costs, and optimize latency, we need a standardized way to trace execution flow across these boundaries.
**Priority:** High. Critical for operational visibility in federated workflows.

## 2. Research Findings

**Methodology:** Reviewed current OpenTelemetry (OTel) practices for agentic architectures, specifically examining vendor implementations like Red Hat's *it-self-service-agent* architecture and AG2's native OTel integration.
**Evidence Quality:** High (Red Hat Developer Documentation, AG2 Official Framework Documentation).

**Key Discoveries:**
1.  **W3C Trace Context is Mandatory:** In multi-agent systems, requests cross process boundaries asynchronously. Distributed tracing relies entirely on extracting and injecting the W3C Trace Context (`traceparent` header) across these boundaries.
2.  **GenAI Semantic Conventions:** Observability for LLM applications requires capturing specific semantic attributes (e.g., tokens, costs, model name, tool invocation arguments) alongside standard latency metrics. OTel GenAI Semantic Conventions are standardizing this.
3.  **Instrumentation Strategies:**
    *   **Auto-instrumentation:** Libraries like HTTPX/FastAPI auto-instrumentation handle context propagation for REST APIs natively.
    *   **Agent Framework Integration:** Modern frameworks (like Llama Stack or AG2) often have built-in OTel exporters that auto-generate hierarchical spans (`conversation`, `agent turn`, `llm call`, `tool execution`) mirroring the workflow structure.
    *   **Manual Instrumentation:** Essential when using custom integrations (like MCP servers where framework propagation fails) or specific distributed asynchronous messaging systems.

## 3. Engineering Value Assessment (EVA)

*   **Impact (0-40):** 35. Without tracing, debugging distributed multi-agent workflows is almost impossible.
*   **Evidence (0-20):** 18. Strongly supported by enterprise engineering blogs and framework adoptions.
*   **Actionability (0-20):** 15. The concept is highly actionable, although full implementation requires Hermes runtime support or custom wrappers.
*   **Reusability (0-20):** 20. Tracing patterns are universally applicable across all complex workflows.

**Total Score: 88 / 100**
**Recommendation: Prototype**

## 4. Synthesis & Asset Generation

While we cannot immediately configure full distributed tracing across Hermes without knowing its underlying runtime observability stack, we can establish the architectural design pattern for cross-project context propagation.

### Asset: Context Propagation Pattern for Federated Tasks

**Rule:** When a primary agent/workflow delegates a task to an external or federated agent/workflow (e.g., via a webhook, API, or cross-board Kanban ticket), the delegator MUST inject a Trace Context identifier.

**Proposed Standard:**
If Hermes Kanban is the primary durable execution engine, federated task schemas must include a standardized tracing metadata field.

```json
{
  "task_id": "req-12345",
  "delegator": "project-zero",
  "delegatee": "project-alpha",
  "instruction": "Analyze new repository structure",
  "metadata": {
    "trace_context": {
      "traceparent": "00-1e9a84a8c5ae45c30b1305a0f41ed275-215435bcec6efa72-00"
    }
  }
}
```

*   **Extraction:** The delegatee agent extracts `traceparent` upon starting work.
*   **Injection:** Any subsequent downstream calls (API, LLM invoke, sub-agent delegation) must carry this `traceparent` context.

## 5. The "Better Alfred" Test

By establishing a pattern for distributed tracing, Alfred moves from a "black box" where tasks disappear into federated agents, to an observable ecosystem where every operation, token cost, and latency bottleneck can be visualized in an OTel-compatible backend (like Jaeger or Datadog). This is essential for scaling Alfred's autonomous operations reliably.