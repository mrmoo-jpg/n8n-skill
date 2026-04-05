# AI Agent Node
<!-- Node type: @n8n/n8n-nodes-langchain.agent -->
<!-- Sources: Reports 1, 4 -->

## Overview
The AI Agent node is the cognitive routing engine for n8n's LangChain integration. It dynamically decides which tools to invoke, parses responses, and iteratively reasons toward objectives. Available as ReAct, Plan-and-Execute, and other agent types.

## Version Selection
See references/agent-nodes.md for the v2.2 vs v3.0 decision matrix.

**Key rule**: Default to v2.2 unless using flagship models (GPT-4o, Claude) on n8n >= 2.3.5.

## Known Bugs & Workarounds

### 1. Tool Hallucination Loops (String Injection)
- **Symptom**: Agent generates plaintext `[Used tools:...]` instead of executing tools; forgets prior tool calls
- **Root Cause**: v3.0 injects tool history as hardcoded strings into context rather than native ToolMessage objects. RLHF models (DeepSeek-V3, Qwen 2.5, Llama 3) interpret strings as few-shot examples and mimic them. Legacy memory nodes omit `tools_call` array entirely.
- **Workaround**: Decouple memory -- Redis DB 1 for agent internal memory, Redis DB 2 for user-facing chat. OR downgrade to Agent v2.2.
- **References**: GitHub #23819, #22112

### 2. Context Window Truncation Bypass (Streaming)
- **Symptom**: Postgres Chat Memory ignores `contextWindowLength`; sends unlimited history; triggers 429 Token Rate Limit errors
- **Root Cause**: Webhook with `responseMode: "streaming"` bypasses synchronous memory middleware responsible for truncation
- **Workaround**: Set Webhook to `responseMode: "responseNode"` (disable streaming) OR replace Webhook with Chat Trigger node
- **References**: GitHub #18024

### 3. Session ID Persistence Bug
- **Symptom**: "Clear Session" command fails; agent continues using old session data
- **Root Cause**: Node.js backend and execution cache hold prior state; frontend Session ID doesn't replace backend parameters
- **Workaround**: Force manual workflow re-execution. In production: clear browser localStorage/sessionStorage; remove "pinned" executions from UI
- **References**: Community #217652

### 4. Schema Parsing Failure (Database Tools)
- **Symptom**: `Invalid value for 'content': expected a string, got null`; malformed object insertion errors
- **Root Cause**: Agent generates stringified JSON instead of strictly typed JavaScript objects
- **Workaround**: Disable magic formatting button; use `{{ JSON.parse($fromAI('fieldName')) }}`
- **References**: Community #192004

### 5. MCP Schema Contamination
- **Symptom**: `Unexpected keyword argument` Pydantic validation error from MCP server
- **Root Cause**: n8n merges parent execution context (sessionId, chatInput, toolCallId) into MCP tool parameters via requests-response.ts
- **Workaround**: Upgrade to n8n >= 1.123.14 or >= 2.3.5 (PR #24263). Temp: downgrade to Agent v2.2 or add **kwargs on MCP server.
- **References**: GitHub #21716

### 6. Batch Processing Index Deterioration
- **Symptom**: `ERROR: Multiple matching items for expression`; all batch items get same data
- **Root Cause**: Sub-node proxy handler doesn't propagate batch index; `.item` locks to item[0]
- **Workaround**: Use `.all()[$json.currentIndex].json.property` with Set node injecting `$itemIndex`. OR place Loop node (batch size=1) before Agent.
- **References**: GitHub #27239

### 7. Async Exception Bubbling (Tool Crashes)
- **Symptom**: HTTP Request tool error crashes entire workflow; webhook returns empty body
- **Root Cause**: Tool sub-nodes don't respect "Continue On Fail" -- unhandled promise rejections bubble to highest execution layer
- **Workaround**: Isolate tool in separate sub-workflow (Execute Workflow Tool). OR use Code Node Tool with try/catch wrapper:
```javascript
try {
  const response = await fetch('https://api.example.com/data');
  if (!response.ok) throw new Error(`HTTP Error: ${response.status}`);
  const data = await response.json();
  return JSON.stringify(data);
} catch (error) {
  return JSON.stringify({ status: 'error', message: error.message });
}
```
- **References**: Community #283789

### 8. UI Validation "No prompt specified"
- **Symptom**: `NodeOperationError: No prompt specified` despite valid expression in GUI
- **Root Cause**: GUI validates against item[0] only; later batch items may have null values for the referenced property
- **Workaround**: Use named-node routing: `{{ $('TriggerNode').first().json.query }}`. Add Filter node before Agent to drop null items. Or use fallback: `{{ $json.query || "default prompt" }}`
- **References**: GitHub #24096, #15692

### 9. LLM Formatting Hallucinations (Alt Endpoints)
- **Symptom**: Agent hangs or outputs raw JSON text instead of calling tools when using Cerebras, OpenRouter, or vLLM
- **Root Cause**: V3 prompt formatting causes alt models to misinterpret tool schemas as text generation templates
- **Workaround**: Downgrade to Agent v2.2 (typeVersion: 2.2). Disable "Use Responses API" toggle for endpoints like Cerebras.
- **References**: Reddit, Community #234330

### 10. Custom Code Tool Silent Data Drop
- **Symptom**: Tool: "" in logs despite successful code execution; or `SyntaxError: "[object Object]" is not valid JSON`
- **Root Cause**: LangChain serialization bridge fails on complex objects; silently returns empty string to LLM
- **Workaround**: Always `return JSON.stringify(result, null, 2)` or map to formatted text string. Never return raw arrays/objects.
- **References**: Reddit, GitHub #18051

### 11. Azure OpenAI 404
- **Symptom**: Persistent 404 errors when connecting to Azure OpenAI
- **Root Cause**: Base URL includes `/openai` path that n8n auto-appends, creating duplicate path
- **Workaround**: Set base URL to `https://resource-name.openai.azure.com/` only (strip `/openai`)
- **References**: Community #270156

### 12. Checkpoint Restore State Conflicts
- **Symptom**: Restored agent tries to reuse consumed single-use tokens (e.g., Vault tokens)
- **Root Cause**: System snapshots roll back local memory but can't undo external service actions
- **Workaround**: Design tools that verify external state rather than trusting local memory after interruption

## Error Handling Best Practices
1. Isolate volatile tools in sub-workflows (Execute Workflow Tool)
2. Use Code Node Tool with try/catch for HTTP calls
3. Always stringify custom code tool outputs
4. Add Filter node before Agent to sanitize null data
5. Set explicit timeouts on all HTTP nodes (20-60 seconds)
