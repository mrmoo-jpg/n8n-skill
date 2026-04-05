# Example: Stripe Customer Search + Update
<!-- Sources: Report 3 -->

## Goal
Search for a Stripe customer by email, then update their tax status and metadata.

## Node 1: HTTP Request (Search)

| Setting | Value |
|---------|-------|
| Method | GET |
| URL | `https://api.stripe.com/v1/customers/search` |
| Authentication | Predefined Credential (HTTP Basic Auth) |
| Query Parameters | `query`: `email:'sally@rocketrides.io'` |
| Timeout | 60000 |

### Auth Header (if manual)
```
Authorization: Basic <Base64(sk_test_YOUR_KEY:)>
```
Note the trailing colon before Base64 encoding.

### For Org Keys, add headers:
- `Stripe-Context`: `<Account_ID>`
- `Stripe-Version`: `2024-09-30.acacia`

### Response Handling
- Results in `data` array (may contain multiple customers)
- Check `has_more` for pagination
- Email is NOT unique — add secondary filtering if needed
- **Eventual consistency**: Search index lags up to 1 minute (1 hour during outages)

## Node 2: HTTP Request (Update)

| Setting | Value |
|---------|-------|
| Method | POST |
| URL | `https://api.stripe.com/v1/customers/{{ $json.data[0].id }}` |
| Content-Type | `application/x-www-form-urlencoded` |
| Body Parameters | See below |
| Timeout | 60000 |

### Body (Form-Urlencoded)
```
tax_exempt=exempt
tax[validate_location]=immediately
metadata[internal_status]=premium
```

### Critical Warnings
- `tax_exempt` accepts only: `none`, `exempt`, `reverse`
- To DELETE a metadata key: `metadata[key_name]=` (empty string value)
- **DANGER**: Empty string to root `metadata` parameter erases ALL metadata
- Stripe does NOT support bulk updates — one customer per request
- Updates are partial — omitted fields remain unchanged

## Pagination (if needed)
- First request: no pagination params
- If `has_more` is true: extract `next_page` from response
- Next request: add `page=<next_page>` to query params
- Repeat until `has_more` is false
- Max `limit=100` per page (default 10)
