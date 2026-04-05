# n8n Expression Reference

## Data Access

- `{{ $json.field }}` -- Access current item's JSON property
- `{{ $json.body.message }}` -- Access nested property
- `{{ $node["NodeName"].json.field }}` -- Access specific node's output (legacy syntax)
- `{{ $('NodeName').item.json.property }}` -- Named source reference (proxy-based)
- `{{ $('NodeName').first().json.field }}` -- First item from named node (safe for batch)
- `{{ $('NodeName').last().json.field }}` -- Last item from named node
- `{{ $('NodeName').all()[index].json.field }}` -- Explicit index access (required for agent sub-node batch processing)

## Pagination

- `{{ $pageCount * <limit> }}` -- Offset calculation (e.g., `{{ $pageCount * 50 }}`)

## Type Casting & Parsing

- `{{ JSON.parse($fromAI('fieldName')) }}` -- Force strict object casting from agent output
- `{{ JSON.stringify($json.past_steps) }}` -- Serialize array to string for LLM prompt
- `{{ new Date(Number($json.body.timestamp)).toLocaleString("pt-BR") }}` -- Timestamp casting
- `{{ encodeURIComponent(query) }}` -- URL-encode query parameters

## Fallback / Defensive

- `{{ $json.field || "default value" }}` -- Logical OR fallback for nullable fields
- `{{ $json.body.message || $json.body.buttonsResponseMessage.message || "No message" }}` -- Multi-level fallback chain

## Execution Context

- `{{ $execution.id }}` -- Current execution ID (useful for session_id)
- `{{ $itemIndex }}` -- Current item index in batch processing

## Array Manipulation (Code Node)

- `current_plan.slice(1)` -- Remove first element (pop completed step)
- `past_steps.push({ step, result })` -- Append to history ledger
- `$crypto.createHash('sha256').update(JSON.stringify(payload)).digest('hex')` -- Idempotency hash
