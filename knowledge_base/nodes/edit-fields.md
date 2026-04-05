# Edit Fields (Set) Node
<!-- Node type: n8n-nodes-base.set -->
<!-- Sources: Reports 1, 2 -->

## Overview
Maps, renames, and transforms JSON properties. Used extensively for state initialization, reconciliation, and defensive data mapping.

## Key Patterns

### State Initialization (Plan-and-Execute)
Initialize the FSM state schema at workflow start:
- `session_id`: `{{ $json.session_hash }}`
- `objective`: `{{ $json.query }}`
- `current_plan`: `[]` (empty array)
- `past_steps`: `[]` (empty array)
- `final_response`: null
- `iteration_count`: 0

### State Reconciliation (After Planner/Replanner)
Overwrite state variables with LLM outputs:
- `current_plan`: `{{ $json.steps }}` (from Planner)
- `current_plan`: `{{ $json.updated_plan }}` (from Replanner)
- `final_response`: `{{ $json.response }}` (from Replanner)

### Defensive Fallback Mapping (Webhook Data)
For fragile nested JSON from webhooks:
```
{{ $json.body.message || $json.body.buttonsResponseMessage.message || "No message" }}
```

Timestamp casting:
```
{{ new Date(Number($json.body.timestamp)).toLocaleString() }}
```
