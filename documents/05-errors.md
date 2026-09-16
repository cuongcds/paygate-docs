---
title: Errors
nav_order: 5
---

# Errors

Every error response follows the same shape:

```json
{ "success": false, "error": { "code": "invalid_signature", "message": "..." } }
```

## Codes

| `code` | HTTP | When | Notes |
| --- | --- | --- | --- |
| `missing_authentication` | 401 | Neither `Authorization: Bearer` nor `X-App-Key` was sent | |
| `invalid_signature` | 401 | HMAC signature doesn't match | Also used for `app_not_found`-like cases (wrong `api_key`, app not `hmac`, app suspended) — deliberately vague to avoid revealing which check failed |
| `expired_timestamp` | 401 | `X-Timestamp` more than 5 minutes off the server's clock | Check both directions — a client clock running fast fails the same as one running slow |
| `invalid_token` | 401 | Firebase ID Token failed signature/issuer/expiry checks | |
| `app_not_found` | 401 | Firebase ID Token's `aud` doesn't match any registered app | |
| `invalid_request` | 400 | Body validation failed (missing/invalid field) | `message` describes the specific field |
| `payment_method_not_allowed` | 403 | Requested `payment_method` isn't enabled for your app's environment | e.g. `test` against a `production` app |
| `provider_error` | 502 | Stripe itself rejected the request | Transient — safe to retry with backoff |
| `invalid_card_details` | 400 | Test card declined (`test_card_code=4000000000000002`) | |
| `insufficient_funds` | 402 | Test card declined (`test_card_code=4000000000009995`) | |
| `not_found` | 404 | `GET /subscriptions/{external_ref}` — no subscription record for that user; or `GET /transactions/{transaction_id}` — id doesn't exist, or belongs to a different app/user | Not necessarily an error in your flow |
| `forbidden` | 403 | Firebase ID Token caller requested a different `external_ref` than their own token's `sub` | |

Any other `error.code` not listed here — including the case where a webhook's own signature check fails (returns `invalid_signature` too, but at `400` rather than `401`, since that's PayGate's own inbound endpoint, not one you call) — should be treated as opaque; don't pattern-match on `message` text, only on `code`.

## Retry guidance

| Situation | Retry? |
| --- | --- |
| `401`/`403` codes | No — fix the request (signature, token, permissions) first |
| `400` (`invalid_request`, `invalid_card_details`) | No — fix the payload |
| `402` (`insufficient_funds`) | No — this is a real decline, ask the user for another payment method |
| `502` (`provider_error`) | Yes — exponential backoff, this is Stripe/network flaking, not your integration |
| Network timeout / no response | Yes — but see idempotency note below |

## Idempotency

`POST /api/v1/checkout-sessions` is **not** idempotent — retrying a timed-out request creates a second transaction. If you need safe retries, track your own request ID client-side and check `GET /api/v1/subscriptions/{external_ref}` before retrying, to avoid double-charging a user who actually completed the first attempt.
