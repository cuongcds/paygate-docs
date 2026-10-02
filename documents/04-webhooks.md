---
title: Webhooks
nav_order: 4
has_children: true
---

# Webhooks

PayGate can push events to your own server so you don't have to poll. This page covers both directions:

- **Outgoing** (PayGate → you): register a `webhook_url` on your app and PayGate will POST events to it. See [04.01 — Registering your webhook](04.01-registering-your-webhook.html) for setup and signature verification.
- **Incoming** (Stripe → PayGate): `POST /api/v1/webhooks/stripe` is PayGate's own endpoint — Stripe calls it, not you. Documented below so you understand what triggers your outgoing events.
- **Incoming** (bank transfer → PayGate): `POST /api/v1/webhooks/bank-transfer` is called only by PayGate's own `email-notification` service, a relay that forwards a parsed bank balance-change email together with its header fields. It is signed with a shared secret (`X-Timestamp` + `X-Signature = hex hmac_sha256(BANK_NOTIFY_SECRET, "{timestamp}.{raw_body}")`, timestamp within 5 minutes) and is not part of your integration. PayGate alone decides whether the email is genuine: it authenticates the sender from the headers the receiving mail server added (direct DKIM from an allowlisted bank domain, a trusted forwarder, or a trusted ARC chain such as Gmail auto-forward), then, in `staging`/`production`, checks the secret 10-digit code in the forward address (`paygate+{code}@...`) against the bank account selected by the app that owns the transaction (`APP_ENV=development` skips only this recipient-code check). Transport problems (oversized body over 256 KB, bad signature or timestamp, malformed JSON) answer `400`; every content decision answers `200` with `{status, reason?}` — `processed`, `already_completed`, `duplicate`, `underpaid`, `ignored` (`not_bank_email`, `no_transfer_code`, `unknown_transaction`, `transaction_not_pending`, `not_bank_transfer_transaction`, `bank_account_unavailable`) or `rejected` (`untrusted_sender`, `invalid_recipient`). The stored webhook event keeps only a redacted summary (message id, bank, amount, transfer code, decision) — never the code, addresses or headers. A confirmed transfer produces the same outgoing `transaction.completed` event as PayOS.

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
