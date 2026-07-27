---
name: Launch a Criteo Retail Media campaign
description: Authenticate, list accounts, create a campaign and a creative on the Criteo Retail Media API (the HookLogic product line).
api: openapi/hooklogic-criteo-retailmedia-openapi-original.json
operations: [GetCurrentApplication, GetAccounts, GetCampaignsByAccountId, CreateCampaignsByAccountId, GetAccountCreatives, CreateCreative]
---

# Launch a Criteo Retail Media campaign

Operating instructions for an agent creating a retail media campaign. Every step
uses a real operationId from the harvested OpenAPI.

## 1. Authenticate
- POST `https://api.criteo.com/oauth2/token` with your `client_id`/`client_secret`
  (client-credentials flow). The response is a Bearer access token valid for **900
  seconds** — refresh on `401`.
- Send `Authorization: Bearer <token>` on every request.

## 2. Confirm application + account context
- `GetCurrentApplication` — confirm the app identity and scopes granted.
- `GetAccounts` — list the accounts your app may act on; capture the `accountId`.

## 3. Create the campaign
- `GetCampaignsByAccountId` — check existing campaigns for the account.
- `CreateCampaignsByAccountId` — create the campaign under `accountId`. Send the
  `{ data: { type, attributes } }` envelope.

## 4. Attach a creative
- `GetAccountCreatives` — review existing creatives.
- `CreateCreative` — create the creative to run in the campaign.

## Rules
- Errors follow RFC 7807 in an `errors[]` block (traceId, type, code, title,
  detail) — see `errors/hooklogic-criteo-problem-types.yml`.
- Rate limits: client-credentials apps get 250 calls/min (40/min on reporting) —
  handle `429` with backoff. See `rate-limits/hooklogic-criteo-rate-limits.yml`.
- No idempotency-key contract is published; do not assume safe retries on POST.
