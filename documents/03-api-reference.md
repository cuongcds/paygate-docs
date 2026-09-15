# API Reference

All responses use one envelope:

```json
{ "success": true, "data": { } }
{ "success": false, "error": { "code": "invalid_signature", "message": "..." } }
```

Always check `success` before reading `data`/`error`. See [Errors](05-errors.md) for the complete `error.code` list, and [Authentication](02-authentication.md) for the two auth strategies used below.

## Endpoints

- [03.01 — POST /api/v1/checkout-sessions](03.01-checkout-sessions.md)
- [03.02 — GET /api/v1/subscriptions/{external_ref}](03.02-subscriptions.md)
- [03.03 — GET /checkout/{public_token} (hosted picker page)](03.03-checkout-page.md)
- [03.04 — Testing without a real Stripe account](03.04-testing.md)

`POST /api/v1/webhooks/stripe` is not called by you — see [Webhooks](04-webhooks.md) instead.
