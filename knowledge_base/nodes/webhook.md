# Webhook Node
<!-- Node type: n8n-nodes-base.webhook -->
<!-- Sources: Report 1 -->

## Overview
Primary ingress entry point for event-driven workflows. Listens for HTTP requests, parses payloads into JSON, and triggers workflow execution.

## Known Bugs & Workarounds

### 1. Empty Payload (Phantom Payload)
- **Symptom**: Webhook receives POST but body is `{}` despite sender confirmation
- **Root Cause**: Missing `content-type: application/json` header -- Express skips JSON parsing. Or reverse proxy (Nginx) strips payload.
- **Workaround**: Force sender to include `content-type: application/json`. If impossible, check "Binary" tab in execution; use Code node to convert binary buffer to JSON.

### 2. Memory Crash During Webhook Surge
- **Symptom**: Violent memory crashes and container reboots during webhook floods or infinite loops
- **Root Cause**: Default mode writes all payload data directly to V8 memory heap
- **Workaround**: Set environment variable: `N8N_DEFAULT_BINARY_DATA_MODE=filesystem` at Docker container level. Streams payloads to disk, bypasses RAM.
- Add Error Trigger circuit breaker for infinite loop detection.

### 3. CORS Multi-Origin Failure
- **Symptom**: Browser preflight OPTIONS rejected when multiple origins configured
- **Root Cause**: Node returns first statically defined URL in `Access-Control-Allow-Origin` instead of dynamically evaluating caller origin
- **Workaround**: Restrict to single origin per endpoint. For dynamic origins, use reverse proxy (Traefik/Nginx) for CORS negotiation before request reaches webhook.
- **References**: GitHub #18528

### 4. Nested JSON Properties Evaluate to Undefined
- **Symptom**: Data evaluates to `undefined` or `Invalid Date` when mapping from webhook body
- **Root Cause**: Deeply nested properties may be null or missing in specific event variations
- **Workaround**: Use Edit Fields with fallback OR: `{{ $json.body.message || $json.body.buttonsResponseMessage.message || "No message" }}`. Cast timestamps: `{{ new Date(Number($json.body.timestamp)).toLocaleString() }}`

### 5. Streaming Bypasses Output Guardrails
- **Symptom**: PII or hallucinated content streams directly to user without sanitization
- **Root Cause**: `responseMode: "streaming"` pipes LLM chunks directly to TCP socket with zero interception
- **Impact**: Also causes context window truncation bypass (see Agent node bug #2)
- **Workaround**: Use `responseMode: "responseNode"` for sanitizable output. Or build middleware proxy for streaming sanitization.

## Configuration Notes

### Response Modes
| Mode | Behavior | Use When |
|------|----------|----------|
| `responseNode` | Synchronous; waits for Respond to Webhook node | Need to sanitize/process output before sending |
| `lastNode` | Returns output of last node | Simple workflows |
| `streaming` | Real-time chunked output | Chat UIs (but bypasses truncation and sanitization) |

### Environment Variables
- `N8N_DEFAULT_BINARY_DATA_MODE=filesystem` -- Required for production webhook-heavy workloads
