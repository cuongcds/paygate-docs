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

Pick **HMAC** if your integration calls PayGate from your own backend (server-to-server). Pick **Firebase ID Token** if your client (mobile/web) already authenticates end users with Firebase Auth and should call PayGate directly, with no secret embedded in the client. Creating plans and prices requires an HMAC app.

Every request below is authenticated the same way (`X-App-Key`, `X-Timestamp`, `X-Signature` for HMAC); the headers are abbreviated after the first example.

## 2. Set up a customer and a plan (once)

A subscription is sold on a **price** of a **plan**, to a **customer**. Create the plan and its prices once (for example from a setup script):

```bash
curl -X POST https://payments.example.com/api/v1/plans \
  -H "X-App-Key: pgk_xxx" \
  -H "X-Timestamp: 1700000000" \
  -H "X-Signature: <computed HMAC-SHA256, see 02-authentication.md>" \
  -H "Content-Type: application/json" \
  -d '{
    "code": "premium",
    "name": "Premium",
    "prices": [
      { "name": "Premium monthly", "amount": 199000, "currency": "VND", "interval_unit": "month", "interval_count": 1 },
      { "name": "Premium yearly",  "amount": 1990000, "currency": "VND", "interval_unit": "year", "interval_count": 1 }
    ]
  }'
```

The response lists each price with its `price_…` id — keep the one you want to sell (see [03.07 — Plans and prices](03.07-plans-and-prices.html)). Then create a customer for each person you bill and keep the returned `cus_…` id ([03.06 — Customers](03.06-customers.html)):

```bash
curl -X POST https://payments.example.com/api/v1/customers \
  -H "X-App-Key: pgk_xxx" -H "X-Timestamp: 1700000000" -H "X-Signature: <HMAC>" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Jane Doe", "email": "jane@example.com", "merchant_customer_id": "user-42" }'
```

## 3. Create a checkout session

```bash
curl -X POST https://payments.example.com/api/v1/checkout-sessions \
  -H "X-App-Key: pgk_xxx" -H "X-Timestamp: 1700000000" -H "X-Signature: <HMAC>" \
  -H "Content-Type: application/json" \
  -d '{
    "external_ref": "user-42",
    "mode": "subscription",
    "price_id": "price_8a1d6c3e9f024b7d5a2c1e90",
    "customer_id": "cus_5d2e7b9a1c4f8036ab12de45",
    "success_url": "https://yourapp.com/success",
    "cancel_url": "https://yourapp.com/cancel"
  }'
```

Response:

```json
{
  "success": true,
  "data": {
    "checkout_url": "https://payments.example.com/checkout/9f3a...64hexchars",
    "activated": false,
    "trial": null
  }
}
```

Redirect the end user to `checkout_url`. PayGate (its hosted payment page, or the payment provider) handles the rest and eventually redirects back to your `success_url`/`cancel_url`. If the price has a free trial, the subscription is activated immediately (`activated: true`) and `checkout_url` is already your `success_url` — see [03.01](03.01-checkout-sessions.html). For a one-time purchase use `"mode": "payment"` with `plan_ref`, `amount` and `currency` instead.

## 4. Handle the redirect back

Your `success_url`/`cancel_url` are called with no reliable query parameters to trust for granting access — **do not** mark a user as paid just because they landed on `success_url`. Payment confirmation must come from the webhook (step 5) or by polling `GET /api/v1/subscriptions/{external_ref}` (step 6), never from the redirect alone.

## 5. Receive the webhook (recommended)

Register a `webhook_url` on your app and PayGate will POST signed events such as `subscription.updated` and `subscription.renewed` to it — see [Registering your webhook](04.01-registering-your-webhook.html) and [Webhooks](04-webhooks.html).

## 6. Or poll subscription status

```bash
curl https://payments.example.com/api/v1/subscriptions/user-42 \
  -H "X-App-Key: pgk_xxx" \
  -H "X-Timestamp: 1700000000" \
  -H "X-Signature: <computed HMAC-SHA256>"
```

```json
{
  "success": true,
  "data": {
    "plan_ref": "premium",
    "plan_id": "plan_3f9c1a7e5b2d4c8a91e0f6b2",
    "price_id": "price_8a1d6c3e9f024b7d5a2c1e90",
    "customer_id": "cus_5d2e7b9a1c4f8036ab12de45",
    "status": "active",
    "current_period_end": "2026-11-03 08:00:00",
    "in_trial": false,
    "trial_ends_at": null
  }
}
```

`status` turns `expired` as soon as `current_period_end` passes. See [03.08 — Subscription lifecycle](03.08-subscription-lifecycle.html) for reminders and renewal, [API reference](03-api-reference.html) for every endpoint, [Authentication](02-authentication.html) for exact signature computation, and [Errors](05-errors.html) for the full error code list.
