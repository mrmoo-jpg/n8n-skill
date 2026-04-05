# Example: HubSpot Contact Search + Lifecycle Update
<!-- Sources: Report 3 -->

## Goal
Search for a HubSpot contact by email, then update their lifecycle stage and lead status.

## Node 1: HTTP Request (Search)

| Setting | Value |
|---------|-------|
| Method | POST |
| URL | `https://api.hubapi.com/crm/v3/objects/contacts/search` |
| Authentication | Bearer Token |
| Content-Type | `application/json` |
| Timeout | 60000 |

### Auth Header
```
Authorization: Bearer YOUR_ACCESS_TOKEN
```
Token must have scopes: `crm.objects.contacts.read`, `crm.objects.contacts.write`

**Legacy `hapikey` is deprecated** — HubSpot auto-deactivates detected tokens.

### Request Body
```json
{
  "filterGroups": [
    {
      "filters": [
        {
          "propertyName": "email",
          "operator": "EQ",
          "value": "sally@rocketrides.io"
        }
      ]
    }
  ],
  "limit": 10,
  "properties": ["email", "lifecyclestage", "hs_object_id"]
}
```

### Notes
- Max 6 filter groups per search
- `properties` array limits response fields (performance optimization)
- `operator` values: EQ, NEQ, LT, LTE, GT, GTE, BETWEEN, IN, NOT_IN, HAS_PROPERTY, NOT_HAS_PROPERTY, CONTAINS_TOKEN, NOT_CONTAINS_TOKEN

## Node 2: HTTP Request (Update)

| Setting | Value |
|---------|-------|
| Method | PATCH |
| URL | `https://api.hubapi.com/crm/v3/objects/contacts/{{ $json.results[0].id }}` |
| Content-Type | `application/json` |
| Timeout | 60000 |

### Request Body (Forward Progression)
```json
{
  "properties": {
    "lifecyclestage": "customer",
    "hs_lead_status": "CLOSED"
  }
}
```

### Lifecycle Stage Values (ordered)
subscriber → lead → marketingqualifiedlead → salesqualifiedlead → opportunity → customer → evangelist → other

### Lead Status Values
New, Open, In Progress, Open Deal, Unqualified, Attempted to Contact, Connected, Bad Timing

### Critical: Backward Lifecycle Movement
HubSpot enforces a **Directed Acyclic Graph (DAG)** — forward-only by default.

To move a contact BACKWARD (e.g., customer → lead):

**Step 1**: Clear the lifecycle stage
```json
{ "properties": { "lifecyclestage": "" } }
```
Wait for HTTP 200 OK confirmation.

**Step 2**: Set the new (earlier) stage
```json
{ "properties": { "lifecyclestage": "lead" } }
```

A single PATCH trying to go backward will silently fail.

## Pagination (if needed)
- If response includes `paging.next.after`:
- Add `"after": "<token>"` to root of JSON body in next request
- Max `limit=200` per page
- For parallel extraction: subdivide by `hs_object_id` ranges using GT/LT operators
