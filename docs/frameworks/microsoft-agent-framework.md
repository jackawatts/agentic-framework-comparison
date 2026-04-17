# Microsoft Agent Framework (MAF)

Microsoft's unified agent framework. GA **3 April 2026**. First-party path for Azure OpenAI + Azure AI Foundry.

**Research benchmark date:** April 2026.

## TL;DR

| Dimension | Verdict |
|---|---|
| Fit for Azure AI Foundry + OpenAI | **Strong** — first-party connectors, hosted MCP, Foundry Agent Service, Foundry tracing |
| .NET support | **Strong** — `Microsoft.Agents.AI` v1.1.0 stable, idiomatic DI, class-based skills |
| TypeScript support | **Missing** — .NET and Python only. This is the single biggest deficiency for TS teams. |
| Python support | Strong — `agent-framework` v1.0.1 |
| MCP | Strong client; hosted MCP on Foundry first-class |
| Skills | Strong — adopts the open `agentskills.io` spec |
| Observability | Strong — OTel GenAI semconv, native Azure Monitor / Foundry tracing |
| Guardrails | Weak (first-party) — you wire Azure AI Content Safety + middleware yourself |

## Packages and versions

| Runtime | Package(s) | Version (Apr 2026) |
|---|---|---|
| .NET (NuGet) | `Microsoft.Agents.AI`, `Microsoft.Agents.AI.OpenAI`, `Microsoft.Agents.AI.AzureAI`, `Microsoft.Agents.AI.Workflows`, `Microsoft.Agents.AI.Hosting` | **1.1.0** stable / 1.0.0-rc5 preview |
| Python (PyPI) | `agent-framework` (meta), with `agent-framework-core`, `agent-framework-openai`, `agent-framework-foundry` | **1.0.1** stable / 1.0.0rc6 preview |
| TypeScript | — | **none** |

Platform minimums: .NET 8.0 / 9.0 / netstandard2.0 / net472; Python 3.10+.

Repo: [microsoft/agent-framework](https://github.com/microsoft/agent-framework) (~9.5k stars, MIT). Samples: [microsoft/Agent-Framework-Samples](https://github.com/microsoft/Agent-Framework-Samples).

## Model support

| Provider | .NET package | Notes |
|---|---|---|
| Azure OpenAI | `Microsoft.Agents.AI.OpenAI` | First-class |
| OpenAI direct | `Microsoft.Agents.AI.OpenAI` | First-class |
| Azure AI Foundry (model catalog + Foundry Agent Service) | `Microsoft.Agents.AI.AzureAI` | First-class; `AIProjectClient.AsAIAgent(...)` |
| Anthropic (Claude) | via connector | Supported |
| Ollama / Foundry Local | via connector | Local dev |
| Gemini / Bedrock | community | Mentioned in 1.0 blog, no first-party provider page yet |

Docs: [providers page](https://learn.microsoft.com/en-us/agent-framework/agents/providers/).

## Tool calling

.NET: `AIFunctionFactory.Create(MyMethod)` from `Microsoft.Extensions.AI`; typed method → JSON schema auto-generation.
Python: `@tool` decorator + `typing.Annotated` + Pydantic `Field` for descriptions.
Built-in approval workflow: `approval_mode="never_require" | "always_require"` on tools and MCP resources.

## MCP

**Client with first-class hosted-MCP integration on Foundry** ([hosted-mcp-tools docs](https://learn.microsoft.com/en-us/agent-framework/agents/tools/hosted-mcp-tools)):

- Remote MCP via HTTP / streamable HTTP with custom auth headers.
- stdio supported via underlying MCP SDK.
- `RequireApproval "never" | "always"` gate per tool.
- Python rc6 added `structuredContent` (MCP 2025-06 spec).
- Not itself an MCP server, but agents can be exposed via **A2A** (A2A 1.0 support flagged as "coming soon").

## Memory

Two layers, both pluggable:

- **Short-term** — `AgentSession` / `session` with `StateBag` / `state` dict, `ChatMessageStore`. Foundry persistent agents give service-managed history for free.
- **Long-term / retrieval** — `ContextProvider` / `AIContextProvider` API. Built-in `TextSearchProvider` with in-memory vector store. Vector store connectors inherited from `Microsoft.Extensions.VectorData`:
  - **Azure AI Search**
  - **Azure Cosmos DB** (NoSQL + Mongo vCore)
  - **Redis** (incl. Azure Managed Redis)
  - Postgres, Elasticsearch, Qdrant, Weaviate
- Foundry Agent Service provides managed memory ("Memory in Foundry").
- Python 1.0.1 added **Cosmos DB NoSQL checkpoint storage**.
- Mem0 and Neo4j called out as first-party-adjacent.

No explicit "episodic vs semantic" split — primitives are conversational history, persistent key-value, and vector retrieval.

## Multi-agent orchestration

Two primitives ([workflows docs](https://learn.microsoft.com/en-us/agent-framework/workflows/)):

- **Agents** — LLM-driven tool use.
- **Workflows** — graphs of `Executors` + `Edges`, typed messages, checkpointing, human-in-the-loop, supersteps, streaming events.

Built-in patterns: **sequential, concurrent, hand-off, group chat, Magentic (Magentic-One)**.

Caveat: Python `agent_framework.orchestrations` is **flagged preview even after 1.0 GA** ([roadmap discussion #4262](https://github.com/microsoft/agent-framework/discussions/4262)). API may still evolve.

.NET rc5 added durable workflows and return-to-previous routing in handoff workflows.

## Observability

First-class ([observability docs](https://learn.microsoft.com/en-us/agent-framework/agents/observability)):

- OTel **GenAI Semantic Conventions** emitted natively.
- Spans: `invoke_agent <name>`, `chat <model>`, `execute_tool <fn>`.
- Metrics: `gen_ai.client.operation.duration`, `gen_ai.client.token.usage`, `agent_framework.function.invocation.duration`.
- .NET: `.UseOpenTelemetry(...)` + `.WithOpenTelemetry(...)`.
- Python: `configure_otel_providers()` or `opentelemetry-instrument` CLI for zero-code auto-instrumentation.
- Native Azure Monitor / Foundry tracing: `await client.configure_azure_monitor(...)`. Agents show up in [Foundry Observability](https://learn.microsoft.com/en-us/azure/azure-monitor/app/agents-view) with zero extra wiring.

## Guardrails

**No first-party guardrails module.** You compose:

- Azure AI Content Safety for PII / content filters.
- Foundry guardrails control plane ([Foundry guardrails](https://learn.microsoft.com/en-us/azure/foundry/guardrails/guardrails-overview)).
- Middleware pattern: intercept agent runs / tool calls for PII redaction, prompt-injection checks, etc.
- Community OSS `AgentGuard` offers declarative `UseAgentGuard()` middleware for .NET with `RedactPII()`, `CheckGroundedness()` ([AgentGuard](https://filipw.github.io/AgentGuard/)).
- Separate Microsoft repo [Agent Governance Toolkit](https://github.com/microsoft/agent-governance-toolkit) covers policy, identity, sandboxing.

Docs are explicit: you own RAI mitigations, metaprompts, content filters.

## Streaming

Token-by-token and event streaming:
- .NET: `RunStreamingAsync` → `AgentResponseUpdate`.
- Python: `async for update in agent.run(..., stream=True)`.
- Workflow events: `AgentResponseEvent`, `AgentResponseUpdateEvent`.

## Skills

MAF adopts the **open [Agent Skills spec](https://agentskills.io)** — portable `SKILL.md` packages with frontmatter + scripts/references/assets, progressive disclosure (`load_skill`, `read_skill_resource`, `run_skill_script`). Three sources:

1. File-based (filesystem directories).
2. Code-defined (`AgentInlineSkill` / `Skill`).
3. **Class-based** (.NET only, `AgentClassSkill<T>` with `[AgentSkillResource]` / `[AgentSkillScript]` attributes).

`AgentSkillsProviderBuilder` (.NET) mixes sources and filters.

Caveat: `SubprocessScriptRunner` ships but is "demo only" — production needs sandboxing.

## Deployment on Azure

Recommended targets:

- **Azure Functions** (Durable Task integration for long-running workflows).
- **Azure App Service** ([tutorial](https://learn.microsoft.com/en-us/azure/app-service/tutorial-ai-agent-web-app-semantic-kernel-foundry-dotnet)).
- **Azure Container Apps / AKS**.
- **Foundry Agent Service hosted runtime** (preview) — deploy custom MAF code into a managed runtime without containers ([Hosted agents docs](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/hosted-agents)).

Two Foundry interop modes:

1. **Server-side Foundry agent** — `AIProjectClient.AgentAdministrationClient.CreateAgentVersionAsync(...)` (.NET) / `FoundryChatClient` + `as_agent(...)` (Python). State, history, hosted MCP live in Foundry.
2. **Host MAF code inside Foundry's managed runtime** — Hosted agents preview accepts MAF (and LangGraph/CrewAI) custom code.

## Gaps / deficiencies

1. **No TypeScript SDK** — the single largest gap. TS teams must fall back to a Python/.NET service behind a TS front-end, or use a different framework.
2. **Orchestrations preview in Python** — GroupChat / Sequential / Concurrent / Magentic APIs may still evolve post-1.0.
3. **A2A 1.0 not yet GA** — "coming soon" per 1.0 blog.
4. **No first-party guardrails library** — wire Azure AI Content Safety / middleware / AgentGuard yourself.
5. **`SubprocessScriptRunner` for skills is demo-only** — production sandboxing is DIY.
6. **Foundry Hosted agents runtime is preview** — contract still stabilising.
7. **.NET leads Python** on durable workflows, class-based skills, and DI ergonomics; Python catching up.
8. **Gemini / Bedrock** mentioned in the 1.0 blog but no first-party provider pages yet — treat as community connectors.

## References

- [Overview](https://learn.microsoft.com/en-us/agent-framework/overview/)
- [Docs hub](https://learn.microsoft.com/en-us/agent-framework/)
- [Providers](https://learn.microsoft.com/en-us/agent-framework/agents/providers/)
- [Sessions](https://learn.microsoft.com/en-us/agent-framework/agents/conversations/session)
- [Skills](https://learn.microsoft.com/en-us/agent-framework/agents/skills)
- [Observability](https://learn.microsoft.com/en-us/agent-framework/agents/observability)
- [Hosted MCP](https://learn.microsoft.com/en-us/agent-framework/agents/tools/hosted-mcp-tools)
- [Workflows](https://learn.microsoft.com/en-us/agent-framework/workflows/)
- [Foundry hosted agents](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/hosted-agents?view=foundry)
- [1.0 announcement](https://devblogs.microsoft.com/agent-framework/microsoft-agent-framework-version-1-0/)
- [GitHub](https://github.com/microsoft/agent-framework) · [Releases](https://github.com/microsoft/agent-framework/releases) · [Roadmap](https://github.com/microsoft/agent-framework/discussions/4262)
- [NuGet](https://www.nuget.org/packages/Microsoft.Agents.AI/)
- [Python 2026 breaking changes](https://learn.microsoft.com/en-us/agent-framework/support/upgrade/python-2026-significant-changes)
