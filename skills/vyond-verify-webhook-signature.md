---
name: Subscribe to Vyond webhooks and verify the signature
description: Register, update and delete Vyond webhook subscriptions, and verify the HMAC-SHA256 signature on every delivered event before acting on it.
api: openapi/vyond-openapi-original.json
generated: '2026-08-05'
method: generated
source: https://api.vyond.com/doc/#tag/Webhook
operations:
  - WebhookController.getWebhooks
  - WebhookController.createWebhook
  - WebhookController.updateWebhook
  - WebhookController.deleteWebhook
---

# Subscribe to Vyond webhooks and verify the signature

Webhooks are the correct way to learn that a generation, Turbo task or export has
finished. Polling works but every operation declares a `429` with no published
limit, so polling at scale is guesswork.

## Register a subscription

`WebhookController.createWebhook` — `POST /rest/v1/webhooks/`

- `url` — **https only**
- `name` — a label
- `events[]` — any of:
  `video_generation.succeeded`, `video_generation.failed`,
  `video_export.succeeded`, `video_export.failed`,
  `turbo_generation.succeeded`, `turbo_generation.failed`
- `status` — `enabled` or `disabled`

The create response is `WebhookWithSecret` — **the secret is returned once.** Store
it immediately; it is your only means of verifying deliveries.

All four webhook operations require a **user-owned** token. An account/service token
returns `403 invalid owner type (non-user token)`.

## Manage subscriptions

- `WebhookController.getWebhooks` — `GET /rest/v1/webhooks/`. Returns the full list;
  there is no pagination.
- `WebhookController.updateWebhook` — `PATCH /rest/v1/webhooks/{webhookId}`. Use
  `status: disabled` to pause deliveries without losing the registration.
- `WebhookController.deleteWebhook` — `DELETE /rest/v1/webhooks/{webhookId}`.

`404` on update/delete means the webhook does not exist **or does not belong to
your user** — the two are not distinguished.

## One-off callbacks

Instead of a persistent subscription you can pass `callbackWebhook` (`{url, secret}`)
inline on `TurboController.createTurbo`,
`ContentGenerationV2Controller.createGeneration` or `VideoController.exportVideo`.
It is scoped to that single task.

The `secret` must match `^[0-9A-Za-z]{64}$` — exactly 64 characters, digits and
English letters only. **If you omit it, Vyond delivers the event unsigned**, and you
have no way to establish authenticity. Always supply one.

## Verify every delivery

Vyond signs with HMAC-SHA256 over the timestamp and the raw body:

1. Capture the **raw** request body before any JSON parsing. In Express:

   ```js
   app.use(express.json({
     verify: (req, res, buf, encoding) => {
       req.rawBody = buf.toString(encoding || 'utf-8');
     },
   }));
   ```

2. Read the two headers:

   ```js
   const timestamp = req.headers['x-vyond-request-timestamp'];
   const signature = req.headers['x-vyond-signature'];
   ```

3. Build the message with a colon delimiter:

   ```js
   const message = `${timestamp}:${req.rawBody}`;
   ```

4. Verify the hex signature against your stored secret:

   ```js
   const { webcrypto } = require('node:crypto');
   const enc = new TextEncoder();
   const key = await webcrypto.subtle.importKey(
     'raw', enc.encode(secret),
     { name: 'HMAC', hash: { name: 'SHA-256' } },
     false, ['verify'],
   );
   const isValid = await webcrypto.subtle.verify(
     { name: 'HMAC' }, key,
     Buffer.from(signature, 'hex'),
     enc.encode(message),
   );
   ```

5. **Reject stale timestamps.** Vyond recommends a tolerance such as ten minutes but
   does not enforce one — replay protection is entirely yours to implement.

## Event body

```ts
type WebhookEventBody = {
  event: string;
  data?: Record<string, unknown>;
  error?: { code: string };
};
```

`data` differs per event; `error` is present only on failure events. Known
`error.code` values: `UNSUITABLE_CONTENT`, `TIMEOUT`, `GENERATION_FAILED`,
`CANCELLED`. Full catalog: `asyncapi/vyond-webhooks.yml`.

## Known gaps

- There is no test/ping operation. You cannot verify a new endpoint without running
  a real, credit-consuming generation.
- No delivery-retry policy, dead-letter behaviour or delivery log is documented, so
  make your handler idempotent on `data.id` and assume you may see an event twice
  or not at all.
- `turbo_generation.cancelled` is documented but is not in the subscribable
  `events` enum.
