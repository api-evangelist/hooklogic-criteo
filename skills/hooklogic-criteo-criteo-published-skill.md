---
name: Criteo
description: Use when building integrations with Criteo's Retail Media API or Marketing Solutions API, managing campaigns, audiences, accounts, or analytics programmatically, or troubleshooting API authentication and error responses.
metadata:
    mintlify-proj: criteo
    version: "1.0"
---

# Criteo Developer Portal APIs

## Product summary

Criteo Developer Portal provides two main APIs for programmatic access to advertising platforms: **Retail Media API** (for Commerce Max/Yield users managing retail campaigns) and **Marketing Solutions API** (for Commerce Growth users managing ad sets and audiences). All APIs use OAuth 2.0 authentication, follow consistent REST patterns with JSON payloads, and enforce rate limits (250 calls/min for Client Credentials, 10 calls/min per account for Authorization Code). Base URL: `https://api.criteo.com/<version>/retail-media/` or `https://api.criteo.com/<version>/marketing-solutions/`. Access the full documentation at https://developers.criteo.com.

## When to use

Reach for this skill when:
- Building server-to-server integrations with Criteo's advertising platforms
- Creating or updating campaigns, line items, audiences, or ad sets programmatically
- Pulling performance reports, analytics, or account data
- Implementing OAuth authentication (Client Credentials or Authorization Code flow)
- Debugging API errors, rate limit issues, or authentication failures
- Designing bulk operations for multiple campaigns or line items (max 50 IDs per call)
- Setting up asynchronous operations for catalogs or reports (status/output polling)

## Quick reference

### Authentication endpoints
| Task | Method | Endpoint |
|------|--------|----------|
| Get access token | POST | `https://api.criteo.com/oauth2/token` |
| Token lifetime | — | 900 seconds (15 minutes) |
| Refresh token (Auth Code only) | — | Valid 6 months; revoked if user role changes |

### OAuth grant types
| Type | Use case | Rate limit | Token scope |
|------|----------|-----------|------------|
| Client Credentials | Server-to-server, single data owner | 250 calls/min (default), 40 calls/min (reporting) | Application level |
| Authorization Code | Multi-user, self-service platforms | 10 calls/min per account per user | Account level (scales with consents) |

### API patterns
| Operation | Method | Endpoint pattern |
|-----------|--------|-----------------|
| Create entity | POST | `/{plural-parent}/{parentId}/{plural-entity}` |
| List entities | GET | `/{plural-parent}/{parentId}/{plural-entity}` |
| Get single entity | GET | `/{plural-entity}/{entityId}` |
| Update entity | PUT | `/{plural-entity}/{entityId}` |
| Append to list | POST | `/{plural-parent}/{parentId}/{plural-entity}/append` |
| Delete from list | POST | `/{plural-parent}/{parentId}/{plural-entity}/delete` |
| Async request (reports/catalogs) | POST | `/{plural-parent}/{parentId}/{plural-entity}` |
| Check async status | GET | `/{plural-entity}/{entityId}/status` |
| Retrieve async output | GET | `/{plural-entity}/{entityId}/output` |

### Response structure
All responses include:
- `data`: Array of resource objects with `id`, `type`, `attributes`
- `errors`: Array of error objects with `code`, `title`, `detail`, `source`, `traceId`
- `meta` or `metadata`: Pagination info (offset/limit or pageIndex/pageSize)

### Pagination
| Parameter | Type | Default | Notes |
|-----------|------|---------|-------|
| `offset` | integer | 0 | Skip first N records |
| `limit` | integer | 500 | Max records per page |
| `pageIndex` | integer | — | 0-indexed page number |
| `pageSize` | integer | 25 | Records per page |
| `limitToId` | string (repeatable) | — | Filter by specific IDs |

### Common HTTP status codes
| Code | Meaning | Action |
|------|---------|--------|
| 200 | Success | Proceed |
| 201 | Created | Resource created successfully |
| 400 | Bad request | Check syntax, required fields, payload size |
| 401 | Unauthorized | Refresh access token |
| 403 | Forbidden | Check permissions/consent on account |
| 404 | Not found | Verify entity IDs in URL |
| 413 | Payload too large | Break request into smaller chunks |
| 429 | Rate limit exceeded | Implement exponential backoff (1s, 2s, 4s...) |
| 500/503 | Server error | Retry with exponential backoff |

## Decision guidance

### When to use Client Credentials vs Authorization Code

| Scenario | Client Credentials | Authorization Code |
|----------|-------------------|-------------------|
| Single data owner, internal integration | ✓ | — |
| Multi-user platform, self-service | — | ✓ |
| Need higher rate limits | ✓ (250/min) | — (10/min per account) |
| Scaling with user consent | — | ✓ (auto-scales) |
| Simpler setup | ✓ | — |
| User isolation required | — | ✓ |

### When to use synchronous vs asynchronous endpoints

| Endpoint type | Use case | Response time |
|---------------|----------|---------------|
| Synchronous | Campaign/line item CRUD, audience management | Real-time |
| Asynchronous | Catalog uploads, report generation | Polling required (status → output) |

### Bulk calls vs single requests

| Approach | Max IDs | When to use |
|----------|---------|------------|
| Single request | 1 | One campaign/line item |
| Bulk request | 50 | Multiple campaigns/line items, reporting |
| Multiple bulk calls | N×50 | Hundreds of entities (loop with pagination) |

## Workflow

### 1. Set up authentication
1. Log in to Criteo Partner Portal
2. Create a partner account (if needed)
3. Create an organization
4. Create an API app and select OAuth method (Client Credentials or Authorization Code)
5. Generate credentials (client_id, client_secret)
6. Store credentials securely (never commit to version control)

### 2. Obtain access token
1. POST to `https://api.criteo.com/oauth2/token` with:
   - `client_id`, `client_secret`, `grant_type=client_credentials` (or authorization code flow)
2. Extract `access_token` from response (valid 900 seconds)
3. Include token in `Authorization: Bearer {token}` header for all requests
4. Refresh token before expiry (implement 15-min refresh cycle)

### 3. Make API calls
1. Identify the resource (account, campaign, line item, audience, etc.)
2. Choose HTTP method (GET, POST, PUT) based on operation
3. Construct endpoint URL with version and resource path
4. Set `Content-Type: application/json` header
5. Include access token in Authorization header
6. Send request with properly formatted JSON payload
7. Parse response: check `errors` block first, then process `data`

### 4. Handle pagination
1. For list endpoints, check `meta.count` (offset/limit) or `metadata.totalItemsAcrossAllPages`
2. If more results exist, increment `offset` or `pageIndex` and retry
3. Continue until response contains fewer items than limit or `meta.count` equals current offset + response count

### 5. Handle asynchronous operations (reports/catalogs)
1. POST request to create async job (returns job ID)
2. Poll `/status` endpoint with job ID until status is "completed" or "failed"
3. Retrieve output via `/output` endpoint once complete
4. Parse output data from response

### 6. Handle errors
1. Check HTTP status code first
2. Read `errors[].code` and `errors[].detail` for specific issue
3. For 4xx errors: fix request (syntax, required fields, permissions)
4. For 429 errors: wait and retry with exponential backoff
5. For 5xx errors: retry with exponential backoff (10s, 20s, 40s...)
6. Include `traceId` from error when escalating to support

## Common gotchas

- **Token expiry**: Access tokens expire after 900 seconds. Implement proactive refresh before expiry, not just on 401 response.
- **Bulk call ID limit**: Sending >50 IDs in a single bulk request returns 400 error. Always chunk into batches of ≤50.
- **Reporting rate limit**: Reporting endpoints have stricter limits (40 calls/min vs 250 calls/min default). Account for this in retry logic.
- **Authorization Code refresh tokens**: Refresh tokens expire after 6 months and are revoked if user role changes. Require re-authorization.
- **Payload size**: Requests >413 limit fail. Break large payloads into multiple requests or use bulk append/delete endpoints.
- **Pagination format mismatch**: Some endpoints use `offset`/`limit`, others use `pageIndex`/`pageSize`. Check endpoint docs.
- **Missing required fields**: POST requests require all "R" (Required) fields. Omitting optional fields sets them to defaults. PUT requests omit fields = null.
- **Type field in requests**: The `type` field in request payloads is optional but can help validate structure. Omit if unsure.
- **Empty bulk responses**: Non-existent resources in bulk requests return 200 with empty `data` array, not 404.
- **Consent not granted**: 403 errors often mean user hasn't granted consent on the account. Relaunch OAuth flow if using Authorization Code.
- **Report output limits**: Report responses capped at 100k rows. Narrow date range or reduce IDs if hitting limit.
- **Asynchronous polling**: Don't poll status endpoint too frequently. Implement exponential backoff for polling (start at 1s, increase gradually).

## Verification checklist

Before submitting API integration work:

- [ ] Access token is obtained and included in `Authorization: Bearer` header
- [ ] Token refresh logic is implemented (refresh before 900s expiry)
- [ ] All required fields are present in POST/PUT payloads
- [ ] Bulk requests contain ≤50 IDs
- [ ] Pagination logic handles both `offset`/`limit` and `pageIndex`/`pageSize` formats
- [ ] Error handling checks `errors` block and includes `traceId` in logs
- [ ] Rate limit handling implements exponential backoff for 429 and 5xx errors
- [ ] Asynchronous operations poll `/status` before calling `/output`
- [ ] Content-Type header is set to `application/json`
- [ ] Entity IDs in URLs are URL-encoded if needed
- [ ] Sensitive credentials (client_secret, tokens) are not logged or committed
- [ ] Response parsing handles both `meta` and `metadata` pagination formats
- [ ] Test with small datasets before scaling to bulk operations

## Resources

**Comprehensive navigation**: https://developers.criteo.com/llms.txt

**Critical documentation pages**:
1. [API Overview & Getting Started](https://developers.criteo.com/criteo-apis/docs/overview) — Entry point for all APIs
2. [Authentication & OAuth Setup](https://developers.criteo.com/criteo-apis/docs/authentication) — Token generation and grant types
3. [API Troubleshooting Guide](https://developers.criteo.com/criteo-apis/docs/api-troubleshooting-guide) — Error codes, debugging, escalation

---

> For additional documentation and navigation, see: https://developers.criteo.com/llms.txt