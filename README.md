# Agentic Framework Comparison — Azure AI Foundry

Comparison of production agent frameworks for a team building on Azure AI Foundry. Refreshed weekly by a Claude Code GitHub Action. Benchmark date of the current revision: **April 2026**.

Scope, hard constraints, and the scoring rubric are in [`docs/evaluation-criteria.md`](docs/evaluation-criteria.md). Axes that are genuinely binary (language support, MCP server support, streaming) use ✅ / ❌. Axes with meaningful variation use the four-point scale: **Strong / Workable / Weak / Missing**.

## Candidates

| Framework | Why in | Deep dive |
|---|---|---|
| **Microsoft Agent Framework (MAF) 1.0** | First-party Azure AI Foundry integration for .NET and Python | [docs/frameworks/microsoft-agent-framework.md](docs/frameworks/microsoft-agent-framework.md) |
| **OpenAI Agents SDK** | Official from OpenAI; official TS SDK; lightweight and production-ready | [docs/frameworks/openai-agents-sdk.md](docs/frameworks/openai-agents-sdk.md) |
| **LangGraph (+ LangChain)** | Most popular OSS agent runtime; first-party `langchain-azure-ai`; mature TS | [docs/frameworks/langgraph.md](docs/frameworks/langgraph.md) |
| **Claude Agent SDK** | Anthropic's official SDK; targets Claude-on-Foundry; reference for skills + MCP-native design | [docs/frameworks/claude-agent-sdk.md](docs/frameworks/claude-agent-sdk.md) |

Live frameworks not in scope:

- **Mastra, CrewAI, Pydantic AI, LlamaIndex agents** — TS or Python-only with weaker Azure integration, or narrower feature set than the four above.

Deprecated or retired frameworks are not named anywhere in this repo.

## Comparison matrix

Scoring applies to the framework's *April 2026* state. Links in each cell lead to the relevant section of the deep-dive.

| Axis | [MAF 1.0](docs/frameworks/microsoft-agent-framework.md) | [OpenAI Agents SDK](docs/frameworks/openai-agents-sdk.md) | [LangGraph](docs/frameworks/langgraph.md) | [Claude Agent SDK](docs/frameworks/claude-agent-sdk.md) |
|---|---|---|---|---|
| **TypeScript support** | ❌ | ✅ | ✅ | ✅ |
| **.NET support** | ✅ | ❌ | ❌ | ❌ |
| **Python support** | ✅ | ✅ | ✅ | ✅ |
| **Azure OpenAI** | Strong (first-party) | Workable (client shim) | Strong (`langchain-azure-ai`) | Missing |
| **Azure AI Foundry model catalog** | Strong | Workable | Strong | Strong (Claude models only) |
| **Foundry Agent Service interop** | Strong (both directions) | Weak | Workable (Hosted Agents preview) | N/A |
| **MCP client** | Strong (hosted MCP on Foundry) | Strong (+ HostedMCPTool) | Workable (via `langchain-mcp-adapters`) | Strong (native, stdio + HTTP + SSE) |
| **MCP server (expose agent)** | ❌ (A2A "coming soon") | ❌ | ❌ (FastMCP separately) | ✅ |
| **Memory — short-term** [¹](#short-term-memory-options) | Strong (`AgentSession`, Foundry persistent) | Workable (Sessions: SQLite, Redis, etc.) | Strong (checkpointers) | Strong (sessions, sub-agents) |
| **Memory — long-term / vector** | Strong (Azure AI Search, Cosmos DB, Redis, etc. via `Microsoft.Extensions.VectorData`) | Weak (BYO) | Workable (Cosmos DB community; Postgres/Redis first-party) | Weak (BYO) |
| **Skills / reusable capabilities** | Strong (adopts agentskills.io spec; class-based in .NET) | Weak (manual composition) | Workable (middleware / toolkits) | Strong (Anthropic Skills) |
| **Multi-agent orchestration** [²](#multi-agent-orchestration-options) | Strong (workflows GA .NET; Python orchestrations preview) | Strong (handoffs, agents-as-tools) | Strong (StateGraph, supervisor, swarm) | Workable (sub-agents only) |
| **Observability — OTel native** [³](#observability-options) | Strong (GenAI semconv; Azure Monitor / Foundry OOB) | Weak (Python-only via Logfire; TS has no path) | Strong (OTel export, no LangSmith needed) | Weak (hooks only) |
| **Guardrails — first-party** | Weak (compose via middleware + Azure AI Content Safety) | Workable (input/output/tool tripwires) | Workable (middleware + built-in PII) | Workable (permission model + hooks) |
| **Streaming** | ✅ | ✅ | ✅ | ✅ |
| **Deployment on Azure** | Strong (ACA, Functions, App Service, Foundry Hosted Agents) | Workable (ACA / ACA preferred, Functions OK) | Strong (ACA, App Service, AKS; MS tutorial) | Workable (ACA; inference runs on Anthropic infra) |
| **License / cost** | MIT | MIT | MIT (OSS; LangSmith & Platform paid but optional) | Anthropic Commercial Terms (SDK free; inference metered) |
| **Docs quality** | Strong (MS Learn hub, samples repo) | Strong (dedicated Py + TS sites) | Workable (fragmented across sub-domains) | Strong (single docs hub) |
| **Maturity (stars, cadence)** | ~9.5k stars; weekly releases | 21.2k + 2.7k stars; weekly releases | 29.5k + 2.8k + 134k stars; weekly releases | Weekly releases; new-ish |

Ratings distilled from the deep dives. Dissent in a cell should be resolved by updating both the matrix and the deep dive — never silently.

### Short-term memory options

Terms referenced in the *Memory — short-term* row. One line per primitive.

- **`AgentSession`** (MAF): per-thread session object with a `StateBag` / `state` dict and pluggable `ChatMessageStore`.
- **Foundry persistent** (MAF): Foundry Agent Service stores conversation history server-side — zero local memory wiring.
- **Sessions** (OpenAI Agents SDK): per-conversation state with built-in `SQLiteSession`, `OpenAIConversationsSession`, `RedisSession`, `SQLAlchemySession`, `DaprSession`, `EncryptedSession`; `SessionABC` for custom backends (e.g. Cosmos DB).
- **checkpointers** (LangGraph): persist graph state per thread — in-memory, SQLite, Postgres (first-party), Redis (official community), Cosmos DB (community). Enables resume-after-crash.
- **sessions** (Claude Agent SDK): resumable conversation state by session ID with automatic context compaction.
- **sub-agents** (Claude Agent SDK): isolated conversation context per child agent; reports back to parent. Used for scoped memory, not just task delegation.

### Multi-agent orchestration options

Terms referenced in the *Multi-agent orchestration* row. One line per primitive.

- **workflows** (MAF): graph of typed `Executor`s + `Edge`s with checkpointing, human-in-the-loop, and supersteps — the explicit-control-flow primitive.
- **orchestrations** (MAF): built-in coordination patterns (sequential, concurrent, hand-off, group chat, Magentic). GA on .NET; **preview** in Python.
- **handoffs** (OpenAI Agents SDK): a `transfer_to_*` tool generated from `handoffs=[agent, ...]` that delegates the conversation to another agent.
- **agents-as-tools** (OpenAI Agents SDK): expose an agent as a callable tool so another agent can invoke it — supports parallel fan-out via `asyncio.gather`.
- **StateGraph** (LangGraph): typed state machine with nodes, edges, conditional edges, subgraphs, `Send` (fan-out), `Command` (update-and-route), and `interrupt` for HITL.
- **supervisor** (LangGraph): hierarchical pattern via `langgraph-supervisor` — one coordinator routes tasks to worker agents.
- **swarm** (LangGraph): peer-level pattern via `langgraph-swarm` — agents hand off to each other without a coordinator.
- **sub-agents** (Claude Agent SDK): spawn isolated agents with custom system prompts + tool allowlists; parent delegates via the `Agent` tool. No workflow graph primitive.

### Observability options

Terms referenced in the *Observability* row. One line per option.

- **GenAI semconv**: [OpenTelemetry GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — `gen_ai.*` spans and metrics. The vendor-neutral shape for emitting agent telemetry.
- **Azure Monitor**: Azure's observability platform (Application Insights). Accepts OTLP and gives trace / metric / log querying; works with any OTel exporter.
- **Foundry tracing**: Azure AI Foundry's built-in agent observability view. MAF-built agents appear here with zero extra wiring (`configure_azure_monitor(...)` or .NET OTel builder).
- **OTel export** (LangGraph): first-class OpenTelemetry export via env vars (`OTEL_EXPORTER_OTLP_ENDPOINT`, `LANGSMITH_OTEL_ONLY=true`). Routes to Azure Monitor, Langfuse, Phoenix — no LangSmith required.
- **LangSmith**: LangChain's proprietary paid trace/eval service. Optional; LangGraph runs fine without it.
- **Logfire** (OpenAI Agents SDK): Pydantic-built OTel instrumentation bridge. Current documented path from the SDK to Azure Monitor, **Python only**.
- **hooks** (Claude Agent SDK): lifecycle callbacks (`PreToolUse`, `PostToolUse`, `Stop`, `SessionStart`, etc.). The only built-in extension point for observability — no native OTel exporter, so OTel integration is DIY.

## Recommendation

### TypeScript → **OpenAI Agents SDK**, or **LangGraph JS** for complex graphs

- **OpenAI Agents SDK** (`@openai/agents`) for the smallest, most idiomatic surface against Azure OpenAI Responses. Requires a shim (`setDefaultOpenAIClient(new AsyncAzureOpenAI(...))`). Strong handoffs, tripwire guardrails, hosted tools. Downsides: default tracing leaks to OpenAI's dashboard (disable or swap processor); no native OTel to Azure Monitor from TS; no MCP server support; hosted-tool coverage on Azure trails openai.com.
- **LangGraph JS** (`@langchain/langgraph`) when you need explicit state machines, durable checkpointing, or non-trivial multi-agent orchestration. First-party OTel export (no LangSmith needed); `langchain-azure-ai` is MS-authored. More moving parts than the OpenAI SDK; docs are fragmented across multiple domains.

### .NET → **Microsoft Agent Framework 1.0**

The canonical Azure AI Foundry path. First-party connectors for Azure OpenAI, Foundry Agent Service, hosted MCP, Azure Monitor / Foundry tracing, and all major Azure vector stores. MIT. Samples and docs cover every axis. Class-based skills and DI ergonomics are .NET-leading. Foundry **Hosted Agents** (preview) lets you deploy MAF code straight into a managed Azure runtime.

Known risks to plan around: first-party guardrails are absent (compose Azure AI Content Safety + middleware); A2A 1.0 is not yet GA; Foundry Hosted Agents is preview.

### Python → **Microsoft Agent Framework**, or **OpenAI Agents SDK** for a lighter surface

Python is the only language where all four frameworks are first-class, so the choice is about fit rather than availability.

- **Microsoft Agent Framework Python** (`agent-framework`) for first-party Azure Foundry integration — `FoundryChatClient`, hosted MCP, Foundry Agent Service interop, `configure_azure_monitor(...)` for observability, and vector-store connectors for every Azure data service (Azure AI Search, Cosmos DB, Redis, Postgres). Known risk: the Python `agent_framework.orchestrations` module (Sequential / Concurrent / Hand-off / Group Chat / Magentic) is flagged preview even after 1.0 GA — .NET leads Python here.
- **OpenAI Agents SDK Python** (`openai-agents`) for the largest user base and most idiomatic Responses surface. Azure OpenAI needs the `AsyncAzureOpenAI` + `set_default_openai_client(...)` shim; default tracing leaks to OpenAI's dashboard (disable or swap processor). For Azure Monitor, the documented route is Pydantic Logfire → OTLP → App Insights — this is the one language where that path exists.
- **LangGraph Python** (`langgraph`) when you need explicit state machines, durable checkpointing, or multi-agent supervisor / swarm patterns. Same Azure story as LangGraph JS; Python is the more mature surface of the two.

### Claude Agent SDK — only when the model is Claude

Out of scope for OpenAI models on Foundry — the SDK is Claude-only. In scope if the team ever pivots to Claude models hosted in Foundry (Opus 4.7, Sonnet 4.6, Haiku 4.5 in East US2 / Sweden Central). MCP- and Skills-native; strong permission model and sub-agents. Inference runs on Anthropic infrastructure even when billed via Azure Marketplace — factor the latency (100–500 ms) and compliance posture into any evaluation.

## Deficiencies of the current generation (as of April 2026)

Common gaps across frameworks, worth planning for regardless of choice:

1. **No framework has first-class guardrails.** Every recommendation above defers to Azure AI Content Safety / Guardrails AI / NeMo Guardrails / internal middleware.
2. **MCP server support is consistently weaker than MCP client.** If you need to expose your agents to other agents, plan to bolt on FastMCP or A2A separately.
3. **TS + .NET parity does not exist.** Pick one language and commit.
4. **Long-term memory stores are rarely first-party.** Azure-native Cosmos DB integration exists (MAF first-party; LangGraph community; OpenAI SDK BYO) but expect to design the memory layer explicitly.
5. **Observability is uneven across languages.** MAF and LangGraph Python are strong; OpenAI Agents TS has no documented Azure Monitor path.
6. **Hosted tool coverage on Azure trails openai.com** for anything Responses-API-based.

## References

All sources are linked inline in the per-framework deep dives. The primary hubs:

- [Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/)
- [OpenAI Agents SDK (Python)](https://openai.github.io/openai-agents-python/) · [(TypeScript)](https://openai.github.io/openai-agents-js/)
- [LangChain / LangGraph docs](https://docs.langchain.com/oss/python/langgraph/overview)
- [Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk)
- [Azure AI Foundry](https://learn.microsoft.com/en-us/azure/ai-foundry/)
