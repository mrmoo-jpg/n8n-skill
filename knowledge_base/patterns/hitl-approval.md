# Human-in-the-Loop (HITL) Approval
<!-- Sources: Report 2 -->

## When to Use
High-risk agent actions requiring human authorization: production DB modifications, financial transactions, external communications.

## Architecture
```
Replanner → Switch (high-risk detected?)
    → YES → Slack/Email (proposed action + context + approve/reject buttons)
           → Wait Node (On Webhook Call)
               → Timeout → Fallback (halt + alert admin)
               → Approved → Route back to Executor
               → Rejected → Halt workflow
    → NO → Continue normal execution
```

## Node Configuration

### Switch Node
- Detect high-risk actions in `updated_plan` (keyword matching or LLM classification)

### Slack/Email Node
- Include: Proposed action, context from `past_steps`, risk assessment
- Embed unique callback URL from Wait node in approval buttons

### Wait Node
- Operation: "On Webhook Call"
- Configure timeout threshold (e.g., 1 hour)
- Timeout branch routes to admin alert + workflow halt

### Post-Approval
- Approved payload flows back to Merge node → Executor cycle continues
- The approved state becomes the next step to execute

## Gotchas
- Always set timeout on Wait node — never let workflows hang indefinitely
- Log all approval/rejection decisions in past_steps for audit trail
- Consider embedding action context directly in the button URL to prevent manipulation
