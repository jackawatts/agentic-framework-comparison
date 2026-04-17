# LangGraph (and LangChain)

**Research benchmark date:** April 2026.

## LangGraph vs LangChain — the distinction

These are two libraries from the same team. Use both, for different things:

| Library | Role | When to reach for it |
|---|---|---|
| **LangGraph** | Low-level, durable, stateful **orchestration runtime**. Graphs of nodes + edges with checkpointing, human-in-the-loop, streaming. | Custom control flow, subgraphs, multi-agent, anything where you care about persistence / resume semantics. |
| **LangChain** | Higher-level framework of **model abstractions, integrations, and prebuilt agent patterns**, built *on top of* LangGraph. | Standard ReAct / tool-calling agents; quickly assembling model + tools + memory. |

**Both are actively developed.** What *is* deprecated:

- Legacy `AgentExecutor` / `initialize_agent` — gone in LangChain 1.0 (Oct 22, 2025).
- **LCEL pipe chains (`prompt | llm | parser`)** — no longer recommended; legacy Runnables moved to `langchain-classic`.
- `create_react_agent` from `langgraph.prebuilt` — superseded by `create_agent` from `langchain.agents` (middleware-based).

Recommended split in 2026: use `create_agent` from LangChain for standard tool-calling agents; drop to raw LangGraph `StateGraph` when you need custom flow, subgraphs, or multi-agent orchestration.

Docs: [LangChain 1.0 GA announcement](https://changelog.langchain.com/announcements/langchain-1-0-now-generally-available), [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview).

## TL;DR

| Dimension | Verdict |
|---|---|
| Fit for Azure OpenAI / Foundry | **Strong** — first-party `langchain-azure-ai` with Entra ID auth |
| TypeScript support | **Strong** — `@langchain/langgraph` v1.8.9, full feature parity |
| .NET support | **Missing** — community `tryAGI/LangChain` is stale (v0.15.0 from June 2024), no LangGraph equivalent |
| Python support | Strong — `langgraph` v1.1.6 |
| MCP | Client only (stdio + HTTP / SSE) via `langchain-mcp-adapters` |
| Memory | Strong — checkpointers (Postgres, SQLite, Redis, Cosmos DB community) + long-term Store |
| Observability | **Good** — first-class OTel export, no LangSmith required |
| Guardrails | Workable — middleware pattern + built-in PII / HITL |

## Packages and versions

| Runtime | Package | Version (Apr 2026) |
|---|---|---|
| Python | `langgraph` | **1.1.6** (1.1.7a2 alpha Apr 14). Python 3.10+, includes 3.14 |
| Python | `langchain-core` | 1.x stable; `1.3.0a3` alpha Apr 16 |
| TypeScript | `@langchain/langgraph` | **1.8.9** (Apr 16, 2026) |
| TypeScript | `@langchain/core` | 1.x stable |
| .NET | — | community `tryAGI/LangChain` stale (June 2024) |

Repos (all MIT):
- [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph) — 29.5k stars
- [langchain-ai/langgraphjs](https://github.com/langchain-ai/langgraphjs) — 2.8k stars, 291 releases
- [langchain-ai/langchain](https://github.com/langchain-ai/langchain) — 134k+ stars, 1,211 releases

## Install commands

**Python:**
```bash
pip install -U langgraph langchain langchain-azure-ai azure-identity
pip install langgraph-checkpoint-postgres langgraph-checkpoint-sqlite
pip install langchain-mcp-adapters
```

**TypeScript:**
```bash
npm install @langchain/langgraph @langchain/core @langchain/openai
npm install @langchain/mcp-adapters
```

## .NET teams

Do not use LangChain/LangGraph .NET. The community port `tryAGI/LangChain` is MIT but at v0.15.0 from June 2024 — stale, no LangGraph equivalent. Pick [Microsoft Agent Framework](./microsoft-agent-framework.md) for .NET, or run LangGraph in a Python/TS sidecar behind an HTTP API.

## Azure OpenAI / Foundry support

First-class via **`langchain-azure-ai`** (v2, Foundry SDK) — Microsoft-authored, Microsoft-documented. Supports Azure OpenAI, DeepSeek, Mistral, Phi, Cohere via OpenAI-compatible APIs. Entra ID auth via `DefaultAzureCredential`.

```python
from langchain.chat_models import init_chat_model
from azure.identity import DefaultAzureCredential
model = init_chat_model("azure_ai:gpt-4.1", credential=DefaultAzureCredential())
```

Set `AZURE_AI_PROJECT_ENDPOINT` for Foundry project routing.

**Migration note:** Azure AI Inference SDK namespaces (`AzureAIChatCompletionsModel`) are deprecated and **retire 2026-05-30** — migrate to OpenAI / v1-compatible `AzureAIOpenAIApiChatModel`. Alternative: plain `langchain-openai`'s `AzureChatOpenAI`.

Docs: [MS Learn — LangChain on Foundry](https://learn.microsoft.com/en-us/azure/foundry/how-to/develop/langchain-models), [langchain-azure repo](https://github.com/langchain-ai/langchain-azure).

## Tool calling

`model.bind_tools([...])` returns a runnable emitting tool-call messages.

High-level: `create_agent(model, tools, system_prompt, checkpointer, middleware=[...])` from `langchain.agents` (the 1.0 replacement for `create_react_agent`). Tools are plain decorated Python functions or `@tool`-decorated callables.

## MCP

**Client only, no server adapter.**

- Python: [`langchain-mcp-adapters`](https://pypi.org/project/langchain-mcp-adapters/)
- TS: [`@langchain/mcp-adapters`](https://www.npmjs.com/package/@langchain/mcp-adapters)
- `MultiServerMCPClient` supports stdio + HTTP / SSE transports, converts MCP tools to LangChain tools.

To expose a LangGraph agent *as* an MCP server, use FastMCP separately.

Docs: [MCP integration](https://docs.langchain.com/oss/python/langchain/mcp).

## Memory

- **Short-term (thread)** — checkpointers.
  - Built-in: `InMemorySaver`, `SqliteSaver`.
  - First-party: `langgraph-checkpoint-postgres`.
  - Official community: [`langgraph-redis`](https://github.com/redis-developer/langgraph-redis) (Redis-maintained), MongoDB (JS).
- **Long-term (cross-thread)** — `Store` interface.
  - `InMemoryStore`, `PostgresStore`, `RedisStore`.
- **Azure-native** — [`langgraph-checkpoint-cosmosdb`](https://pypi.org/project/langgraph-checkpoint-cosmosdb/) (**community PyPI, not officially LangChain-maintained**); paired with the Azure sample repo [AzureCosmosDB/multi-agent-langgraph](https://github.com/AzureCosmosDB/multi-agent-langgraph). Also `langchain-azure-cosmosdb` and `langchain-azure-postgresql` for vector stores.

## Graph primitives

- `StateGraph` — typed state (TypedDict / Pydantic).
- `add_node`, `add_edge`, `add_conditional_edges`.
- Compiled subgraphs as nodes.
- `Send` API — map / fan-out to parallel node instances with per-instance payloads.
- `Command` — node return that both updates state and routes; replaces many conditional edges; also enables parent-graph navigation from subgraphs.
- `interrupt` for human-in-the-loop.

Docs: [Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api).

## Multi-agent orchestration

- **Supervisor** — [`langgraph-supervisor`](https://pypi.org/project/langgraph-supervisor/) / `@langchain/langgraph-supervisor`. Hierarchical.
- **Swarm** — `langgraph-swarm`. Handoff-based, same-level agents.
- **2026 maintainer guidance:** build supervisor patterns with plain tool-calling / handoff tools for more context-engineering control, rather than helper libraries.

## Observability

- **LangSmith** is paid/proprietary; optional.
- **OTel export is first-class.** Set `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_HEADERS`, `LANGSMITH_OTEL_ONLY=true` to bypass LangSmith entirely. Requires `langsmith>=0.4.25` ([OTel docs](https://docs.langchain.com/langsmith/trace-with-opentelemetry)).
- **Azure Monitor / Foundry tracing** — standard OTLP, no special integration needed.
- Alternatives: Langfuse (MIT, self-hostable), Arize Phoenix, OpenLLMetry.

## Guardrails

- Middleware (`@before_agent` / `@after_agent` hooks) in `create_agent`, or explicit guardrail nodes in a `StateGraph`.
- LangChain 1.x ships built-in **PII-detection middleware** and **HITL middleware**.
- Content safety: integrate Azure AI Content Safety (`langchain-azure-ai` includes tools), Guardrails AI, or NeMo Guardrails.

Docs: [Guardrails](https://docs.langchain.com/oss/python/langchain/guardrails).

## Streaming

`graph.stream(..., stream_mode=)` supports:
- `values` — full state each step
- `updates` — state deltas
- `messages` — token-level + metadata
- `custom` — user emissions
- `debug`
- combinations via list

`astream_events()` exposes fine-grained events (LLM start/stream/end, tool start/end, node enter/exit).

## Deployment on Azure

**OSS is fully self-hostable and Azure-runnable with no LangChain paid dependency.**

- **Azure Container Apps** (recommended) — MS has a [sample guide](https://learn.microsoft.com/en-us/azure/app-service/tutorial-ai-agent-web-app-langgraph-foundry-python).
- **Azure App Service** (Python + Node.js tutorials).
- **AKS** for heavy workloads.
- **LangGraph Platform** (paid, optional) — hosted Cloud / Hybrid / Self-Hosted Enterprise. Self-Hosted Dev free up to 100k node executions/month. Production Self-Hosted requires Enterprise contract ([pricing](https://www.langchain.com/pricing-langgraph-platform)).

To self-host without the Platform: FastAPI / Express wrapping a compiled graph + Postgres / Cosmos checkpointer + OTel exporter. No license or call-home.

## Cost

LangGraph + LangChain OSS: MIT, fully free, no call-home, no mandatory cloud service. LangSmith (observability) and LangGraph Platform (hosted runtime) are optional paid add-ons. A self-hosted deployment with OTel + Azure Monitor needs neither.

## Gaps / deficiencies

1. **No viable .NET path** — community port stale; TS is the language of choice on the Microsoft stack if you want LangGraph.
2. **Churn** — 1.0 migration broke many tutorials; `create_react_agent` → `create_agent`; LCEL chains dropped; `langgraph-prebuilt` 1.0.9+ causes `ExecutionInfo` ImportErrors with older pins.
3. **Security** — CVE-2025-67644 SQL injection in SQLite checkpointer metadata filtering — patch to latest; don't use SQLite for production anyway.
4. **Production gaps** — no built-in retries / fallbacks / CI-CD; you assemble.
5. **Heavy fan-out** — multi-agent scale/parallelism is not LangGraph's strength; Temporal / Ray preferred.
6. **Debugging UX** — graph introspection is weak without LangSmith; OTel helps but is inferior.
7. **Memory coherence** — cross-session Store design is easy to get wrong.
8. **Cosmos DB checkpointer** is community, not first-party LangChain.
9. **No MCP server adapter** — expose via FastMCP separately.
10. **Docs fragmentation** — docs.langchain.com, reference.langchain.com, blog.langchain.com, changelog.langchain.com still cross-link; version-specific lookup is fiddly.

## References

- [docs.langchain.com overview](https://docs.langchain.com/oss/python/langgraph/overview)
- [LangGraph repo](https://github.com/langchain-ai/langgraph) · [LangGraph JS](https://github.com/langchain-ai/langgraphjs)
- [langchain-azure](https://github.com/langchain-ai/langchain-azure)
- [MS Learn — LangChain Foundry models](https://learn.microsoft.com/en-us/azure/foundry/how-to/develop/langchain-models)
- [MCP adapters](https://docs.langchain.com/oss/python/langchain/mcp)
- [OTel + LangSmith](https://docs.langchain.com/langsmith/trace-with-opentelemetry)
- [LangGraph Platform pricing](https://www.langchain.com/pricing-langgraph-platform)
- [Cosmos DB checkpointer (community)](https://pypi.org/project/langgraph-checkpoint-cosmosdb/)
- [tryAGI/LangChain (.NET port, stale)](https://github.com/tryAGI/LangChain)
