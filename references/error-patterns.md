# n8n Error Pattern Catalog

## Agent & LLM Errors

### 1. Agent Hallucination Loops
- **Symptom**: Agent output contains literal `[Used tools: ...]` strings instead of actual tool calls; loops repeatedly without progress
- **Root Cause**: v3.0 Agent injects tool definitions as string context instead of native ToolMessage objects; RLHF-tuned models mimic the formatting instead of executing
- **Fix**: Switch to Agent v2.2, which uses native ToolMessage objects. If v3.0 required, use only flagship models (GPT-4o, Claude)

### 2. Context Window Truncation Bypass (Streaming)
- **Symptom**: Postgres Chat Memory loads entire conversation history despite `contextWindowLength` being set; token limit exceeded, degraded responses
- **Root Cause**: Streaming response mode (`responseMode: "streaming"`) bypasses the truncation logic in Postgres Chat Memory
- **Fix**: Set `responseMode: "responseNode"` on the Webhook node, OR replace Webhook with Chat Trigger node

### 3. Session ID Not Clearing
- **Symptom**: "Clear Session" button clicked but agent retains prior conversation context; stale responses persist
- **Root Cause**: Backend session state not invalidated; pinned executions and browser storage preserve old session references
- **Fix**: Force manual re-execution of the workflow, clear browser localStorage/sessionStorage, unpin all executions

### 4. Schema Parsing Failure (got null)
- **Symptom**: Database write operations fail with null values; agent output appears correct in logs but downstream node receives empty/null
- **Root Cause**: Agent returns stringified JSON instead of typed objects; n8n's magic formatting button auto-parses incorrectly
- **Fix**: Disable the magic formatting button on the field. Use `{{ JSON.parse($fromAI('fieldName')) }}` for explicit casting

### 17. LLM Formatting Hallucinations (Alt Endpoints)
- **Symptom**: Agent produces malformed tool calls, markdown artifacts, or repeated formatting strings when using non-OpenAI endpoints (Cerebras, OpenRouter, vLLM)
- **Root Cause**: v3.0 Agent's string-injection tool format is incompatible with OpenAI-compatible API endpoints that lack native tool-calling support
- **Fix**: Use Agent v2.2 with alternative endpoints. Reserve v3.0 for direct OpenAI/Anthropic API connections only

### 18. Custom Code Tool Silent Data Drop
- **Symptom**: Tool node returns empty string ("") to agent despite code executing correctly; agent responds as if tool returned nothing
- **Root Cause**: Code Node Tool silently drops raw array/object return values; only string returns are passed through
- **Fix**: Always `return JSON.stringify(result, null, 2)` or return pre-formatted text from Code Node Tools

## Pagination & Batch Errors

### 5. Pagination OOM
- **Symptom**: Workflow crashes with out-of-memory error during paginated API fetching; n8n process killed
- **Root Cause**: All pages accumulated in memory simultaneously instead of streaming/processing per-page; large datasets (100k+ records) exhaust available RAM
- **Fix**: Process each page within the loop before fetching next. Use batch size limits. Add memory guard: check `process.memoryUsage().heapUsed` and break if threshold exceeded

### 6. Infinite Pagination Loop
- **Symptom**: Pagination loop never terminates; workflow runs indefinitely fetching the same or empty pages
- **Root Cause**: Stop condition checks only for empty response but API returns empty array `[]` with 200 status, or returns same last page repeatedly
- **Fix**: Add dual stop conditions: (1) empty response body AND (2) response length < page size. Include a hard ceiling counter (e.g., max 1000 iterations)

### 7. HTTP Request Hangs (Zombie Processes)
- **Symptom**: HTTP Request node never completes; execution stuck in "running" state indefinitely; requires manual cancellation
- **Root Cause**: No timeout configured on HTTP Request node; remote server holds connection open or drops without RST
- **Fix**: Set explicit timeout on every HTTP Request node (recommended: 30s for APIs, 120s for file downloads). Add retry logic with exponential backoff

### 8. 401 on Subsequent Paginated Requests
- **Symptom**: First paginated request succeeds but subsequent pages return 401 Unauthorized
- **Root Cause**: OAuth token expires mid-pagination; credential node does not auto-refresh between loop iterations
- **Fix**: Add token refresh check before each paginated request. Use credential node's built-in refresh mechanism or add explicit refresh logic at loop start

### 14. Batch Processing Index Deterioration
- **Symptom**: Agent sub-node processes same item repeatedly in batch; all iterations use data from item[0]
- **Root Cause**: `.item` proxy in agent sub-nodes locks to first item; does not iterate with batch index
- **Fix**: Replace `.item` with `.all()[$json.currentIndex]` and inject `$itemIndex` via a Set node before the agent. Alt: place Loop node with batch size = 1 before Agent node

## Webhook Errors

### 9. Webhook Empty Payload
- **Symptom**: Webhook node triggers but `$json.body` is undefined or empty object
- **Root Cause**: Sender uses `Content-Type` header that n8n doesn't auto-parse (e.g., `text/plain`), or body is sent as query params instead of POST body
- **Fix**: Verify sender's Content-Type is `application/json`. If not controllable, add Code node after Webhook to manually parse: `JSON.parse($json.body)` or access `$json.query`

### 10. Webhook Memory Crash (OOM)
- **Symptom**: n8n process crashes when webhook receives large payload (file uploads, base64 images, bulk data)
- **Root Cause**: Entire payload loaded into memory; n8n has no built-in streaming for webhook bodies
- **Fix**: Set `N8N_PAYLOAD_SIZE_MAX` env var to limit accepted payload size. For file uploads, use external storage (S3 pre-signed URL) and pass only the reference URL through the webhook

### 11. CORS Multi-Origin Failure
- **Symptom**: Browser frontend gets CORS errors when calling n8n webhook from multiple domains; works from one domain but fails from others
- **Root Cause**: n8n Webhook node only supports a single origin in CORS configuration; no wildcard or multi-origin list
- **Fix**: Place a reverse proxy (nginx, Caddy) in front of n8n to handle CORS headers. Set `Access-Control-Allow-Origin` dynamically based on request Origin header at the proxy level

### 12. Nested JSON Undefined
- **Symptom**: Expression `{{ $json.body.data.nested.field }}` returns undefined despite payload containing the data
- **Root Cause**: Intermediate property is a string (stringified JSON) not a parsed object; dot-access fails silently
- **Fix**: Parse at each stringified boundary: `{{ JSON.parse($json.body.data).nested.field }}`. Add defensive checks: `{{ $json.body?.data ? JSON.parse($json.body.data).nested.field : "fallback" }}`

## MCP & Integration Errors

### 13. MCP Schema Contamination (Pydantic)
- **Symptom**: MCP tool calls fail with Pydantic validation errors; unexpected fields `sessionId`, `chatInput`, `toolCallId` in request body
- **Root Cause**: n8n merges parent workflow context (session metadata) into MCP tool parameters before sending to remote server
- **Fix**: Upgrade to n8n >= 1.123.14 (PR #24263 adds schema-based allowlist filtering). Temp fix: downgrade to Agent v2.2 (legacy handling avoids bleed), or add `**kwargs` to MCP server tool signatures
- **Versions**: Fixed in n8n >= 1.123.14, >= 2.3.5

### 19. Azure OpenAI 404
- **Symptom**: All Azure OpenAI API calls return 404 Not Found
- **Root Cause**: Base URL includes trailing `/openai` (e.g., `https://myresource.openai.azure.com/openai`); n8n auto-appends `/openai`, causing path duplication
- **Fix**: Set base URL to `https://resource-name.openai.azure.com/` without `/openai` suffix

## Infrastructure & Deployment Errors

### 15. Async Exception Bubbling (Tool Crashes Workflow)
- **Symptom**: Unhandled error in a tool node (HTTP Request, Code Node) crashes the entire agent workflow instead of returning error to agent
- **Root Cause**: HTTP Request tools inside Agent cannot use "Continue On Fail" setting; exceptions bubble up and terminate execution
- **Fix**: Wrap tool logic in a sub-workflow using Execute Workflow Tool node (isolates failures). Alt: use Code Node Tool with explicit try/catch returning error as string to agent

### 16. UI Validation "No prompt specified"
- **Symptom**: Agent node shows validation error "No prompt specified" in UI despite prompt being configured
- **Root Cause**: UI validation does not recognize dynamically set prompts (expressions, connected inputs); only checks static text field
- **Fix**: Enter a placeholder static prompt, then switch to expression mode and set the dynamic value. The validation error is cosmetic -- execution works regardless

### 20. Community Node Docker Build Failure
- **Symptom**: Docker build fails when installing community nodes; `npm install` errors during image build
- **Root Cause**: Community nodes require native dependencies or specific Node.js versions not present in the base n8n Docker image
- **Fix**: Use multi-stage Docker build. Install community nodes in a build stage with full build tools (`node-gyp`, `python3`, `make`), then copy `node_modules` to the slim runtime image. Pin exact node versions in package.json

### 21. Checkpoint Restore State Conflicts
- **Symptom**: Workflow resumes from checkpoint but produces incorrect results; variables contain stale values from pre-checkpoint state
- **Root Cause**: Static/global variables and in-memory caches are not reset on checkpoint restore; only node execution data is restored
- **Fix**: Never store mutable state in static variables or global scope. Use workflow-scoped storage (Set node output, workflow static data via `$getWorkflowStaticData('global')`) that participates in checkpoint serialization. Reset all counters and accumulators explicitly at restoration points
