---
title: Webhooks
nav_order: 4
has_children: true
---

# Webhooks

PayGate can push events to your own server so you don't have to poll. This page covers both directions:

- **Outgoing** (PayGate → you): register a `webhook_url` on your app and PayGate will POST events to it. See [04.01 — Registering your webhook](04.01-registering-your-webhook.html) for setup and signature verification.
- **Incoming** (Stripe → PayGate): `POST /api/v1/webhooks/stripe` is PayGate's own endpoint — Stripe calls it, not you. Documented below so you understand what triggers your outgoing events (it only reports completed one-time payments).
- **Incoming** (bank transfer → PayGate): `POST /api/v1/webhooks/bank-transfer` is called only by PayGate's own `email-notification` service, a relay that forwards a parsed bank balance-change email together with its header fields. It is signed with a shared secret (`X-Timestamp` + `X-Signature = hex hmac_sha256(BANK_NOTIFY_SECRET, "{timestamp}.{raw_body}")`, timestamp within 5 minutes) and is not part of your integration. PayGate alone decides whether the email is genuine: it authenticates the sender from the headers the receiving mail server added (direct DKIM from an allowlisted bank domain, a trusted forwarder, or a trusted ARC chain such as Gmail auto-forward), then, in `staging`/`production`, checks the secret 10-digit code in the forward address (`paygate+{code}@...`) against the bank account selected by the app that owns the transaction (`APP_ENV=development` skips only this recipient-code check). Transport problems (oversized body over 256 KB, bad signature or timestamp, malformed JSON) answer `400`; every content decision answers `200` with `{status, reason?}` — `processed`, `already_completed`, `duplicate`, `underpaid`, `ignored` (`not_bank_email`, `no_transfer_code`, `unknown_transaction`, `transaction_not_pending`, `not_bank_transfer_transaction`, `bank_account_unavailable`) or `rejected` (`untrusted_sender`, `invalid_recipient`). The stored webhook event keeps only a redacted summary (message id, bank, amount, transfer code, decision) — never the code, addresses or headers. A confirmed transfer produces the same outgoing `transaction.completed` event as PayOS.

## Events PayGate sends you

| `event_type` | When it fires | `data` fields |
| --- | --- | --- |
| `subscription.updated` | Every time a subscription changes — first activation, renewal, a new customer attached, a new price chosen | `external_ref`, `paygate_customer_id`, `merchant_customer_id`, `plan_id`, `price_id`, `plan_ref`, `status`, `current_period_end`, `in_trial`, `trial_ends_at` |
| `subscription.renewed` | A payment was applied to an already existing subscription (a renewal, including the first payment after a trial) | `external_ref`, `paygate_customer_id`, `merchant_customer_id`, `plan_id`, `price_id`, `plan_ref`, `provider_code`, `transaction_id`, `amount`, `currency`, `previous_period_end`, `current_period_end`, `was_trial` |
| `subscription.renewal_reminder_sent` | A renewal reminder email was sent to the customer (once per stage per period) | `external_ref`, `paygate_customer_id`, `merchant_customer_id`, `plan_id`, `price_id`, `plan_ref`, `reminder_day`, `period_end`, `in_trial`, `sent_at` |
| `transaction.completed` | A one-time (`mode: "payment"`) payment completed, with any payment method including Stripe | `external_ref`, `plan_ref`, `amount`, `currency`, `status` |

Notes:

- `status` in `subscription.updated` is the effective status ([03.02](03.02-subscriptions.html)); `paygate_customer_id` is the `cus_…` id and `merchant_customer_id` is the value you gave when creating the customer (`null` if none).
- `reminder_day` is `3` or `1` (days before `period_end`), or `0` for the "expired" notice. The payload never contains the renewal link.
- A second payment for a period that was already renewed does not produce `subscription.renewed`.
- Delivery is at-least-once; deduplicate on your side.

Treat unrecognized `event_type` values as ignorable, not an error. Payload examples are in [04.01](04.01-registering-your-webhook.html#payload-examples).

## Stripe is a one-time payment method

PayGate, not Stripe, owns subscriptions: Stripe is only used to collect a single payment (the first payment or each renewal). Stripe's subscription/invoice events (`invoice.paid`, `customer.subscription.*`) are not used. When a Stripe payment completes, you receive `subscription.updated` (+ `subscription.renewed` for a renewal) for a subscription, or `transaction.completed` for a `mode: "payment"` checkout — exactly as with PayOS, bank transfer or the test method.

PayGate's own endpoint `POST /api/v1/webhooks/stripe` accepts Stripe's `checkout.session.completed` and `checkout.session.async_payment_succeeded` (a payment counts only once Stripe reports it as paid). Every other Stripe event type is accepted (200 OK) but ignored — this is not an error on your side. Events are deduplicated by Stripe's `event.id` — a retried delivery is acknowledged immediately without re-running any business logic.

## If you don't register a webhook

You can still integrate without one:

1. **Poll** `GET /api/v1/subscriptions/{external_ref}` after redirecting the user back from checkout, or on a schedule.
2. **Never treat arrival at your `success_url` as proof of payment** — that redirect happens client-side and carries no signed confirmation. Always verify against `GET /api/v1/subscriptions/{external_ref}` or a verified webhook delivery before unlocking paid features.
