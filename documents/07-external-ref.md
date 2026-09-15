# Understanding `external_ref`

`external_ref` is your identifier for the end user — the one thing PayGate uses to tie a checkout, a subscription, and a transaction back to "your user #123." This page explains exactly where it comes from and where it shows up again, since the rules differ by **auth strategy**, not by checkout `mode`.

## It depends on auth strategy, not on `mode`

`mode: "payment"` vs `mode: "subscription"` ([03.01](03.01-checkout-sessions.md)) only changes whether a subscription record is created — it has **no effect** on how `external_ref` is determined. That's controlled entirely by which [authentication strategy](02-authentication.md) your app uses:

| | HMAC | Firebase ID Token |
| --- | --- | --- |
| Where `external_ref` comes from | The `external_ref` field **you send** in the request body | The token's own `sub` claim (Firebase UID) — **derived automatically, never read from the body** |
| Can you choose a different value per request? | Yes — you decide what it means (your own user id, an order id, anything unique per user in your system) | No — always the caller's own UID, by design |
| What if you send `external_ref` in the body anyway? | Used normally | **Silently ignored** — PayGate never trusts a client-supplied identifier when it can cryptographically derive one from the token instead |

This is why `POST /api/v1/checkout-sessions` documents `external_ref` as "only for HMAC" ([03.01](03.01-checkout-sessions.md)) — for Firebase ID Token apps, sending it is a no-op, not an error.

### Why the asymmetry

- **HMAC** is server-to-server: your own backend already authenticated the end user through your own system before it ever calls PayGate, so PayGate has no way to know who the user is except by trusting what your backend tells it.
- **Firebase ID Token** is called directly by an end user's client (mobile/web) — PayGate cannot trust a client to self-report *which* user it is (a malicious client could just lie), so it derives `external_ref` from the token's signature-verified `sub` claim instead. This is also what makes `GET /api/v1/subscriptions/{external_ref}` and `GET /api/v1/transactions/{transaction_id}` safe to expose directly to a client: requesting another user's data by guessing a different `external_ref`/id returns `403 forbidden` / `404 not_found` rather than real data, because the caller can never make the token claim to be someone else.

## Where it shows up again

Once a checkout is created, the same `external_ref` value threads through everything else in the system tied to it:

| Where | How |
| --- | --- |
| [`GET /api/v1/subscriptions/{external_ref}`](03.02-subscriptions.md) | Path parameter — looks up the subscription row keyed by `(app_id, external_ref)` |
| [`GET /api/v1/transactions/{transaction_id}`](03.05-transactions.md) | Returned as `data.external_ref` in the response — compare it against what you expected before trusting the result |
| `success_url`/`cancel_url` redirect params | Appended as `paygate_external_ref` — see [03.01 — What comes back on success_url/cancel_url](03.01-checkout-sessions.md#what-comes-back-on-success_urlcancel_url) |
| [Outgoing webhooks](04.01-registering-your-webhook.md) | Included in `data.external_ref` on every `subscription.updated` event |

It is never translated, hashed, or namespaced — the exact string you sent (HMAC) or the exact `sub` claim (Firebase ID Token) is what comes back everywhere above.

## One `external_ref` can have many transactions, but only one subscription

`external_ref` is **not unique** on `transactions` — every `POST /api/v1/checkout-sessions` call inserts a new transaction row regardless of how many already exist for that `external_ref`. This is intentional: a user can buy a one-time item (`mode: "payment"`) more than once, retry a subscription checkout after canceling or failing, or simply check out again later.

`subscriptions`, by contrast, has exactly one row per `(app_id, external_ref)` — each successful subscription checkout **upserts** that single row rather than adding a new one.

| Table | Rows per `external_ref` |
| --- | --- |
| `transactions` | Unlimited — one new row per checkout attempt, successful or not |
| `subscriptions` | Exactly one (or zero, if the user has never completed a subscription checkout) |

**Practical consequence:** if a user has multiple transactions, `GET /api/v1/subscriptions/{external_ref}` ([03.02](03.02-subscriptions.md)) only ever tells you their *current* subscription state — it can't tell you which specific checkout attempt just happened, or distinguish two separate one-time payments. When you need to confirm one particular attempt (e.g. handling a `success_url` redirect, or reconciling a specific purchase), use the `paygate_transaction_id` from that redirect with [`GET /api/v1/transactions/{transaction_id}`](03.05-transactions.md) instead of relying on `external_ref` alone.

## Practical implications

- **Pick a stable, permanent value.** If you use your own database's user id, don't reuse it for a different user later (e.g. after account deletion) — PayGate has no way to know the old association should be forgotten, and a new user would inherit the old user's subscription history.
- **HMAC apps choose their own scheme.** It doesn't have to be a numeric user id — an order id, a household id, anything unique per "entity that has a subscription" in your domain works, as long as you're consistent about what it means across your calls to `checkout-sessions`, `subscriptions`, and how you interpret the webhook payload.
- **Firebase ID Token apps get this for free but lose flexibility.** You can't group multiple Firebase users under one shared `external_ref` (e.g. "family plan") — each UID is its own `external_ref`, always.
- **Don't try to mix strategies for the same app.** An app is configured for exactly one auth strategy at creation time ([02 — Authentication](02-authentication.md)) — if you need both a trusted backend flow and a direct-from-client flow, that's two separate PayGate apps, each with its own `external_ref` semantics.
