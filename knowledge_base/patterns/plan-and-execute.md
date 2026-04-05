# Plan-and-Execute Pattern
<!-- Sources: Report 2 -->

## When to Use
Complex multi-step objectives requiring strategic planning, iterative execution, and adaptive replanning. Replaces brittle ReAct single-agent loops for long-running tasks.

## Architecture
```
Webhook → Code (hash) → Edit Fields (init state)
→ Basic LLM Chain (Planner) → Edit Fields (merge plan)
→ Merge Node ←──────────────────────────────┐
→ Execute Workflow (Executor sub-workflow)    │
→ Code (state mutation)                       │
→ Basic LLM Chain (Replanner)                │
→ Edit Fields (reconcile)                     │
→ Switch Node ─── fulfilled → Output         │
              ├── failsafe → Error Trigger    │
              └── unmet ──────────────────────┘
```

## Global State Schema

| Variable | Type | Description |
|----------|------|-------------|
| session_id | String | `{{ $execution.id }}` or webhook hash — for idempotency |
| objective | String | Immutable original user goal |
| current_plan | Array | Remaining steps; mutate via `Array.slice(1)` after each step |
| past_steps | Array of {step, result} | Historical ledger of actions |
| final_response | String (nullable) | Populated when Replanner determines objective met |
| iteration_count | Integer | Safety counter to prevent runaway loops |

## Node-by-Node Configuration

### 1. Webhook (Trigger)
- HTTP POST with objective in body
- Response mode: "When Last Node Finishes"

### 2. Code Node (Idempotency Hash)
```javascript
const payloadHash = $crypto.createHash('sha256')
  .update(JSON.stringify($input.item.json.body)).digest('hex');
return { json: { session_hash: payloadHash, query: $input.item.json.body.query } };
```
- Check hash against database; skip if duplicate

### 3. Edit Fields (State Init)
- session_id: `{{ $json.session_hash }}`
- objective: `{{ $json.query }}`
- current_plan: `[]`
- past_steps: `[]`
- final_response: null
- iteration_count: 0

### 4. Basic LLM Chain (Planner)
- Model: High-capability (GPT-4o, Claude)
- Require Specific Output Format: ON
- Structured Output Parser: JSON with `steps` array of strings
- System prompt: "Decompose the objective into sequential, actionable sub-tasks. Do not execute. Only output the structured plan."
- User prompt: `{{ $json.objective }}`

### 5. Edit Fields (Merge Plan)
- current_plan: `{{ $json.steps }}`

### 6. Merge Node (Cyclical Entry)
- Input 1: From Planner phase
- Input 2: From Switch node (loop back)

### 7. Execute Workflow (Executor)
- Separate sub-workflow
- Input Data Mode: "Define below"
- Pass: objective + `step_to_execute: {{ $json.current_plan[0] }}`
- Wait for completion: ON
- Sub-workflow contains its own tools, HTTP requests, error handling

### 8. Code Node (State Mutation)
```javascript
let state = $input.all()[0].json;
state.past_steps.push({
  step: state.current_plan[0],
  result: state.executor_result
});
state.current_plan = state.current_plan.slice(1);
state.iteration_count = state.iteration_count + 1;
return [{ json: state }];
```

### 9. Basic LLM Chain (Replanner)
- Model: High-capability
- Structured Output Parser: `{response: String|null, updated_plan: Array|null}`
- System prompt: "Analyze objective against past_steps. If fulfilled, provide final answer in 'response'. If not, revise remaining steps in 'updated_plan'."
- User prompt: Objective + Past Steps + Remaining Plan

### 10. Edit Fields (Reconcile)
- final_response: `{{ $json.response }}`
- current_plan: `{{ $json.updated_plan }}`

### 11. Switch Node (Conditional Edge)
- Rule 1: `{{ $json.final_response }}` is not empty → Output
- Rule 2: `{{ $json.iteration_count }}` >= 10 → Error Trigger
- Rule 3: `{{ $json.final_response }}` is empty → Merge Node (loop)

## State Persistence
- Use Supabase or PostgreSQL for production
- Upsert state after each iteration
- Enables: crash recovery, idempotency, long-running workflows

## Gotchas
- Do NOT use n8n native Loop node for this — causes execution bloat and memory leaks
- Implement context window pruning before Replanner if past_steps grows large
- Individual HTTP nodes in sub-workflow need explicit timeouts (20-60s)
- Set iteration_count limit as failsafe (e.g., 10)
