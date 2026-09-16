---
title: Postman collection
nav_order: 6
---

# Postman collection

A ready-to-import Postman collection covering the merchant-facing endpoints (HMAC auth): [`../api/paygate-merchant-api.postman_collection.json`](../api/paygate-merchant-api.postman_collection.json).

Firebase ID Token auth isn't included — that strategy is meant to be called from your client app directly (see [02 — Authentication](02-authentication.html)), not exercised from a generic API tool.

## Import

1. Postman → **Import** → select `api/paygate-merchant-api.postman_collection.json`.
2. Open the collection's **Variables** tab and fill in:

   | Variable | Value |
   | --- | --- |
   | `base_url` | Your gateway's base URL, no trailing slash (e.g. `https://payments.example.com`) |
   | `api_key` | Your app's `api_key` |
   | `api_secret` | Your app's `api_secret` |
   | `external_ref` | Any identifier for a test end user, e.g. `test-user-1` |

3. Save the collection (Ctrl/Cmd+S) so the variable values persist.

## How signing works

Every request in the collection carries `X-App-Key`, `X-Timestamp`, `X-Signature` headers whose values are template variables (`{{api_key}}`, `{{_computed_timestamp}}`, `{{_computed_signature}}`) rather than literals. A **collection-level pre-request script** computes the timestamp and signature fresh before each send, following the exact HMAC recipe in [02 — Authentication](02-authentication.html):

```
signed_payload = "{timestamp}.{METHOD}.{path}.{raw_body}"
signature      = hex(HMAC_SHA256(signed_payload, api_secret))
```

It uses Postman's built-in `CryptoJS` (no extra library to install) and reads `pm.request.body.raw` as the exact bytes being sent — so if you edit a request's JSON body, the signature is recomputed correctly on the next send without you touching anything.

You never need to compute or paste a signature by hand, and `api_secret` itself is never sent over the wire — only used locally in the script to derive the signature.

## Requests included

| Folder | Request | Maps to |
| --- | --- | --- |
| Checkout Sessions | Create checkout session (production — Stripe/picker) | [03.01](03.01-checkout-sessions.html) without `payment_method`, `mode="subscription"` |
| Checkout Sessions | Create checkout session (test payment_method — synchronous) | [03.01](03.01-checkout-sessions.html) with `payment_method=test`, `mode="subscription"` |
| Checkout Sessions | Create one-time payment (production — Stripe/picker) | [03.01](03.01-checkout-sessions.html) with `mode="payment"` — no `interval`/`interval_count` |
| Checkout Sessions | Create one-time payment (test payment_method — synchronous) | [03.01](03.01-checkout-sessions.html) with `mode="payment"` + `payment_method=test` — `current_period_end` comes back `null` |
| Subscriptions | Get subscription | [03.02](03.02-subscriptions.html) |

To try other test card outcomes (declines), edit the `test_card_code` field per [03.04 — Testing](03.04-testing.html) — no other change needed, the signature updates automatically.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| `401 invalid_signature` | Wrong `api_secret` variable, or you hand-edited a header instead of letting the pre-request script fill it | 
| `401 expired_timestamp` | Your machine's clock is off — sync it (NTP) |
| `403 payment_method_not_allowed` | Your app's environment doesn't allow `test_card_code` requests — check `apps.environment` in the app portal |

See [05 — Errors](05-errors.html) for the full list of error codes.
