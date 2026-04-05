# Execute Workflow Node
<!-- Node type: n8n-nodes-base.executeWorkflow -->
<!-- Sources: Reports 2, 4 -->

## Overview
Calls a separate n8n workflow as a sub-workflow. Essential for isolating execution environments and containing errors.

## Key Patterns

### Executor Sub-Workflow (Plan-and-Execute)
Isolates tactical execution from strategic planning:
- **Input Data Mode**: "Define below" for strict parameter typing
- **Parameters**: Pass only `objective` and `step_to_execute: {{ $json.current_plan[0] }}`
- **Wait for completion**: ON (synchronous execution)
- Sub-workflow handles its own HTTP requests, tool calls, and error handling
- Returns clean JSON result to parent workflow

### Error Isolation Wrapper (Agent Tools)
Prevents tool failures from crashing the AI Agent:
1. Create standalone sub-workflow with HTTP Request node (standard, not tool)
2. Enable "Continue On Fail" on the HTTP Request
3. On failure, Set node formats error as: `{"status": "error", "message": "..."}`
4. Sub-workflow always returns successfully (either data or formatted error)
5. In parent workflow, use Execute Workflow Tool instead of HTTP Request Tool
6. Agent receives deterministic response and can reason about failures

### Parallel Execution (Callback Coordinator)
For parallel sub-workflows:
1. Generate unique `taskId` per parallel execution
2. Pass `resumeUrl` from Wait node to each sub-workflow
3. Parent suspends at Wait node
4. Sub-workflows complete independently, write results to DB
5. Callback Coordinator verifies all tasks complete, POSTs to `resumeUrl`
6. Parent resumes, queries DB for unified results
