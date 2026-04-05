# Workflow: Explore Patterns and Best Practices

<required_reading>
**Load available patterns:**
1. Glob `knowledge_base/patterns/` to list available pattern documentation
2. Glob `knowledge_base/examples/` to list available example workflows
</required_reading>

<process>

**Step 1: Understand Interest**

Ask the user what kind of pattern they're looking for:
- **Agent patterns**: AI agent with tools, multi-agent orchestration, RAG pipelines
- **Integration patterns**: Connecting specific services, webhook handling, API orchestration
- **Logic patterns**: Branching, looping, error handling, retry strategies
- **Data patterns**: Transformation, aggregation, batch processing, pagination

**Step 2: Search Knowledge Base**

Search `knowledge_base/patterns/` and `knowledge_base/examples/` for relevant content. Present what's available.

If the knowledge base has relevant patterns:
- Summarize the pattern with a conceptual overview
- Show the node sequence and key configurations
- Highlight decision points and trade-offs

If no relevant patterns exist in the knowledge base:
- Tell the user explicitly
- Offer to describe a general approach based on n8n's capabilities
- Suggest which documentation to add to the knowledge base

**Step 3: Adapt to User's Context**

If the user has a specific use case:
- Map the pattern to their requirements
- Identify where customization is needed
- Note any nodes that require additional documentation

**Step 4: Document for Reuse**

If the user wants to save a pattern:
- Offer to write it to `knowledge_base/patterns/` for future reference

</process>

<success_criteria>
This workflow is complete when:
- [ ] User's interest area identified
- [ ] Relevant patterns found or gaps identified
- [ ] Pattern explained with practical application
- [ ] User knows how to adapt it to their use case
</success_criteria>
