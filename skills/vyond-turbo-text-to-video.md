---
name: Generate a video from a text prompt with Vyond Turbo
description: Submit a text prompt (optionally grounded on up to three reference documents) to Vyond Turbo, track the task to completion by webhook or polling, and collect the expiring download URL and credit cost.
api: openapi/vyond-openapi-original.json
generated: '2026-08-05'
method: generated
source: https://api.vyond.com/doc/#tag/Turbo
operations:
  - TurboController.createTurbo
  - TurboController.getTurbo
  - WebhookController.createWebhook
---

# Generate a video from a text prompt with Vyond Turbo

Turbo is the one-shot path: a prompt in, a finished video out. Base URL
`https://api.vyond.com`, bearer token on every call.

## Prerequisites

- The account needs a **Turbo license**. Without it every Turbo call returns
  `403 missing required scope, invalid owner type or no Turbo license`.
- The token must be user-owned.
- **Turbo spends credits and accepts no `Idempotency-Key`.** Never blind-retry a
  create that timed out.

## 1. Submit the prompt

`TurboController.createTurbo` — `POST /rest/v1/turbo/`

Body is `TurboCreateReqBody`:

- `prompt` — required, 1–10,000 characters
- `mode` — `STILL_IMAGE` (default) or `VIDEO_CLIP`
- `files` — up to **3** reference files, **100 MB each**, documents capped at
  **100 pages**. `.txt`, `.pdf`, `.docx`, `.pptx`. Only available when the request
  is sent as `multipart/form-data`.
- `callbackWebhook` — `{url, secret}`; https url, 64-character alphanumeric secret

Failures:

- `400` — invalid request or file validation failed
- `422 Unprocessable Entity` — **malware detected in an uploaded file.** Treat this
  as terminal and report it; do not resubmit the same file.
- `429`, `500` — back off.

The response gives you the task (thread) `id`.

## 2. Wait for it

**Webhook.** Turbo events are *always emitted* for every task, so you may register
the subscription before the task finishes and still receive it. Either supply
`callbackWebhook` on the create call, or register a persistent subscription with
`WebhookController.createWebhook` (owner type must be `user`).

| Event | Meaning |
|---|---|
| `turbo_generation.succeeded` | `data` carries `downloadUrl`, `creditConsumed`, `expiredAt`, `turboThreadUrl` |
| `turbo_generation.failed` | `error.code` is `TIMEOUT` or `GENERATION_FAILED` |
| `turbo_generation.cancelled` | `error.code` is `CANCELLED` |

Note: `turbo_generation.cancelled` is documented with a sample payload but is
**not** in the subscribable `Webhook.events` enum — it appears to reach only a
one-off `callbackWebhook`. If cancellation matters to you, poll as well.

Always verify the signature — see `vyond-verify-webhook-signature.md`.

**Polling.** `TurboController.getTurbo` — `GET /rest/v1/turbo/{id}`.

`status` is one of `queued`, `processing`, `completed`, `failed`, `cancelled`.
`404` means the task does not exist for this user. Back off exponentially; the
`429` has no published limit and no `Retry-After`.

## 3. Collect the result

On `completed`, `TurboGetResBody` carries:

- `downloadUrl` — **expires** at `expiredAt`; fetch the bytes now
- `creditConsumed` — the actual cost of this generation
- `turboThreadUrl` — the human-facing page at `app.vyond.com/turbo/t/{id}`

If the URL has expired, re-call `TurboController.getTurbo` for a fresh one rather
than caching it.

## Cost discipline

`creditConsumed` is reported only *after* the fact, and no endpoint reads the
account balance. If you are running Turbo in a loop, record `creditConsumed` per
task yourself — the API gives you no running total and no budget check, and export
elsewhere in the API will start returning `402 Payment Required` once the account
is exhausted.
