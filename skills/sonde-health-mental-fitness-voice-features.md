---
name: sonde-health-mental-fitness-voice-features
description: >-
  Run Sonde Health's asynchronous Mental Fitness voice-feature scoring over 30 seconds
  of free speech — mint a scoped token, register the subject, upload the sample, create
  the inference job, then poll for the acoustic feature scores and the aggregate Mental
  Fitness score.
api: Sonde Platform Service API
base_url: https://api.sondeservices.com
operations:
  - POST /platform/v1/oauth2/token
  - POST /platform/v2/users
  - POST /platform/v1/storage/files
  - POST /platform/async/v1/inference/voice-feature-scores
  - GET /platform/async/v1/inference/voice-feature-scores/{asyncJobId}
  - GET /platform/v1/inference/voice-feature-scores
scopes:
  - sonde-platform/users.write
  - sonde-platform/storage.write
  - sonde-platform/voice-feature-scores.write
  - sonde-platform/voice-feature-scores.read
source: https://sondehealth.atlassian.net/wiki/spaces/SA/pages/2706702379/Mental+Fitness+Voice+Features+API
generated: '2026-08-28'
method: generated
---

# Score Mental Fitness voice features

Requires partner credentials issued by Sonde. This is the **asynchronous** surface:
you create a job, then poll it.

## 1. Get a scoped access token

`POST /platform/v1/oauth2/token` with `Authorization: Basic <base64(client-id:client-secret)>`,
`Content-Type: application/x-www-form-urlencoded`, body `grant_type=client_credentials`
and `scope=sonde-platform/users.write sonde-platform/voice-feature-scores.write sonde-platform/voice-feature-scores.read sonde-platform/storage.write`.

`expires_in` is 3600 seconds. A polling loop that outlives the token must refresh it.

## 2. Register the subject

`POST /platform/v2/users` with `Authorization: <access_token>`. Include device metadata
(mandatory since API 2.7) and, optionally, your own `userIdentifier`.

## 3. Request a signed upload URL

`POST /platform/v1/storage/files` with the file type, the end user's `countryCode` and
the `userIdentifier`. HIPAA residency applies: the audio must not leave the end user's
country.

## 4. Upload the sample

PUT the WAV to the signed S3 URL. Mental Fitness needs **30 seconds of free speech**
(roughly 2.5 MB), elicited by a prompt — an open question such as "how are you
feeling", or one of a randomized prompt set.

## 5. Create the inference job

`POST /platform/async/v1/inference/voice-feature-scores` with
`Authorization: <access_token>` and the stored file location. This returns an
`asyncJobId`.

## 6. Poll for the scores

`GET /platform/async/v1/inference/voice-feature-scores/{asyncJobId}`.

When the job completes you get per-feature acoustic scores — **Smoothness, Control,
Liveliness, Energy Range, Clarity, Crispness, Speech Rate, Pause Duration** — plus the
**aggregate Mental Fitness score**, which integrates the six acoustic scores (added in
API 3.3, 2021-09-20). Values run 0-100.

Poll with backoff. Sonde publishes no rate limits and no `Retry-After` header, so there
is no server-side signal telling you how often is too often; the docs say cloud scoring
takes 12-30 seconds depending on connectivity and sample length. Start at a few seconds
and back off — do not tight-loop an endpoint whose limits are undocumented.

## 7. Read history (optional)

`GET /platform/v1/inference/voice-feature-scores` returns score history, filterable by
time (`from`/`to`) and user. `GET /platform/v1/inference/voice-feature-scores/{voiceFeatureScoreId}`
fetches one.

## Interpreting the scores

Sonde's own framing: all of these vocal biomarkers "have been shown to trend lower for
individuals with reduced mental fitness and trend to higher values for individuals with
higher mental fitness", but "other factors may also contribute (e.g. airway conditions,
speech disorders, intoxication)".

These are **wellness measures, not diagnostics**. Do not present a score as a diagnosis,
a screening result, or a clinical finding, and do not let an agent act on one
autonomously. Sonde's Journaling docs also state plainly that detection of self-harm or
harm-to-others intent **is not supported** — never build a safety-triage path on this
API.

## Errors

Vendor JSON envelope `{ code, message, requestId }`. See
`errors/sonde-health-problem-types.yml`. The on-device SDKs surface the equivalent
quality failures as `Low Speech`, `High Noise`, and `Low Speech and High Noise`.

No idempotency key is published — a retried job creation may produce a duplicate job.
