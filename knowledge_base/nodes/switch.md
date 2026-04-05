# Switch Node
<!-- Node type: n8n-nodes-base.switch -->
<!-- Sources: Report 2 -->

## Overview
Multi-path conditional router. Evaluates expressions against rules to route data down specific branches. Critical for FSM (Finite State Machine) patterns in agentic workflows.

## Key Pattern: FSM Conditional Edge

Used as the decision brain in Plan-and-Execute workflows:

### Rule Configuration
| Rule | Expression | Route To |
|------|-----------|----------|
| Objective Fulfilled | `{{ $json.final_response }}` is not empty | Output Node (end workflow) |
| Failsafe Termination | `{{ $json.iteration_count }}` >= 10 | Error Trigger (prevent runaway) |
| Objective Unmet (Loop) | `{{ $json.final_response }}` is empty | Merge Node (loop back to Executor) |

### Architecture Note
- The Switch node enforces deterministic routing — the LLM never controls execution paths directly
- It only reads data variables output by the LLM; hardcoded rules enforce structural integrity
- Prefer Switch over IF node for agentic workflows (multiple paths needed)
