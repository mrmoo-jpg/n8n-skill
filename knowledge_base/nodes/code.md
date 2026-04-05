# Code Node
<!-- Node type: n8n-nodes-base.code -->
<!-- Sources: Reports 2, 4 -->

## Overview
Executes custom JavaScript within workflows. Essential for state mutation, data transformation, and creating agent tool wrappers.

## Key Patterns

### State Mutation (Plan-and-Execute)
Used to update FSM state between agent iterations:
```javascript
let state = $input.all()[0].json;
let executed_step = state.current_plan[0];
let execution_result = state.executor_result;

state.past_steps.push({
  step: executed_step,
  result: execution_result
});

state.current_plan = state.current_plan.slice(1);
state.iteration_count = state.iteration_count + 1;

return [{ json: state }];
```

### Idempotency Hash
```javascript
const payloadHash = $crypto.createHash('sha256')
  .update(JSON.stringify($input.item.json.body))
  .digest('hex');
return { json: { session_hash: payloadHash, query: $input.item.json.body.query } };
```

### Try/Catch Wrapper (Agent Tool)
When used as a LangChain Code Node Tool, wrap all logic in try/catch:
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

### Serialization Rule
**Always return strings from Code Node Tools** — never return raw objects or arrays:
- Bad: `return targetRecords;` → silent data drop
- Good: `return JSON.stringify(targetRecords, null, 2);`
- Better: `return targetRecords.map(r => 'ID: ' + r.id + ', Name: ' + r.fields.Name).join('\n');`

### Context Window Pruning
Before passing past_steps to Replanner, check size and prune if needed:
```javascript
const MAX_CHARS = 50000;
let steps = $input.all()[0].json.past_steps;
let serialized = JSON.stringify(steps);
if (serialized.length > MAX_CHARS) {
  // Keep only last N steps or summarize
  steps = steps.slice(-5);
}
return [{ json: { ...($input.all()[0].json), past_steps: steps } }];
```
