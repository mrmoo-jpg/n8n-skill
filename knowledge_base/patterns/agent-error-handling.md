# Agent Error Handling Patterns
<!-- Sources: Reports 1, 4 -->

## When to Use
Any workflow where AI Agent tools interact with external APIs or services that may fail.

## Pattern 1: Sub-Workflow Isolation

**Problem**: HTTP Request tools inside Agent crash entire workflow on failure.

**Solution**:
1. Create standalone sub-workflow with HTTP Request (standard node, not tool)
2. Enable "Continue On Fail" on HTTP Request
3. On failure: Set node formats `{"status": "error", "message": "..."}`
4. Sub-workflow always returns success (data or error message)
5. Parent: Use Execute Workflow Tool pointing to sub-workflow
6. Agent receives response and can reason about failures gracefully

## Pattern 2: Code Node Try/Catch Wrapper

**Problem**: Same as above, but lighter-weight.

**Solution**: Use LangChain Code Node Tool instead of HTTP Request Tool:
```javascript
try {
  const response = await fetch('https://api.example.com/data');
  if (!response.ok) throw new Error(`HTTP Error: ${response.status}`);
  const data = await response.json();
  return JSON.stringify(data); // MUST stringify
} catch (error) {
  return JSON.stringify({ status: 'error', message: error.message });
}
```

## Pattern 3: Dual-Memory Architecture

**Problem**: Agent tool logs and JSON bleed into user-facing chat.

**Solution**:
- Redis DB 1: Agent internal reasoning (tool logs, raw JSON)
- Redis DB 2: User-facing chat (clean conversation only)
- Prevents few-shot mimicry hallucinations from tool history strings

## Pattern 4: Error Trigger Circuit Breaker

**Problem**: Infinite webhook loops or recursive failures.

**Solution**:
1. Error Trigger node monitors execution failures
2. On repeated failures in narrow time window, flip database flag
3. Flag causes workflow to reject further processing
4. Prevents runaway API billing and resource exhaustion

## Key Rules
- Always stringify Code Node Tool outputs
- Always set HTTP timeouts (20-60 seconds)
- Never rely on Agent's "Continue On Fail" for tool sub-nodes (it doesn't work)
- Add Filter node before Agent to drop null/undefined data items
