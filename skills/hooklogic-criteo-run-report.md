---
name: Run a Criteo Retail Media async performance report
description: Kick off an asynchronous performance report, poll for completion, and fetch the output on the Criteo Retail Media API.
api: openapi/hooklogic-criteo-retailmedia-openapi-original.json
operations: [GenerateAsyncPerformanceReport, GetAsyncExportStatus, GetAsyncExportOutput]
---

# Run a Criteo Retail Media async performance report

Criteo reporting uses a generate -> poll -> fetch async pattern. Every step uses
a real operationId from the harvested OpenAPI.

## 1. Authenticate
- POST `https://api.criteo.com/oauth2/token` (client-credentials). Bearer token,
  900s lifetime. Reporting endpoints are limited to **40 calls/min**.

## 2. Generate the report
- `GenerateAsyncPerformanceReport` — submit the report request (date range,
  dimensions, metrics). The response returns an export/report id.

## 3. Poll for completion
- `GetAsyncExportStatus` — poll with the export id until status is complete.
  Back off between polls to respect the reporting rate limit.

## 4. Fetch the output
- `GetAsyncExportOutput` — download the finished report. Output may be
  `application/json`, `application/x-json-stream`, or `application/csv`.

## Rules
- `429` means you exceeded the reporting rate limit — back off and retry.
- Errors follow RFC 7807 (`errors[]` with traceId) — capture the `traceId` for
  Criteo support escalation.
