# OpenAI Agents SDK

OpenAI's lightweight, production-oriented agent framework. MIT-licensed, official Python and TypeScript packages, no official .NET.

**Research benchmark date:** April 2026.

## TL;DR

| Dimension | Verdict |
|---|---|
| Fit for Azure OpenAI | **Workable** — requires a client shim (`setDefaultOpenAIClient` with `AsyncAzureOpenAI`), but well-trodden |
| TypeScript support | **Strong** — `@openai/agents` v0.8.3, feature parity close to Python |
| .NET support | **Missing** — no official .NET. Community port `incursa/openai-agents-dotnet` only. |
| Python support | Strong — `openai-agents` v0.14.1 |
| MCP | Client only (stdio + streamable HTTP + SSE), plus `HostedMCPTool` for server-side MCP on Responses |
| Memory | Workable — Sessions (short-term) built-in; long-term is BYO |
| Observability | **Weak for Azure** — default tracing exports to OpenAI's dashboard; Azure Monitor path is Pydantic Logfire (Python only) |
| Hosted tools on Azure | Partial — Azure lags OpenAI direct (computer-use preview, shell preview) |

## Packages and versions

| Runtime | Package | Version (Apr 2026) | Install |
|---|---|---|---|
| Python | `openai-agents` | 0.14.1 | `pip install openai-agents` |
| TypeScript / Node | `@openai/agents` | 0.8.3 | `npm install @openai/agents` (Node 22+, Deno, Bun) |
| .NET | — | — | use MAF or community port |

Repos: [openai/openai-agents-python](https://github.com/openai/openai-agents-python) (~21.2k stars, MIT, release 2026-04-15); [openai/openai-agents-js](https://github.com/openai/openai-agents-js) (~2.7k stars, MIT, release 2026-04-06).

Docs: [Python](https://openai.github.io/openai-agents-python/), [TypeScript](https://openai.github.io/openai-agents-js/), [product page](https://developers.openai.com/api/docs/guides/agents-sdk).

## Model support

No Azure-specific model class — you shim the default client:

**Python:**
```python
from openai import AsyncAzureOpenAI
from agents import set_default_openai_client, set_default_openai_api
client = AsyncAzureOpenAI(azure_endpoint=..., api_key=..., api_version=...)
set_default_openai_client(client)
set_default_openai_api("chat_completions")  # or "responses"
```

**TypeScript:** same shape via `setDefaultOpenAIClient(client)` with an Azure-configured `OpenAI` instance ([config guide](https://openai.github.io/openai-agents-js/guides/config/)).

Docs: [Microsoft blog – Azure + APIM with OpenAI Agents SDK](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/use-azure-openai-and-apim-with-the-openai-agents-sdk/4392537), [Python models guide](https://openai.github.io/openai-agents-python/models/).

**Responses API on Azure** is GA and supports code interpreter, file search, web search, remote MCP, image gen, computer-use-preview ([Azure Responses docs](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/responses)). `OpenAIResponsesModel` targets it, but features track whichever tools Azure has enabled in the region/model — expect lag behind openai.com.

Known sharp edges on Azure: token-accounting field mismatch (`prompt_tokens` / `completion_tokens` vs `input_tokens` / `output_tokens`); APIM requires unchecking "Subscription required" or adding `Ocp-Apim-Subscription-Key`.

## Tool calling

- **Hosted tools (OpenAI direct):** `WebSearchTool`, `FileSearchTool`, `CodeInterpreterTool`, `ComputerTool`, `ImageGenerationTool`, `HostedMCPTool`, shell (April 2026 addition).
- **On Azure OpenAI (via Responses):** code interpreter, file search, web search (`web_search` recommended), hosted MCP, image gen, computer-use-preview. Not all regions/models have every tool. Computer use is preview.
- **Custom function tools:** Python `@function_tool` decorator auto-derives JSON Schema from type hints; TS `tool({ name, parameters: zodSchema, execute })`.

## MCP

**Client only** — agents consume MCP, do not expose as MCP servers.

Transports:
- `MCPServerStdio`
- `MCPServerStreamableHttp`
- `MCPServerSse` (deprecated upstream)
- `HostedMCPTool` delegates to OpenAI / Azure Responses server-side MCP.

Spec version not pinned in docs; aligned with recent MCP (streamable HTTP current). Verify at runtime against target server.

Docs: [Python](https://openai.github.io/openai-agents-python/mcp/), [TS](https://openai.github.io/openai-agents-js/guides/mcp/).

## Memory / sessions

Per-conversation state via Sessions ([Python docs](https://openai.github.io/openai-agents-python/sessions/)). Built-ins:

- `SQLiteSession`
- `OpenAIConversationsSession`
- `OpenAIResponsesCompactionSession`

Extensions: `RedisSession`, `SQLAlchemySession`, `DaprSession`, `EncryptedSession`, `AsyncSQLiteSession`. Implement `SessionABC` for custom backends (e.g. Cosmos DB).

**Long-term / semantic memory is BYO** — wire Azure AI Search via `file_search` tool, or build a custom memory tool backed by Cosmos DB + embeddings.

## Multi-agent / handoffs

`handoffs=[agent, handoff(agent, on_handoff=..., input_type=..., input_filter=...)]` generates a `transfer_to_*` tool.

- Tool name / description override.
- Typed metadata.
- History-filter function.
- `RECOMMENDED_PROMPT_PREFIX` for best results.

Parallel orchestration via `asyncio.gather` over multiple `Runner.run` calls, or the "agents as tools" pattern.

Docs: [Handoffs](https://openai.github.io/openai-agents-python/handoffs/).

## Guardrails

Input / output / tool guardrails with tripwire semantics ([docs](https://openai.github.io/openai-agents-python/guardrails/)):

- **Input guardrails** run before the first agent, parallel by default.
- **Output guardrails** run against the final agent output.
- **Tool guardrails** wrap function tools pre/post.
- Function returns `GuardrailFunctionOutput(output_info, tripwire_triggered)`; raising `InputGuardrailTripwireTriggered` / `OutputGuardrailTripwireTriggered` halts the run.

No built-in content-safety — pair with Azure AI Content Safety via a tool-guardrail.

## Observability / tracing

Built-in tracing covers agent runs, LLM calls, tools, handoffs, guardrails, audio.

**Default destination is OpenAI's trace dashboard** — requires an `OPENAI_API_KEY`. On Azure-only deployments:

- Disable: `set_tracing_disabled(True)` or `OPENAI_AGENTS_DISABLE_TRACING=1`.
- Or swap processors: `set_trace_processors([...])` for Langfuse / LangSmith / Braintrust / W&B.

**Path to Azure Monitor / Application Insights:** Pydantic Logfire instrumentation → OTLP → OTel Collector → App Insights ([Microsoft blog](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/monitor-openai-agents-sdk-with-application-insights/4393949)). **Python only** — no documented TS equivalent.

No native OTel exporter — this is the biggest observability gap for Azure teams.

## Streaming

Token and event streaming first-class:
- Python: `Runner.run_streamed()` yields `StreamEvent`s (raw model deltas, run items, agent-updated).
- TS: `run(agent, input, { stream: true })` → async iterator.

## Voice / realtime

`RealtimeAgent` + `RealtimeRunner` (WebSocket / WebRTC; TS adds SIP); `VoicePipeline` (STT → agent → TTS). Target model `gpt-realtime-1.5`. **Azure Realtime is preview with uneven region/model coverage** — treat as Azure-risky for production; expect manual endpoint wiring and partial parity.

## Deployment on Azure

No Azure-specific deployment story from OpenAI. Typical shapes:

- **Azure Container Apps** (recommended for async Python/Node).
- **Azure Functions** for short-lived tool agents; long-running multi-turn streams hit timeout limits — prefer ACA.
- **AKS** for heavy orchestration.

Azure-specific gotchas:

- Managed Identity: construct `AsyncAzureOpenAI` with `DefaultAzureCredential` + bearer token provider, not API keys.
- Trace export requires egress to OpenAI or an OTel collector — firewall accordingly.
- Region vs Responses-tool availability mismatch is common.

## Gaps / deficiencies

1. **No official .NET SDK** — community port only, not recommended.
2. **Azure OpenAI requires a client shim** — not first-party; no Azure-native model class.
3. **Default tracing leaks to OpenAI's dashboard** — must explicitly disable for Azure-only workloads.
4. **No first-party OpenTelemetry exporter** — Azure Monitor integration is community/Logfire glue, Python only.
5. **Hosted tool coverage on Azure trails openai.com** — computer-use and shell are preview/restricted on Azure.
6. **[TS streaming bug #855](https://github.com/openai/openai-agents-js/issues/855)** — `TypeError: terminated` on Responses streaming under load.
7. **Sessions are per-conversation only** — long-term memory requires Azure AI Search / Cosmos DB you wire yourself.
8. **Assistants API deprecated** 2026-08-26; Azure still supports post-sunset, but new work must target Responses ([OpenAI deprecations](https://developers.openai.com/api/docs/deprecations)).
9. **No MCP server support** — you can't expose an agent as an MCP server without extra glue.
10. **Cross-language split** — a TS + .NET team cannot standardise on this SDK; .NET side defaults to MAF with a different mental model.

## Cost

MIT-licensed, free. Cost = Azure OpenAI inference + hosted-tool charges (code interpreter session, file search storage, web search queries, computer use) + Azure infra (ACA / Functions / Monitor). Tracing to OpenAI dashboard is free but requires a non-Azure `OPENAI_API_KEY`.

## References

- [Python docs](https://openai.github.io/openai-agents-python/) · [TypeScript docs](https://openai.github.io/openai-agents-js/)
- [Sessions](https://openai.github.io/openai-agents-python/sessions/)
- [Handoffs](https://openai.github.io/openai-agents-python/handoffs/)
- [MCP (Python)](https://openai.github.io/openai-agents-python/mcp/) · [MCP (TS)](https://openai.github.io/openai-agents-js/guides/mcp/)
- [Guardrails](https://openai.github.io/openai-agents-python/guardrails/)
- [Tracing](https://openai.github.io/openai-agents-python/tracing/)
- [Azure Responses API](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/responses)
- [Azure + APIM integration](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/use-azure-openai-and-apim-with-the-openai-agents-sdk/4392537)
- [App Insights monitoring](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/monitor-openai-agents-sdk-with-application-insights/4393949)
- [Deprecations](https://developers.openai.com/api/docs/deprecations)
