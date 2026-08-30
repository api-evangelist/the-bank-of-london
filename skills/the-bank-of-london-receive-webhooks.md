---
name: the-bank-of-london-receive-webhooks
description: Subscribe to Bank of London webhook events, verify the PS256 JWS signature on every delivery, and handle at-least-once, out-of-order delivery correctly.
api: Bank of London API
version: v2
generated: '2026-08-30'
method: generated
source: https://developer.bankoflondon.com/docs/guides/manage-webhooks-guide + openapi/the-bank-of-london-api-openapi.json
operations:
  - CreateWebhook
  - GetWebhooks
  - PatchWebhook
  - DeleteWebhook
  - CreateTestEvent
  - RegenerateWebhookKey
---

# Receive and verify webhooks

## Steps

1. **Expose an HTTPS endpoint** that accepts `POST` and returns **200** quickly. The bank treats any
   non-200 as a failed delivery.

2. **Create the subscription.** `POST /v2/webhooks` (`CreateWebhook`) with the endpoint `url`, the
   `events` you want, and the API `version` that fixes the schema of the event `data` property.
   Available events: `PAYMENT_PENDING`, `PAYMENT_SUCCESSFUL`, `PAYMENT_FAILED`,
   `TRANSACTION_SUCCESSFUL`, `TRANSACTION_EXPORT_SUCCESSFUL`, `TRANSACTION_EXPORT_FAILED`.

3. **Store the public key from the response.** `CreateWebhook` returns a public key. The matching
   private key is held by the bank and is bound to the environment (Sandbox or Live) of the API key
   that created the webhook. **You cannot retrieve the public key again later** — persist it now.

4. **Verify every delivery.** Each webhook request carries an `x-jws-signature` header: a detached
   PS256 JWS whose payload holds `content-digest` (SHA-256 of the request body, hexadecimal) and
   `created` (UNIX timestamp). Verify the signature against the stored public key, recompute the
   digest over the raw body, and check `created` for freshness. **Reject anything that fails — an
   unverified webhook is an unauthenticated instruction to change your ledger.**

5. **Test before going live.** `POST /v2/webhooks/{id}/create-test-event` (`CreateTestEvent`) fires a
   `TEST` event at your endpoint so you can prove delivery and signature verification end to end.

6. **Rotate when needed.** `POST /v2/webhooks/{id}/regenerate-key` (`RegenerateWebhookKey`) issues a
   new key pair and returns the new public key. Deploy the new key before rotating, or you will
   reject live traffic.

## Delivery semantics you must code for

- **At-least-once.** The same event id can arrive more than once. Store processed event `id`s and
  ignore repeats — the bank recommends exactly this.
- **No ordering guarantee.** Sort on the `timestamp` property, never on arrival order. A
  `PAYMENT_SUCCESSFUL` can land before the `PAYMENT_PENDING` for the same payment.
- **Up to 20 minutes latency.** Do not use webhooks as a synchronous confirmation channel.
- **15 retry attempts** on a non-200: 1m, 5m, 15m, 30m, 1h, 3h, 6h, 12h, then 24h for attempts 9–15.
  That is a multi-day tail — make your handler idempotent.
- **Event types are extensible.** New values can be added without a major version change. Default-case
  unknown `eventType` values instead of throwing.

## Payload envelope

`{ id, eventType, eventVersion, data, timestamp }` — `data` is the API response object for that type,
in the schema of the `version` you set on the subscription.
