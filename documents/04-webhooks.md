# Webhooks

`POST /api/v1/webhooks/stripe` is PayGate's own endpoint — Stripe calls it, not you. This page explains what PayGate does with those events so you know what to expect and how to design your own integration around it.

## What PayGate does when Stripe notifies it

| Stripe event | Effect |
| --- | --- |
| `checkout.session.completed` | Marks the transaction `completed`; for subscriptions, fetches the Stripe subscription's `current_period_end` and upserts your subscription record |
| `invoice.paid` | Updates the subscription to `active` with a new `current_period_end` (renewal) |
| `customer.subscription.deleted` | Marks the subscription `canceled` |

Every other Stripe event type is accepted (200 OK) but otherwise ignored — this is not an error on your side.

Events are deduplicated by Stripe's `event.id` — a retried delivery is acknowledged immediately without re-running any business logic.

## How you find out payment succeeded

PayGate doesn't (in the MVP) forward Stripe webhooks onward to your own server — you have two options, and most integrations use both:

1. **Poll** `GET /api/v1/subscriptions/{external_ref}` after redirecting the user back from checkout, or on a schedule.
2. **Ask your gateway operator** whether your specific integration has been set up to receive a push notification (this is operator-specific, not a generic PayGate feature at this time — don't assume it's available without confirming).

**Never treat arrival at your `success_url` as proof of payment** — that redirect happens client-side and carries no signed confirmation. Always verify against `GET /api/v1/subscriptions/{external_ref}` (or your confirmed push mechanism) before unlocking paid features.
