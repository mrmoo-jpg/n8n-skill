# Wait Node
<!-- Node type: n8n-nodes-base.wait -->
<!-- Sources: Report 2 -->

## Overview
Suspends workflow execution until a condition is met. Essential for Human-in-the-Loop (HITL) patterns and parallel execution coordination.

## Key Patterns

### Human-in-the-Loop (HITL) Approval
1. Replanner identifies high-risk action in `updated_plan`
2. Switch node routes to notification path
3. Slack/Email node sends proposed action + context + authorization buttons
4. Buttons embed unique callback URL from Wait node
5. Wait node configured with "On Webhook Call" operation
6. Workflow suspends until human clicks approve/reject
7. Configure timeout threshold to prevent indefinite hanging
8. On timeout: route to fallback branch (halt + alert admin)
9. On approval: route back to Executor cycle

### Parallel Execution Coordinator
- Parent workflow enters Wait node after launching parallel sub-workflows
- Wait node generates `resumeUrl` passed to sub-workflows
- Callback Coordinator workflow verifies all parallel tasks complete
- POSTs to `resumeUrl` to wake parent
- Parent queries database for unified results
