# API Search-then-Update Pattern
<!-- Sources: Report 3 -->

## When to Use
Two-phase API integration: first resolve an entity by identifier (email), then mutate its state. Common for CRM and billing platform integrations.

## Architecture
```
Trigger → HTTP Request (Search by email) → Code/IF (validate results)
    → HTTP Request (Update entity) → Output
```

## Phase 1: Identity Resolution
- Search API endpoint to find entity by non-unique identifier
- Handle: empty results, multiple matches, pagination
- Cache entity ID for Phase 2

## Phase 2: State Mutation
- Update endpoint using entity ID from Phase 1
- Handle: state machine constraints, eventual consistency, metadata hazards

## Concrete Examples

### Stripe (Search + Update Customer)
See `knowledge_base/nodes/http-request.md` → Stripe section
- Search: `GET /v1/customers/search?query=email:'...'`
- Update: `POST /v1/customers/{{id}}` (form-urlencoded)
- Pitfalls: Eventual consistency (up to 1hr lag), email non-uniqueness, metadata deletion hazard

### HubSpot (Search + Update Contact Lifecycle)
See `knowledge_base/nodes/http-request.md` → HubSpot section
- Search: `POST /crm/v3/objects/contacts/search` (JSON body)
- Update: `PATCH /crm/v3/objects/contacts/{{id}}` (JSON body)
- Pitfalls: Forward-only lifecycle DAG, legacy auth deprecated, clear-then-set for backward moves

## Gotchas
- Always handle multiple search results (filter by secondary criteria)
- Add delay between create and search for eventually consistent APIs
- Validate update response before proceeding
- Be aware of state machine constraints (HubSpot lifecycle stages)
