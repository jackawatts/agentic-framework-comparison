# Claude Agent SDK

> Anthropic's official agent SDK. Included in this comparison at the user's request despite being Claude-only, because the team wants a fair look at it alongside OpenAI-targeting frameworks. It can target Claude models hosted in **Azure AI Foundry**, but it cannot target OpenAI models on Foundry.

**Research benchmark date:** April 2026.

## TL;DR

| Dimension | Verdict |
|---|---|
| Fit for *OpenAI on Foundry* | **Out of scope** — SDK is Claude-only, no OpenAI or Azure OpenAI adapter. |
| Fit for *Claude on Foundry* | **Strong** — native path via Foundry's Claude model deployments. |
| TS support | Strong (`@anthropic-ai/claude-agent-sdk` v0.2.x) |
| .NET support | **Missing** officially; community port only. Use Messages API client directly if you must go .NET. |
| MCP | Strong; native client (stdio + HTTP + SSE). |
| Skills | Strong; Anthropic Skills (model-invoked markdown) is first-class. |
| Observability | Weak on OTel; rich lifecycle hooks instead. |

## Packages and versions

| Runtime | Package | Version (Apr 2026) |
|---|---|---|
| Python | `claude-agent-sdk` | 0.1.59 |
| TypeScript/Node | `@anthropic-ai/claude-agent-sdk` | 0.2.110 |
| .NET | none official | [community `0xeb/claude-agent-sdk-dotnet`](https://github.com/0xeb/claude-agent-sdk-dotnet) |

Repos:
- [anthropics/claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python)
- [anthropics/claude-agent-sdk-typescript](https://github.com/anthropics/claude-agent-sdk-typescript)

License: Anthropic Commercial Terms (SDK itself is free to use; inference is metered per token).

## Model support

Claude only — the SDK speaks Anthropic-shaped APIs. You cannot point it at an Azure OpenAI deployment or a non-Claude Foundry model.

Supported inference channels:

| Channel | Env / config |
|---|---|
| Anthropic API | `ANTHROPIC_API_KEY` |
| Amazon Bedrock | `CLAUDE_CODE_USE_BEDROCK=1` + AWS creds |
| Google Vertex | `CLAUDE_CODE_USE_VERTEX=1` + GCP creds |
| **Azure AI Foundry** | `CLAUDE_CODE_USE_FOUNDRY=1`, `ANTHROPIC_FOUNDRY_API_KEY`, `ANTHROPIC_FOUNDRY_RESOURCE` |

Azure AI Foundry hosts Claude Opus 4.7, Sonnet 4.6, Haiku 4.5, and Opus 4.1 in East US2 and Sweden Central. Billing is via Azure Marketplace at Anthropic's standard rates.

**Important architectural note:** Foundry "hosts" Claude at the commerce / auth / endpoint layer only — inference runs on Anthropic infrastructure, not Azure compute. Anthropic's own documentation states this directly: *"Claude models run on Anthropic's infrastructure. This is a commercial integration for billing and access through Azure. As an independent processor for Microsoft, customers using Claude through Microsoft Foundry are subject to Anthropic's data use terms."* ([source](https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry)). This is a proxy integration: Microsoft provides the Azure-native endpoint (`https://{resource}.services.ai.azure.com/anthropic/v1/*`), Entra ID / API key auth, and Marketplace billing; requests are forwarded to Anthropic's backend for inference. Response headers carry both an `apim-request-id` (Azure edge) and a `request-id` (Anthropic) as a tell.

Contrast: Azure OpenAI's GPT models run on Microsoft-operated Azure compute. The "Global Standard" deployment-type name is shared, but the inference-compute locus is not.

Practical implications:

- **Latency** — one extra hop; expect ~100–500 ms typical round trips.
- **Data residency** — if your compliance posture requires inference to happen inside your Azure boundary, Claude on Foundry does not satisfy it.
- **Operational** — Azure-native auth, private networking, and logging apply up to the edge; not to the inference step.

Sources: [Claude in Microsoft Foundry (Anthropic docs)](https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry), [Use Claude models in Foundry (MS Learn)](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/how-to/use-foundry-models-claude).

## Tool calling and built-in tools

Rich set of built-in tools with a permission model — this is the SDK's core differentiator vs. bare function calling:

| Tool | Notes |
|---|---|
| Read / Write / Edit | File I/O with precise edit semantics |
| Bash / Monitor | Shell exec + background-process streaming |
| Glob / Grep | Filesystem search |
| WebSearch / WebFetch | Live web access |
| AskUserQuestion | Structured user prompts with choices |
| Agent | Spawn sub-agents |

Permission modes: `auto`, `manual`, `acceptEdits`. Tool allowlist gates what's callable per session.

## MCP

Native MCP client (stdio + HTTP streamable + SSE). MCP servers configured per-session via `options.mcpServers`. Tool search can mask large tool sets from the context window. MCP server support (exposing the agent as an MCP server) is available via the SDK as well.

Docs: [Agent SDK — MCP](https://code.claude.com/docs/en/agent-sdk/mcp).

## Memory

- **Session persistence** — resume a session by ID; full conversation state replays.
- **Context compaction** — built-in; no explicit API needed.
- **Sub-agents** — isolated contexts that report back to the parent.
- **Skills and CLAUDE.md** — filesystem-scoped long-lived context. Not a database, but effective as a "project memory" primitive.

No built-in semantic-memory store or Cosmos DB / Azure AI Search integration. Long-term memory is BYO (wire a Postgres + pgvector behind a tool, or similar).

## Skills

Markdown-based capability bundles in `.claude/skills/*/SKILL.md`. The model is shown skill metadata (name + description) at session start and invokes them on demand. This is **progressive-disclosure** by design — content doesn't hit the context window until activated.

Paired with custom slash commands (`.claude/commands/*.md`) and sub-agents for composable agent behaviours.

## Multi-agent

- Sub-agents with custom system prompts and tool allowlists.
- `parent_tool_use_id` propagation for tracing hierarchy.
- No explicit worktree sandbox between sub-agents (they share working directory) except via the `Agent` tool's `isolation: "worktree"` option.

No "workflow graph" primitive — orchestration is via sub-agent spawn + the parent agent's reasoning.

## Observability

- Lifecycle **hooks**: `PreToolUse`, `PostToolUse`, `Stop`, `SessionStart`, `SessionEnd`, `UserPromptSubmit`.
- Streaming events expose tool use and tokens.
- **OpenTelemetry is not first-class** — you integrate via hooks. This is the main observability gap for enterprise Azure teams expecting Azure Monitor / App Insights out of the box.

## Guardrails

- Permission modes and tool allowlists are the primary guardrail.
- Hooks can block, modify, or log tool invocations.
- No built-in content-safety filter; pair with Azure AI Content Safety via a `PreToolUse` hook if you adopt this in Azure.

## Streaming

Token-level and tool-event streaming in both Python and TS via async iterables.

## Deployment on Azure

If you commit to Claude on Foundry:

1. Provision a Foundry resource in East US2 or Sweden Central.
2. Deploy Claude Opus 4.7 (or chosen model) as Global Standard.
3. Set `CLAUDE_CODE_USE_FOUNDRY=1`, `ANTHROPIC_FOUNDRY_API_KEY`, `ANTHROPIC_FOUNDRY_RESOURCE`.
4. Host the agent on Azure Container Apps / App Service / Functions.

Latency note: agent → Foundry Claude endpoint → Anthropic inference crosses public internet. ~100–500ms. Fine for most agent workflows, not fine for sub-100ms interactive use.

## Cost

- SDK: free.
- Inference: metered per token — same rates on Foundry as on Anthropic direct, billed through Azure Marketplace.

## Gaps / deficiencies

1. **No OpenAI / Azure OpenAI path** — disqualifier if OpenAI models are a hard requirement.
2. **No official .NET SDK** — community port only; use Messages API client directly if .NET is required.
3. **No built-in OpenTelemetry** — integrate via hooks, which is more work than flipping a flag.
4. **No built-in long-term memory store** — BYO.
5. **Foundry hosting is Anthropic-side, not Azure-side compute** — latency and data-residency implications worth reviewing with compliance.
6. **Foundry regions limited** to East US2 and Sweden Central as of April 2026; Australia-East not yet supported.

## References

- [Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview)
- [Quickstart](https://code.claude.com/docs/en/agent-sdk/quickstart)
- [TypeScript API reference](https://code.claude.com/docs/en/agent-sdk/typescript)
- [Python API reference](https://code.claude.com/docs/en/agent-sdk/python)
- [MCP integration](https://code.claude.com/docs/en/agent-sdk/mcp)
- [Sub-agents](https://code.claude.com/docs/en/agent-sdk/subagents)
- [Skills](https://code.claude.com/docs/en/agent-sdk/skills)
- [Permissions](https://code.claude.com/docs/en/agent-sdk/permissions)
- [Hooks](https://code.claude.com/docs/en/agent-sdk/hooks)
- [Claude in Microsoft Foundry](https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry)
- [Configure Claude Code on Foundry](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/how-to/configure-claude-code)
- [Deploy Claude models in Foundry](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/how-to/use-foundry-models-claude)
- [anthropics/claude-agent-sdk-demos](https://github.com/anthropics/claude-agent-sdk-demos)
