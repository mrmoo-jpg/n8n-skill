# Workflow: Review/Debug an Existing Workflow

<required_reading>
**Search knowledge base as needed during review:**
1. Look up each node type in `knowledge_base/nodes/` to verify configuration
2. Check `knowledge_base/patterns/` for recommended error handling
</required_reading>

<process>

**Step 1: Get the Workflow**

Ask the user to provide the workflow in one of these formats:
- Paste the exported JSON
- Describe the workflow structure (nodes and connections)
- Share a screenshot or node list

**Step 2: Map the Flow**

Reconstruct the workflow as a visual outline:
- List each node with its type and key configuration
- Show the connections and branching logic
- Identify the trigger and all terminal nodes

**Step 3: Check Each Node**

For every node in the workflow:
1. Look up its documentation in `knowledge_base/nodes/`
2. Verify parameters match documented options
3. Check expressions for correct syntax
4. Verify credential types are appropriate
5. Flag any deprecated parameters or approaches

**Step 4: Identify Issues**

Check for common problems:
- **Missing error handling**: Nodes without error branches that could fail
- **Data shape mismatches**: Output of one node doesn't match expected input of the next
- **Expression errors**: Incorrect references, missing fields, wrong syntax
- **Agent issues**: Missing memory, unconfigured tools, vague system prompts
- **Performance**: Unnecessary loops, missing pagination, rate limit risks
- **Security**: Exposed credentials, unsanitized inputs, missing authentication

**Step 5: Recommend Fixes**

For each issue found:
- Explain what's wrong and why it matters
- Provide the specific fix with exact configuration
- Reference the relevant documentation from the knowledge base

</process>

<success_criteria>
This workflow is complete when:
- [ ] Workflow structure is understood and mapped
- [ ] Each node checked against documentation
- [ ] All issues identified with severity
- [ ] Specific fixes provided for each issue
- [ ] User has a clear action plan
</success_criteria>
