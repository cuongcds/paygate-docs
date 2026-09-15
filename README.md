# PayGate API Documentation

Language-agnostic integration reference for PayGate, a multi-tenant payment gateway (Stripe today, more providers later). If you're integrating in PHP, see the [PayGate PHP SDK](https://github.com/cuongcds/paygate-php) — it implements everything described here.

- [01 — Getting started](documents/01-getting-started.md)
- [02 — Authentication](documents/02-authentication.md)
- [03 — API reference](documents/03-api-reference.md)
  - [03.01 — POST /api/v1/checkout-sessions](documents/03.01-checkout-sessions.md)
  - [03.02 — GET /api/v1/subscriptions/{external_ref}](documents/03.02-subscriptions.md)
  - [03.03 — Hosted checkout picker page](documents/03.03-checkout-page.md)
  - [03.04 — Testing without a real Stripe account](documents/03.04-testing.md)
  - [03.05 — GET /api/v1/transactions/{transaction_id}](documents/03.05-transactions.md)
- [04 — Webhooks](documents/04-webhooks.md)
  - [04.01 — Registering your webhook](documents/04.01-registering-your-webhook.md)
- [05 — Errors](documents/05-errors.md)
- [06 — Postman collection](documents/06-postman-collection.md)

## Base URLs

| Environment | Base URL |
| --- | --- |
| Production | `https://payments.example.com` |
| Dev/staging | `https://payments-dev.example.com` |

Replace with the actual domain your gateway operator gave you when they registered your app.
