# Manual Pagination Loop
<!-- Sources: Report 1 -->

## When to Use
Paginating through large API datasets (100+ pages). Replaces built-in pagination which causes OOM crashes.

## Architecture
```
HTTP Request → Code (process + save data) → Code (increment offset) → IF (empty array?)
    ↑                                                                      │
    └──────────────────── YES (more data) ←────────────────────────────────┘
                          NO (done) → next workflow step
```

## Node Configuration

### HTTP Request
- Set `limit` query parameter (e.g., 50)
- Set `offset` from variable (starts at 0)
- Set timeout: 60000

### Code Node (Process + Save)
- Process current batch immediately (write to DB, transform, etc.)
- Clear references to allow garbage collection

### Code Node (Increment)
```javascript
let offset = $input.all()[0].json.offset;
offset = offset + 50; // your page size
return [{ json: { offset } }];
```

### IF Node (Exit Condition)
- Check: returned array length > 0
- TRUE: loop back to HTTP Request
- FALSE: exit loop

## Cursor-Based Variant
For APIs with cursor tokens (Stripe, HubSpot):
- Extract `next_page` / `paging.next.after` from response
- Pass as query param (Stripe) or JSON body field (HubSpot)
- Exit when token is null/absent or `has_more` is false

## Gotchas
- Never use static assignment for offset — must increment dynamically
- Process/save data INSIDE the loop, not after
- Set explicit HTTP timeout to prevent zombie processes
- If using cursor, ensure you handle the first request (no cursor) vs subsequent (with cursor)
