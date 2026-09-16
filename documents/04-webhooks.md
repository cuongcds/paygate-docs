---
title: Webhooks
nav_order: 4
has_children: true
---

# Webhooks

PayGate can push events to your own server so you don't have to poll. This page covers both directions:

- **Outgoing** (PayGate → you): register a `webhook_url` on your app and PayGate will POST events to it. See [04.01 — Registering your webhook](04.01-registering-your-webhook.html) for setup and signature verification.
- **Incoming** (Stripe → PayGate): `POST /api/v1/webhooks/stripe` is PayGate's own endpoint — Stripe calls it, not you. Documented below so you understand what triggers your outgoing events.

## Events PayGate sends you

| `event_type` | When it fires |
| --- | --- |
| `subscription.updated` | Any time a subscription's status or `current_period_end` changes — activation, renewal, or cancellation |

More event types (e.g. one-off `transaction.completed`) may be added later — treat unrecognized `event_type` values as ignorable, not an error.

## What triggers a Stripe-side event

| Stripe event | Effect |
| --- | --- |
| `checkout.session.completed` | Marks the transaction `completed`; for subscriptions, fetches the Stripe subscription's `current_period_end` and upserts your subscription record |
| `invoice.paid` | Updates the subscription to `active` with a new `current_period_end` (renewal) |
| `customer.subscription.deleted` | Marks the subscription `canceled` |

Every other Stripe event type is accepted (200 OK) but otherwise ignored — this is not an error on your side. Events are deduplicated by Stripe's `event.id` — a retried delivery is acknowledged immediately without re-running any business logic.

## If you don't register a webhook

You can still integrate without one:

1. **Poll** `GET /api/v1/subscriptions/{external_ref}` after redirecting the user back from checkout, or on a schedule.
2. **Never treat arrival at your `success_url` as proof of payment** — that redirect happens client-side and carries no signed confirmation. Always verify against `GET /api/v1/subscriptions/{external_ref}` or a verified webhook delivery before unlocking paid features.
