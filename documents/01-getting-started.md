---
title: Getting Started
nav_order: 1
---

# Getting Started

## 1. Get your app credentials

Your PayGate account owner creates an "app" for your integration inside their organization's dashboard (`/apps/new`). You'll receive:

- **`api_key`** — public identifier, safe to log, not secret.
- One of:
  - **`api_secret`** (HMAC strategy) — shown once at creation time. Store it like any other secret; it cannot be retrieved again (only regenerated as a new app).
  - Nothing extra (Firebase ID Token strategy) — see [Authentication](02-authentication.html).

Pick **HMAC** if your integration calls PayGate from your own backend (server-to-server). Pick **Firebase ID Token** if your client (mobile/web) already authenticates end users with Firebase Auth and should call PayGate directly, with no secret embedded in the client.

## 2. Create a checkout session

```bash
curl -X POST https://payments.example.com/api/v1/checkout-sessions \
  -H "X-App-Key: pgk_xxx" \
  -H "X-Timestamp: 1700000000" \
  -H "X-Signature: <computed HMAC-SHA256, see 02-authentication.md>" \
  -H "Content-Type: application/json" \
  -d '{
    "external_ref": "user-42",
    "plan_ref": "premium_1m",
    "amount": 199000,
    "currency": "VND",
    "mode": "subscription",
    "interval": "month",
    "interval_count": 1,
    "success_url": "https://yourapp.com/success",
    "cancel_url": "https://yourapp.com/cancel"
  }'
```

Response:

```json
{ "success": true, "data": { "checkout_url": "https://checkout.stripe.com/c/pay/cs_test_..." } }
```

Redirect the end user to `checkout_url`. That's the entire integration surface for the happy path — Stripe (or PayGate's hosted picker page, see below) handles the rest and eventually redirects back to your `success_url`/`cancel_url`.

## 3. Handle the redirect back

Your `success_url`/`cancel_url` are called with no reliable query parameters to trust for granting access — **do not** mark a user as paid just because they landed on `success_url`. Payment confirmation must come from the webhook (step 4) or by polling `GET /api/v1/subscriptions/{external_ref}` (step 5), never from the redirect alone.

## 4. Receive the webhook (recommended)

Configure your own endpoint to receive PayGate's forwarded Stripe webhook notifications, or ask your gateway operator how webhook delivery to your system is set up for your app (this varies by integration — see [Webhooks](04-webhooks.html) for what PayGate itself does when it receives events from Stripe).

## 5. Or poll subscription status

```bash
curl https://payments.example.com/api/v1/subscriptions/user-42 \
  -H "X-App-Key: pgk_xxx" \
  -H "X-Timestamp: 1700000000" \
  -H "X-Signature: <computed HMAC-SHA256>"
```

```json
{ "success": true, "data": { "plan_ref": "premium_1m", "status": "active", "current_period_end": "2026-04-15 00:00:00" } }
```

See [API reference](03-api-reference.html) for every field, [Authentication](02-authentication.html) for exact signature computation, and [Errors](05-errors.html) for the full error code list.
