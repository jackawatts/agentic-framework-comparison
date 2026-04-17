# Evaluation Criteria

This document defines the axes on which we compare agent frameworks. The weekly refresh workflow uses these axes verbatim — if a new capability (e.g. a new MCP sub-spec) becomes industry standard, add it here first, then the per-framework deep dives get re-scored against it.

Benchmark date of this revision: **April 2026**. Frameworks are expected to support the capabilities that a "modern" agentic scaffold provides as of this date. Anything missing is a deficiency and should be called out in the framework's "Gaps" section, not quietly excluded from scoring.

## Hard constraints

These are gates. A framework that fails any of these is out of scope for this project.

| Gate | Requirement |
|---|---|
| License cost | MIT / Apache-2.0 / BSD OSS, zero per-seat or per-token framework fees. Paid hosted services built *on top* of the framework (e.g. LangGraph Platform, LangSmith) are permitted only if the framework is fully usable without them. |
| Azure runnable | Must run on Azure compute a team already owns (Container Apps, App Service, Functions, AKS, or Azure VMs) without needing a proprietary control plane. |
| Target model | Must be able to call **Azure OpenAI** or **Azure AI Foundry model catalog** endpoints directly, or via a thin adapter. Exception: Claude Agent SDK is included for comparison despite this, with the mismatch called out explicitly. |
| Language | Official support for **TypeScript** and/or **.NET**. Python-only frameworks are excluded unless the Python story is so strong it justifies a language switch (state the case). |

## Evaluation axes

Every framework's deep dive answers every axis below. If the answer is "no" or "not yet", say so — do not skip.

### 1. Model & provider support
- Azure OpenAI (chat completions + Responses API)
- OpenAI direct
- Azure AI Foundry model catalog (non-OpenAI models: Llama, Mistral, Phi, Claude via Foundry, etc.)
- Provider switchability (can you swap models without rewriting the agent)

### 2. Language / runtime support
- TypeScript / Node.js — official package, not community port
- .NET — official package, not community port
- Python — for completeness
- Runtime targets (Node LTS versions, .NET version floor)

### 3. Tool / function calling
- API ergonomics (decorators, typed schemas, manual JSON Schema)
- Parallel tool calls
- Hosted tools (vendor-provided tools like web search, file search, code interpreter) — and which hosted tools work on Azure endpoints vs OpenAI direct only
- Tool approval / permission model

### 4. Model Context Protocol (MCP)
MCP is the assumed inter-agent/tool protocol in 2026. Score:
- MCP **client** support (consume MCP servers) — stdio, HTTP streamable, SSE
- MCP **server** support (expose the agent's tools as an MCP server)
- MCP spec version targeted
- Auth: OAuth 2.1, custom token, both

### 5. Memory
- Short-term / thread state (conversation history persistence)
- Long-term memory (facts across threads/users)
- Episodic vs semantic memory primitives
- Checkpointing / resume-after-crash
- Supported backends (in-memory, SQLite, Postgres, Redis, Cosmos DB, Azure AI Search, vector stores)
- First-class Azure-native backend support

### 6. Skills / reusable capability packaging
"Skills" in the modern sense (Anthropic Skills, Foundry Agent Tools, SK Plugins, LangChain Toolkits): packaged, discoverable, model-invoked bundles of prompts + tools + resources. Score:
- Does a skill-like primitive exist
- Is it discoverable by the model at runtime (progressive disclosure) or loaded up-front
- Is it portable across frameworks (e.g. can a Foundry skill run in MAF, or vice versa)

### 7. Multi-agent orchestration
- Handoff / routing primitive
- Supervisor / orchestrator pattern
- Parallel / fan-out execution
- Workflow graph primitive (explicit state machine)
- Human-in-the-loop interrupt / resume

### 8. Observability
- Built-in tracing
- **OpenTelemetry** support (GenAI semantic conventions)
- Azure Monitor / Application Insights integration
- Azure AI Foundry tracing integration
- Whether observability requires a paid SaaS (e.g. LangSmith) or works with self-hosted/OTel-native backends

### 9. Guardrails / safety
- Input / output content filters
- PII redaction hooks
- Azure AI Content Safety integration
- Jailbreak / prompt-injection mitigations
- Permission / tool-gating model

### 10. Streaming
- Token streaming
- Event streaming (tool calls, handoffs, state changes)
- Resumable streams

### 11. Deployment on Azure
- Recommended compute target(s)
- Foundry Agent Service compatibility (can the framework's agents be deployed into the managed Foundry runtime, or is it self-hosted only)
- Cold-start / packaging notes
- Azure AD / Managed Identity for auth to Azure OpenAI

### 12. Maturity & ecosystem
- GitHub stars and growth
- Release cadence (last 6 months)
- Breaking change posture
- Community size (Discord/Slack, SO answers)
- Production users (public references)

### 13. Documentation quality
- Primary docs URL
- Getting-started path for .NET and TS specifically
- Sample repos
- API reference completeness

### 14. Gaps / deficiencies
Every framework has them. This section is mandatory per-framework.

---

## Scoring rubric

Axes fall into two categories.

**Binary axes** — the fact is either true or false. Qualitative scoring adds noise. Use ✅ / ❌. Nuance (version, maturity, caveat) belongs in the deep dive, not the matrix cell.

Current binary axes:

- Language support: **TypeScript**, **.NET**, **Python**
- **MCP server** (exposing the agent as an MCP server)
- **Streaming**

**Graded axes** — everything else. Use a four-point qualitative scale:

- **Strong** — first-class, idiomatic, documented. Team productive in hours.
- **Workable** — supported but rough. Team productive in days, with some workarounds.
- **Weak** — partial or via unofficial glue. Expect to write plumbing.
- **Missing** — not available; would need to build from scratch or adopt another library.

No numeric score. When a graded axis is effectively binary for the current field (e.g. all four frameworks get "Strong"), consider promoting it to the binary list in the next revision.
