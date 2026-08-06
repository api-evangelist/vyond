---
name: Generate a Vyond video and export it
description: Create a Vyond Go or AI-avatar video from structured parameters, wait for it via webhook or polling, then export it to mp4/webm/gif or a SCORM package and retrieve the download URL before it expires.
api: openapi/vyond-openapi-original.json
generated: '2026-08-05'
method: generated
source: https://api.vyond.com/doc/
operations:
  - ParameterController.getParameters
  - ContentGenerationV2Controller.createGeneration
  - ContentGenerationV2Controller.getGeneration
  - VideoController.exportVideo
  - VideoController.getVideoExportDownload
---

# Generate a Vyond video and export it

Base URL is `https://api.vyond.com`. All calls carry `Authorization: Bearer <token>`.
The REST surface lives under `/rest/{version}/`.

## Before you start

- The token must be a **user-owned** token. Account/service tokens get
  `403 invalid owner type (non-user token)` on the read and webhook operations.
- Generation requires the `VYOND_GO` scope; export requires `VIDEO_EXPORT`.
- **Generation and export both spend account credits, and there is no
  `Idempotency-Key`.** If a create call times out, do NOT blind-retry — you will
  start a second billable job. Poll or wait for the webhook first.

## 1. Look up the valid parameter vocabulary

`ParameterController.getParameters` — `GET /rest/v1/parameters/`

Query `feature` and `parameterKeys`. Generation requests must draw their character,
voice, layout, background and aspect-ratio values from this vocabulary; the enums
are large and Vyond changes them with product releases, so read them, don't hardcode.

Failure: `403` means missing scope, wrong owner type, or a user-state check failed.

## 2. Create the generation

`ContentGenerationV2Controller.createGeneration` — `POST /rest/v2/generations/`

Body is `ContentGenerationV2CreateReqBody`:

- `type` — `vyondGo` or `aiAvatar` (required)
- `vyondGoParams` or `aiAvatarParams` — matching the type
- `callbackWebhook` — optional `{url, secret}`; `url` must be https, `secret` must be
  exactly 64 characters of `[0-9A-Za-z]`

Prefer `/rest/v2/` over `/rest/v1/generations/`. Both are live and neither is marked
deprecated, but only v2 exposes a readable task.

`400` means an invalid or unsupported generation type or parameter — check the
response `reason`/`message`, and re-read step 1.

## 3. Wait for it

Two ways, and you should use both:

**Webhook (preferred).** Supply `callbackWebhook` on the create call, or register a
persistent subscription with `WebhookController.createWebhook`. You will receive
`video_generation.succeeded` carrying `videoId`, `downloadUrl` and `expiredAt`, or
`video_generation.failed` carrying `error.code` (e.g. `UNSUITABLE_CONTENT`).
Verify the signature — see `vyond-verify-webhook-signature.md`.

**Polling (fallback).** `ContentGenerationV2Controller.getGeneration` —
`GET /rest/v2/generations/{id}`. Back off exponentially: every operation declares
`429`, and Vyond publishes no numeric limit and no `Retry-After` header, so you
cannot pace yourself in advance — you can only slow down after being told no.

## 4. Export the video

`VideoController.exportVideo` — `POST /rest/v1/videos/{videoId}/exports`

Body is `ExportVideoReqBody`:

- `category` — `mp4`, `webm`, `gif`, `scorm_1_2`, `scorm_2004_v4`
- `resolution` — one of `240p` `360p` `480p` `600p` `720p` `1080p`
- `revision` — optional; defaults to the latest revision
- `burnInCaption` — optional; permanently renders captions into the frames
- `completionCriteria` — 0–100, **required for SCORM exports**
- `callbackWebhook`, `shouldSendEmailNotification` — optional

Failures that matter here:

- `402 Payment Required` — out of credits. There is no endpoint that reads the
  credit balance, so 402 is your only signal. Do not retry; surface it.
- `409 Conflict` — the conversion failed or has not completed yet.
- `410 Gone` — the conversion was cancelled.
- `429` — export-specific rate limit.

## 5. Retrieve the download URL

`VideoController.getVideoExportDownload` —
`GET /rest/v1/videos/{videoId}/exports/{conversionId}`

`conversionId` is a positive integer. The response carries `downloadUrl` and
`expireAt` — **the URL expires.** Download the bytes immediately; do not persist the
URL and expect it to work later. Re-call this operation to mint a fresh one.

This operation is the only one in the spec with no `security` block, but its own
`401`/`403` responses prove it is authenticated and needs `VIDEO_EXPORT`. Send the
bearer token.

## Errors

All errors use the Vyond envelope, not RFC 9457:

```json
{"err": "INCORRECT_CREDENTIALS", "reason": "INVALID_CREDENTIALS"}
```

`err` is the code to branch on; `details[]` is populated when
`err` is `REQUEST_VALIDATION_FAILED`. Full catalog: `errors/vyond-problem-types.yml`.

## Known gap

There is no operation that lists or reads a video. `videoId` must come from a
generation webhook payload or from the Vyond web application — the API cannot
discover it.
