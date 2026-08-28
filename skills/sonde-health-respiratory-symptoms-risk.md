---
name: sonde-health-respiratory-symptoms-risk
description: >-
  Score a voice sample for Respiratory Symptoms Risk on the Sonde Health platform:
  mint a scoped token, register the subject, get a signed upload URL, upload the WAV,
  and request the measure score. Use when integrating Sonde's respiratory vocal
  biomarker into a partner backend.
api: Sonde Platform Service API
base_url: https://api.sondeservices.com
operations:
  - POST /platform/v1/oauth2/token
  - POST /platform/v2/users
  - POST /platform/v1/storage/files
  - POST /platform/v1/inference/scores
scopes:
  - sonde-platform/users.write
  - sonde-platform/storage.write
  - sonde-platform/scores.write
source: https://sondehealth.atlassian.net/wiki/spaces/SA/pages/2705031260/Respiratory+Symptoms+Risk+API
generated: '2026-08-28'
method: generated
---

# Score Respiratory Symptoms Risk

> **Before you start.** This flow requires partner credentials (`client-id`,
> `client-secret`) issued by Sonde during onboarding. There is no self-service key.
> Sonde's docs mark the v1 measure **deprecated** and recommend the V2 variant
> (`/platform/v2/inference/...`) for new work — see
> `lifecycle/sonde-health-lifecycle.yml`. Run this only where the v1 measure is what
> the partner contract covers.

## 1. Get a scoped access token

`POST /platform/v1/oauth2/token`

- `Authorization: Basic <base64(client-id:client-secret)>`
- `Content-Type: application/x-www-form-urlencoded`
- Body: `grant_type=client_credentials` and
  `scope=sonde-platform/users.write sonde-platform/scores.write sonde-platform/storage.write`

Keep the client credential **server-side only**. Sonde's docs are explicit: do not put
it in client-side code or a public repository. The response gives `access_token`,
`token_type: Bearer`, and `expires_in: 3600` — re-fetch after an hour.

When handing a token to a handset, mint a narrower one. If the device only uploads,
give it `sonde-platform/storage.write` and nothing else.

## 2. Register the subject

`POST /platform/v2/users` with `Authorization: <access_token>`.

Send the subject's demographic attributes (`yearOfBirth`, gender) and device metadata.
Device metadata became **mandatory** at API 2.7 when `POST /platform/v1/users` was
deprecated in favour of v2. Pass your own `userIdentifier` as a query parameter if you
want to map Sonde's data back to your records.

## 3. Request a signed upload URL

`POST /platform/v1/storage/files` with `Authorization: <access_token>`.

Supply the file type (`wav`), the `countryCode` of the **end user**, and the
`userIdentifier`. The country code is not cosmetic: Sonde treats the audio file as PII
under HIPAA and will not let it leave the end user's country, so the signed URL it
returns points at a region-appropriate bucket.

## 4. Upload the WAV

`PUT`/`POST` the raw WAV bytes to the signed S3 URL from step 3 —
`Content-Type: application/octet-stream`. This call goes to S3, not to
`api.sondeservices.com`, and carries no Sonde token.

The sample must be a **single continuous held "ahh" vowel for 6 seconds** (roughly
600 KB), recorded indoors with minimal background noise. Check
`sandbox/sonde-health-sandbox.yml` and Sonde's Audio Format Specifications page before
choosing a recorder configuration.

## 5. Ask for the score

`POST /platform/v1/inference/scores` with `Authorization: <access_token>`, passing the
returned file location, the measure name, and the `userIdentifier`.

The score is a value between 0 and 100.

## Handling the elicitation check

Sonde runs an automatic Elicitation Check (ELCK) on respiratory samples, validating
energy level, voice presence and correct utterance. A failure comes back as:

- **HTTP 422**, `code: ELICITATION_CHECK_FAILED`
- description: "We are not able process the audio-file for audio quality. Please make
  sure to record in a clear voice, in a quiet place, and without pause to pass the
  audio quality check"

Sonde's own guidance: show a neutral message and ask for another sample, and **stop
after three attempts** — treat a third failure as an unavailable test result rather
than retrying indefinitely. Sonde states ELCK accuracy is approximately 80%, so a
correct sample can still be rejected.

## Errors and retries

Every response carries a `requestId`. Quote it to support@sondehealth.com. The error
envelope is `{ code, message, requestId }` — vendor JSON, **not** RFC 9457. See
`errors/sonde-health-problem-types.yml`.

**Retry carefully.** Sonde publishes no idempotency key and no retry-safety statement
for any POST on this path, so a retried `POST /platform/v2/users` or
`POST /platform/v1/inference/scores` may create a duplicate. Prefer checking state over
blind retry, and never auto-retry a write more than once without human confirmation.

## What you cannot undo

There is no delete operation. Removing an uploaded voice sample requires emailing
support@sondehealth.com with the user identifier and the specific deletion request.
No retention window is published — do not tell a user one.
