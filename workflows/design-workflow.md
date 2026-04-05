# Workflow: Design a New Workflow

<required_reading>
**Before proceeding, search the knowledge base:**
1. Glob `knowledge_base/nodes/` to see available node documentation
2. Glob `knowledge_base/patterns/` to see available workflow patterns
</required_reading>

<process>

**Step 1: Understand the Goal**

Ask the user these questions (skip any already answered):
- What is the **end goal** of this workflow? What should happen when it completes?
- What **triggers** the workflow? (Webhook, schedule, manual, event from another service?)
- What **data sources** are involved? (APIs, databases, files, user input?)
- If AI agents are involved: What should the agent **decide or do**? What tools does it need?

Wait for answers before proceeding.

**Step 2: Whiteboard the Concept**

Outline the workflow as a numbered list of logical steps. Use plain language, not n8n-specific terms yet.

Example format:
```
1. Trigger: Webhook receives incoming customer message
2. Route: Switch node checks message intent (question vs complaint vs order)
3. For questions: Agent node with RAG tool searches knowledge base
4. For complaints: Create ticket in Jira, notify support channel
5. For orders: Look up order in database, confirm status
6. Respond: Send reply back through original channel
```

Present this to the user and ask: **"Does this flow match your goal? Want to add, remove, or reorder anything?"**

Wait for approval before proceeding.

**Step 3: Map to n8n Nodes**

For each step in the approved concept:
1. Search `knowledge_base/nodes/` for the relevant node documentation
2. Identify the exact n8n node type (e.g., `n8n-nodes-base.webhook`, `@n8n/n8n-nodes-langchain.agent`)
3. Note any required credentials, configuration, or known limitations
4. If no documentation exists for a needed node, tell the user explicitly

Present the node mapping as a table:
```
| Step | n8n Node | Key Config | Notes |
|------|----------|------------|-------|
| 1    | Webhook  | POST, path=/incoming | Returns response |
| 2    | Switch   | Expression on $json.intent | 3 outputs |
```

**Step 4: Configure Nodes Sequentially**

For each node in the mapping:
1. Read the full documentation from `knowledge_base/nodes/`
2. Provide the exact configuration (parameters, expressions, credentials)
3. Highlight any error handling needed
4. For AI nodes: specify memory type, tools, system prompt, and model settings
5. Wait for user confirmation before moving to the next node

**Step 5: Connect and Validate**

Once all nodes are configured:
- Describe the connections between nodes
- Identify any missing error handling branches
- Suggest test scenarios
- Note any rate limits or API constraints to watch for

</process>

<success_criteria>
This workflow is complete when:
- [ ] User's goal is clearly understood
- [ ] Conceptual flow is approved by user
- [ ] Each step is mapped to a specific n8n node
- [ ] Node configurations are verified against knowledge base docs
- [ ] Error handling is addressed
- [ ] User has a clear path to implement the workflow
</success_criteria>
