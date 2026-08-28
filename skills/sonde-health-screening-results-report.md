---
name: sonde-health-screening-results-report
description: >-
  Pull paged, filtered screening-session outcomes from the Sonde Screening API — each
  row a session token, subject, PASS/FAIL result and calculation timestamp — for
  organization-level reporting.
api: Sonde Screening API
base_url: https://api.sondeservices.com
operations:
  - POST /platform/api/v1/oauth2/token
  - GET /platform/api/v1/screening-results
scopes:
  - sonde-platform/screening-results.list
  - sonde-platform/reports.read
source: openapi/sonde-health-screening-api-openapi.yaml
generated: '2026-08-28'
method: generated
---

# Report on screening results

This is the one Sonde surface with a published OpenAPI —
`openapi/sonde-health-screening-api-openapi.yaml`, harvested verbatim from the open-api
macro on Sonde's Product Partner API space. Read it alongside this skill.

## 1. Get a token

`POST /platform/api/v1/oauth2/token`

- `Authorization: Basic <base64(client-id:client-secret)>` — the spec describes it as
  "Basic64Encode(client_id:client_secret) in the authorization header through Basic HTTP
  authorization"
- `Content-Type: application/x-www-form-urlencoded`
- Body: `grant_type=client_credentials&scope=sonde-platform/screening-results.list`

A malformed request returns **400** with `{ "error": "..." }`.

Note the path: the Screening API sits under `/platform/api/v1/`, not `/platform/v1/`
like the Platform Service API. They are different token endpoints.

## 2. List screening results

`GET /platform/api/v1/screening-results` with `Authorization: <access_token>` and the
`sonde-platform/reports.read` scope.

Query parameters, all optional:

| Parameter | Type | Meaning |
|---|---|---|
| `pageIndex` | integer | The page index out of the whole screening data to fetch |
| `userName` | string | Restrict to one subject |
| `from` | date-time | UTC start of the time frame |
| `to` | date-time | UTC end of the time frame |

## 3. Read the response

```
{
  "requestId": "...",
  "numberOfRecords": 1234,
  "numberOfPages": 2,
  "screeningResults": [
    { "token": {"name": "token_1"},
      "user": {"name": "james@example.com"},
      "result": true,
      "calculatedAt": "2020-10-25T12:12:12Z" }
  ]
}
```

`result` is a boolean: **true = PASS, false = FAIL**. `calculatedAt` is ISO-8601.

## 4. Page correctly

Read `numberOfPages` from the first response and stay inside it. Requesting a page past
the end does **not** return an empty list — it returns **HTTP 500** with
`code: PAGE_INDEX_OUT_OF_BOUND`. An empty result set for a filter that matches nothing
comes back as `numberOfRecords: 0, numberOfPages: 0, screeningResults: []`, which is the
normal no-data shape.

## Errors

| Status | Code | Meaning |
|---|---|---|
| 400 | `INVALID_REQUEST` | Field-level detail in `missingFields`, `invalidFields`, `invalidCombinationFields`. Date-times must be ISO-8601 UTC. |
| 403 | `FORBIDDEN_ACCESS` | You asked for subjects outside your own organization. |
| 404 | `USER_NOT_FOUND` / `ORGANIZATION_NOT_FOUND` | The subject or org could not be resolved. |
| 500 | `PAGE_INDEX_OUT_OF_BOUND` | Page index past `numberOfPages`. |
| 500 | `INTERNAL_SERVER_ERROR` | Server-side failure — retry with backoff, quote `requestId`. |

## Handling the data

The rows carry subject email addresses and health-screening outcomes. Treat the payload
as PHI: do not log it, do not paste it into a prompt or a ticket, and do not move it
across a border. Aggregate before presenting.

This endpoint is read-only, so there is nothing to undo — but there is also no
idempotency or rate-limit signal published anywhere on this API, so page politely.
