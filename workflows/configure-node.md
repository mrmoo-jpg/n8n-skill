# Workflow: Configure a Specific Node

<required_reading>
**Search the knowledge base for the requested node:**
1. Grep `knowledge_base/nodes/` for the node name
2. Read the full documentation file for that node
3. If not found, inform the user and ask them to add the documentation
</required_reading>

<process>

**Step 1: Identify the Node**

Ask the user:
- Which node do you need to configure? (exact name or general description)
- What should this node accomplish in your workflow?
- What data is it receiving from the previous node?

Search `knowledge_base/nodes/` for matching documentation.

**Step 2: Review Documentation**

Read the node's documentation from the knowledge base. Extract:
- All available parameters and their types
- Required vs optional fields
- Available operations/actions
- Expression syntax specific to this node
- Known limitations or gotchas

**Step 3: Provide Configuration**

Present the configuration with:
- **Required parameters**: Every field that must be set
- **Recommended parameters**: Optional fields relevant to the user's use case
- **Expressions**: Any dynamic values using n8n expression syntax (`{{ }}`)
- **Credentials**: If the node requires authentication, specify the credential type

For AI/Agent nodes, additionally specify:
- Model and model parameters (temperature, max tokens)
- System prompt content
- Tool connections and their configuration
- Memory type and settings (window, token, summary)
- Output parsing if applicable

**Step 4: Error Handling**

Based on the node documentation, recommend:
- Whether to enable "Continue on Fail"
- Error output branch configuration
- Retry settings if applicable
- Common failure modes and how to handle them

**Step 5: Verify and Test**

Suggest:
- How to test this node in isolation
- What sample input data to use
- Expected output structure

</process>

<success_criteria>
This workflow is complete when:
- [ ] Node documentation found and reviewed
- [ ] All required parameters specified
- [ ] Expressions use correct n8n syntax
- [ ] Error handling addressed
- [ ] User understands how to test the node
</success_criteria>
