# Authentication

Every `/api/v1/*` request (except the Stripe webhook, which verifies itself via `Stripe-Signature`) must use exactly one of two strategies, chosen when your app was registered. PayGate picks the strategy per request by which header is present — `Authorization: Bearer` is checked first, then `X-App-Key`.

## HMAC (server-to-server)

Use this when your own backend calls PayGate — never embed the `api_secret` in a mobile app or browser bundle.

### Required headers

| Header | Value |
| --- | --- |
| `X-App-Key` | Your app's `api_key` |
| `X-Timestamp` | Current Unix time (seconds), as a string |
| `X-Signature` | See below |

### Computing the signature

```
signed_payload = "{timestamp}.{METHOD}.{path}.{raw_body}"
signature      = hex(HMAC_SHA256(signed_payload, api_secret))
```

Exact rules — get any one of these wrong and the signature will not match:

- **`{timestamp}`** — the same string you send in `X-Timestamp`, unmodified.
- **`{METHOD}`** — HTTP method, **uppercase** (`POST`, `GET`).
- **`{path}`** — the URL path only, **no query string**, with a **leading slash** (e.g. `/api/v1/checkout-sessions`). Not the full URL, not the host.
- **`{raw_body}`** — the exact request body bytes you're sending (e.g. for a `GET` with no body, this is an empty string). Not a re-serialized/re-ordered version of your JSON — sign the literal bytes on the wire.
- Joined with a literal `.` between each part — four parts, three dots.
- The signature itself is **HMAC-SHA256** in **lowercase hex** (not base64).

Timestamps more than 5 minutes (300 seconds) off the server's clock are rejected (`expired_timestamp`) — sync your server clock (NTP) if you see this in practice, not just on retries.

### Example (illustrative values, not a real secret)

```
Method:    POST
Path:      /api/v1/checkout-sessions
Timestamp: 1700000000
Body:      {"plan_ref":"premium_1m","amount":199000,"currency":"VND","mode":"payment","success_url":"https://a.com/ok","cancel_url":"https://a.com/no","external_ref":"user-42"}

signed_payload = "1700000000.POST./api/v1/checkout-sessions.{\"plan_ref\":\"premium_1m\",...}"
X-Signature    = hex(hmac_sha256(signed_payload, "your_api_secret"))
```

### `external_ref`

With HMAC, **you** decide and send `external_ref` in the request body — it identifies your own end user to PayGate. PayGate does not know or care what it represents, only that it's unique per user in your system. It is never derived automatically for this strategy, unlike Firebase ID Token below.

## Firebase ID Token (client calls PayGate directly)

Use this when your client already authenticates end users with Firebase Auth and you want it to call PayGate directly — with no secret embedded in the client, since a mobile/web bundle is never a safe place to keep one.

### Required header

| Header | Value |
| --- | --- |
| `Authorization` | `Bearer <firebase_id_token>` |

That's the only header needed — **do not** also send `X-App-Key`/`X-Timestamp`/`X-Signature`; PayGate identifies your app from the token's `aud` claim (your Firebase project ID, registered against your app at setup time) and never reads those HMAC headers when a `Bearer` token is present.

### How verification works

PayGate verifies the token's RS256 signature against Google's public JWKS (no Firebase Admin SDK involved), then checks:

- `aud` matches your app's registered Firebase project ID.
- `iss` equals `https://securetoken.google.com/{aud}`.
- The token has not expired.

The `sub` claim (Firebase UID) becomes `external_ref` automatically — **any `external_ref` you send in the request body is ignored** for this strategy. This is intentional: it's what lets `GET /api/v1/subscriptions/{external_ref}` safely reject (`403`) a caller trying to read another user's subscription by guessing a different UID in the URL — the token can only ever prove you're calling as yourself.

## Choosing a strategy

| | HMAC | Firebase ID Token |
| --- | --- | --- |
| Caller | Your backend | Client (mobile/web) directly |
| Secret storage | Your backend only | None — no secret needed |
| `external_ref` | You provide it | Derived from the token, always |
| Setup | `api_key` + `api_secret` | `api_key` unused for auth; Firebase project ID registered instead |

An app is configured for exactly one strategy at creation time — this isn't a per-request choice.
