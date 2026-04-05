# AI Agent Node Reference

## Version Selection: v2.2 vs v3.0

| Factor | v2.2 | v3.0 |
|--------|------|------|
| Tool definition format | Native ToolMessage objects | String injection into context |
| OpenAI-compatible endpoints (Cerebras, OpenRouter, vLLM) | Compatible | Incompatible -- causes formatting hallucinations |
| MCP Client Tool | No parameter bleed (legacy handling) | Parameter bleed -- requires n8n >= 1.123.14 or >= 2.3.5 |
| Multi-turn tool calling | Stable | May cause hallucination loops with RLHF models |
| Responses API support | No | Yes |

**Default recommendation**: Use v2.2 unless you specifically need v3.0 features AND are using flagship models (GPT-4o, Claude) AND are on n8n >= 2.3.5.

## Memory Architecture

**Dual-Memory Pattern (recommended for production)**:
- Redis DB 1: Agent's internal reasoning memory (tool logs, JSON arrays)
- Redis DB 2: User-facing chat interface (clean conversation)
- Prevents: Tool metadata bleeding into user UI, few-shot mimicry hallucinations

**Context Window Truncation Bug**:
- Postgres Chat Memory ignores `contextWindowLength` when Webhook uses `responseMode: "streaming"`
- Fix: Use `responseMode: "responseNode"` OR replace Webhook with Chat Trigger node

## MCP Schema Contamination

- n8n merges parent context (sessionId, chatInput, toolCallId) into tool parameters
- Remote MCP servers with strict Pydantic validation reject the payload
- Fix: Upgrade to n8n >= 1.123.14 (PR #24263 adds schema-based allowlist filtering)
- Temp fix: Downgrade to Agent v2.2, or add `**kwargs` acceptance on MCP server side

## Batch Processing in Agent Sub-Nodes

- `.item` proxy locks to item[0] in batch -- all iterations get same data
- Fix: Use `.all()[$json.currentIndex]` with Set node injecting `$itemIndex`
- Alt fix: Place Loop node (batch size = 1) before Agent node

## Tool Error Handling

- HTTP Request tools inside Agent CANNOT use "Continue On Fail"
- Unhandled exceptions crash the entire workflow
- Fix 1: Isolate tool in separate sub-workflow (Execute Workflow Tool node)
- Fix 2: Use Code Node Tool with try/catch wrapper

## Custom Code Tool Serialization

- Returning raw arrays/objects causes silent data drop (Tool output: "")
- Fix: Always `return JSON.stringify(result, null, 2)` or return formatted text

## Session ID Persistence Bug

- "Clear Session" command fails to reset backend state
- Fix: Force manual re-execution, clear localStorage/sessionStorage, unpin executions

## Schema Parsing (Database Tools)

- Agent outputs stringified JSON instead of typed objects
- Fix: Disable magic formatting button, use `{{ JSON.parse($fromAI('fieldName')) }}`

## LangSmith Integration (Observability)

- Env vars: `LANGCHAIN_TRACING_V2=true`, `LANGCHAIN_ENDPOINT`, `LANGCHAIN_API_KEY`
- Inject `session_id` and `iteration_count` into Tracing Metadata parameter on LLM Chain nodes

## Azure OpenAI

- Base URL must be `https://resource-name.openai.azure.com/` (no trailing `/openai`)
- n8n auto-appends `/openai` -- including it causes 404 from path duplication
