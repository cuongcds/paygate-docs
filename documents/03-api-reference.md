---
title: API Reference
nav_order: 3
has_children: true
---

# API Reference

All responses use one envelope:

```json
{ "success": true, "data": { } }
{ "success": false, "error": { "code": "invalid_signature", "message": "..." } }
```

Always check `success` before reading `data`/`error`. See [Errors](05-errors.html) for the complete `error.code` list, and [Authentication](02-authentication.html) for the two auth strategies used below.

## Endpoints

- [03.01 — POST /api/v1/checkout-sessions](03.01-checkout-sessions.html)
- [03.02 — GET /api/v1/subscriptions/{external_ref}](03.02-subscriptions.html)
- [03.03 — GET /checkout/{public_token} (hosted picker page)](03.03-checkout-page.html)
- [03.04 — Testing without a real Stripe account](03.04-testing.html)

`POST /api/v1/webhooks/stripe` is not called by you — see [Webhooks](04-webhooks.html) instead.
