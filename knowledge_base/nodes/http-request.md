# HTTP Request Node
<!-- Node type: n8n-nodes-base.httpRequest -->
<!-- Sources: Reports 1, 3 -->

## Overview
Primary egress interface for n8n. Handles HTTP calls, authentication, SSL/TLS, and data fetching. Critical for API integrations and paginated data retrieval.

## Known Bugs & Workarounds

### 1. Pagination Memory Exhaustion (OOM)
- **Symptom**: Workflow crashes after several pagination iterations; OS-level OOM container termination
- **Root Cause**: Built-in pagination aggregates ALL page data into V8 heap before passing to next node
- **Workaround**: Abandon built-in pagination. Build manual loop: HTTP Request -> Code node (increment) -> IF node (check empty array) -> process/save data -> loop back. Allows garbage collection between iterations.
- **Expression**: `{{ $pageCount * <limit> }}` (e.g., `{{ $pageCount * 50 }}`)

### 2. Infinite Pagination Loop
- **Symptom**: Node requests same page repeatedly without progressing
- **Root Cause**: Offset expression incorrectly formulated, mapped to wrong parameter type, or uses static assignment
- **Workaround**: In Pagination settings -> "Update a Parameter in Each Request" -> Name: `offset` -> Value: `{{ $pageCount * <limit> }}`

### 3. HTTP Request Hangs (Zombie Processes)
- **Symptom**: Workflow hangs indefinitely; cascading `timeout of 3000ms exceeded` errors on unrelated workflows
- **Root Cause**: Target API drops connection without TCP RST; Node.js async promise stays unresolved in memory
- **Workaround**: Set timeout: 60000 in node settings. Route rejected promise through Error Trigger node.

### 4. 401 Unauthorized on Subsequent Requests
- **Symptom**: First request succeeds; paginated requests fail with "No authorization token provided"
- **Root Cause**: Pagination controller strips Bearer Token from Authorization header after initial validation
- **Workaround**: Update platform to latest stable. OR manually set `Authorization: Bearer <token>` in Header Parameters (don't use Credential UI).
- **References**: GitHub #15043

## Pagination Strategy Comparison

| Strategy | Memory Impact | Risk |
|----------|--------------|------|
| Built-in offset | Aggregates all data into V8 heap | OOM on large datasets |
| Manual IF-loop | GC reclaims per iteration | Infinite loop if increment fails |
| Cursor-based loop | Standard memory usage | Requires API cursor support |

## API Configuration: Stripe

### Authentication
- Method: Basic Auth
- Header: `Authorization: Basic <Base64(sk_test_...:)>` (trailing colon required)
- Org keys require: `Stripe-Context` and `Stripe-Version` headers

### Search (Identity Resolution)
- Endpoint: `GET https://api.stripe.com/v1/customers/search`
- Query: `query=email:'sally@rocketrides.io'`
- URL-encode reserved characters
- Max 100 results per page (default 10)
- **Eventual consistency warning**: Search index lags up to 1 minute (up to 1 hour during degradation). Do NOT use in read-after-write flows.
- Email is NOT unique -- results are arrays requiring secondary filtering

### Update (State Mutation)
- Endpoint: `POST https://api.stripe.com/v1/customers/{{customer_id}}`
- Content-Type: `application/x-www-form-urlencoded`
- Example: `tax_exempt=exempt&metadata[internal_status]=premium`
- **Metadata hazard**: Empty string to root `metadata` key erases ALL metadata. Use `metadata[key]=` to delete specific keys.

### Pagination (Stripe)
- Cursor-based: Extract `next_page` from response, inject as `page` query param
- Terminate when `has_more` is `false`
- Max `limit=100`

## API Configuration: HubSpot

### Authentication
- Method: Bearer Token (OAuth 2.0 Private App)
- Header: `Authorization: Bearer YOUR_ACCESS_TOKEN`
- Scopes: `crm.objects.contacts.read`, `crm.objects.contacts.write`
- **Legacy `hapikey` is deprecated** -- tokens auto-deactivated if detected

### Search (Identity Resolution)
- Endpoint: `POST https://api.hubapi.com/crm/v3/objects/contacts/search`
- Body:
```json
{
  "filterGroups": [{
    "filters": [{
      "propertyName": "email",
      "operator": "EQ",
      "value": "sally@rocketrides.io"
    }]
  }],
  "limit": 10,
  "properties": ["email", "lifecyclestage", "hs_object_id"]
}
```
- Max 6 filter groups, max 200 results per page

### Update (State Mutation)
- Endpoint: `PATCH https://api.hubapi.com/crm/v3/objects/contacts/{{contactId}}`
- Body:
```json
{
  "properties": {
    "lifecyclestage": "customer",
    "hs_lead_status": "CLOSED"
  }
}
```
- **Lifecycle stage DAG constraint**: Forward-only progression by default. To move backward, first clear: `{"properties":{"lifecyclestage":""}}`, wait for 200 OK, then set new stage.
- Valid stages: subscriber, lead, marketingqualifiedlead, salesqualifiedlead, opportunity, customer, evangelist, other

### Pagination (HubSpot)
- Cursor-based: Extract `paging.next.after` from response, inject as `"after"` in JSON body
- Max `limit=200`
- For parallel extraction: subdivide by `hs_object_id` ranges with GT/LT operators
