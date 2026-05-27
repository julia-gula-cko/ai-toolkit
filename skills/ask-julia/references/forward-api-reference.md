# Forward API — Reference

## What it is
Forward API lets merchants forward payment requests (including stored card credentials from the Vault) to third-party PSPs, without the card data ever touching the merchant's own systems. CKO decrypts the stored credential, injects it into the request body using placeholder substitution, and forwards to the destination URL.

---

## API Endpoints

| Environment | Endpoint |
|---|---|
| Production — forward request | `POST https://forward.checkout.com/forward` |
| Sandbox — forward request | `POST https://forward.sandbox.checkout.com/forward` |
| Production — secrets management | `POST https://forward.checkout.com/secrets` |
| Sandbox — secrets management | `POST https://forward.sandbox.checkout.com/secrets` |
| JWK (encryption key retrieval) | `GET https://forward.checkout.com/.well-known/jwks` |

**Dashboard location:** Vault → Credential forwarding

---

## Resource naming

| Prefix | What it is | Notes |
|---|---|---|
| `src_` | Payment instrument (stored card) | Permanent, lives in Vault |
| `tok_` | Token | Expires after 15 minutes |
| Network token | Card scheme token | Provisioned via CKO NT flow |
| `secret_` | Merchant secret | Stored in Secrets Manager, referenced as `{{ secret_<name> }}` |

---

## Request body structure

### Forwarding a stored instrument (src_)

```json
{
  "source": {
    "type": "id",
    "id": "src_xxxxxxxxxxxx",
    "cvv_token": "tok_xxxxxxxxxxxx"  // optional
  },
  "destination_request": {
    "url": "https://api.stripe.com/v1/charges",
    "method": "POST",
    "headers": {
      "raw": {
        "Authorization": "Bearer {{ secret_stripe_key }}"
      }
    },
    "body": "{\"amount\": 1000, \"currency\": \"usd\", \"card\": {\"number\": \"{{card_number}}\", \"exp_month\": \"{{card_expiry_month}}\", \"exp_year\": \"{{card_expiry_year_yy}}\", \"cvc\": \"{{card_cvv}}\"}}"
  }
}
```

### Forwarding a token (tok_)

```json
{
  "source": {
    "type": "token",
    "token": "tok_xxxxxxxxxxxx",
    "store_for_future_use": true  // optional
  },
  "destination_request": { ... }
}
```

---

## Placeholders

These are substituted at runtime with decrypted card/token data:

### Card details
- `{{card_number}}` — full card number (PAN)
- `{{card_cvv}}` — CVV / CVC
- `{{card_pin}}` — PIN (if available)
- `{{card_expiry_month}}` — expiry month as integer (e.g. `3`)
- `{{card_expiry_month_mm}}` — expiry month zero-padded (e.g. `03`)
- `{{card_expiry_year_yy}}` — 2-digit year (e.g. `27`)
- `{{card_expiry_year_yyyy}}` — 4-digit year (e.g. `2027`)
- `{{cardholder_name}}`

### Billing address
- `{{billing_address_line1}}`, `{{billing_address_line2}}`
- `{{billing_address_city}}`, `{{billing_address_state}}`
- `{{billing_address_country}}`, `{{billing_address_zip}}`

### Network tokens
- `{{network_token_number}}` — DPAN (network token number)
- `{{network_token_type}}` — scheme (e.g. `VISA`)
- `{{network_token_cryptogram}}` — TAVV/AAV (for CITs with `request_cryptogram: true`)
- `{{network_token_eci}}` — ECI value
- `{{network_token_expiry_month}}`
- `{{network_token_expiry_year_yy}}` / `{{network_token_expiry_year_yyyy}}`

### Network token config in request
```json
"network_token": {
  "enabled": true,
  "required": true,             // true = fail if NT unavailable; false = fall back to card
  "request_cryptogram": true    // true for CITs, false for MITs
}
```

### Network token required vs fallback behavior

**`required: true`**
- Forward API attempts to retrieve a network token (and cryptogram if requested)
- If NT data is not available → request fails with **422**, error code `network_token_not_available`
- The destination forward request is **not sent**

**`required: false`**
- Forward API still attempts NT retrieval first
- If token/cryptogram is unavailable → does **not** fail; continues with card data
- Destination request is sent
- Use `IsNetworkTokenAvailable` conditional in the body template for reliable fallback

### Fallback body template (required = false)

```json
"body": "{% if IsNetworkTokenAvailable %}type=nt&network_token={{ network_token_number }}&nt_exp_month={{ network_token_expiry_month }}&nt_exp_year={{ network_token_expiry_year_yyyy }}&nt_cryptogram={{network_token_cryptogram}}&nt_eci={{network_token_eci}}&nt_type={{network_token_type}}{% else %}type=card&card_number={{card_number}}&card_exp_month={{card_expiry_month}}&card_exp_year={{card_expiry_year_yyyy}}&card_cvc={{card_cvv}}{% endif %}"
```

> **Important:** If you reference network token placeholders without a conditional and no token is available, the placeholders resolve to **empty strings** — no error, just empty values in the forwarded request. Always use `IsNetworkTokenAvailable` when fallback behaviour matters.

---

## Variables (reusable values)

Define once, reference anywhere in the request:

```json
{
  "variables": [
    { "name": "amount", "value": "1000" }
  ],
  "destination_request": {
    "body": "{\"amount\": \"{{amount}}\"}"
  }
}
```

---

## Secrets

Secrets are merchant-specific values (API keys, shared secrets) stored encrypted (KMS) and referenced without exposing the value.

**Reference syntax:** `{{ secret_<name> }}`

Example: `{{ secret_stripe_api_key }}`

**Secrets API:** `POST /secrets` — create/update a secret for a merchant entity.

---

## JWE Encryption

For destinations that require encrypted payloads:

```
{{ jwe_encrypt(data=<value>, key=<jwk>, alg='<key-enc-alg>', enc='<content-enc-alg>', headers=<optional>) }}
```

Supported algorithms:
- Key encryption: `RSA-OAEP`, `RSA-OAEP-256`, `A256GCMKW`
- Content encryption: `A128GCM`, `A256GCM`, `A128CBC-HS256`, `A256CBC-HS512`

Retrieve the target's public JWK first:
`GET https://forward.checkout.com/.well-known/jwks`

---

## Enabling a merchant on Forward API

### What you need
- `client_id` (e.g. `cli_abc123`)
- `entity_id` (e.g. `ent_xyz456`)
- Destination URLs (full domain names, e.g. `api.stripe.com`)
- Which environment: sbox, prod, or both

### Steps
1. Enable entity in **Retool** (internal config tool) — works for both sbox and prod
2. Allowlist each destination URL per entity — URLs are configured at entity level, individually
3. Sbox and prod are **completely independent** — if they need both, do both separately
4. Remind the merchant: API keys must have **`forward` scope**. Without it, calls fail at auth even if everything else is set up.
5. **Dashboard visibility** (optional): enable via LaunchDarkly flag — `temp-credentials-lifecycle-forward-dashboard-310824` or `tempPermissionsCredentialsLifecycleForwardCredentials`. Request in #ask-forward-api with client_id + entity_id.

> **Compliance check (new merchants only):** Before onboarding a brand-new merchant to Forward API for the first time, confirm PCI compliance level via service desk:
> https://checkout.atlassian.net/servicedesk/customer/portal/111/group/296/create/274
> This is a one-time gate, not needed every time you add a URL.

### Required API key scopes (full setup)
For merchants using Forward API with Standalone Vault:
- `forward` ← **always required for forwarding**
- `vault`, `vault:instruments`, `vault:card-metadata` ← for using stored instruments
- `vault:network-tokens` ← if using network tokens
- `gateway`, `gateway:payment-instruments` ← if also using gateway
- `sessions:browser` ← if using sessions
- `tokens` ← if using tokens API

For simple forwarding only: just `forward`.

### CAT configuration
In CAT:
- Enable **Vault** (always required when using `src_` instruments)
- Enable **Network Tokens** (only if forwarding with network tokens)

### Feature requests
New destination URLs or new features → redirect to **Max Lamond** (PM), not engineering.

---

## Diagnosing "destination not allowed" errors

Most common cause in #ask-forward-api. Check in order:

1. **URL not allowlisted for that entity** — check which `entity_id` they're using and confirm that URL is enabled for it in Retool
2. **Wrong environment** — sbox request but only prod was enabled, or vice versa
3. **Entity mismatch** — the instrument was created under a different entity than the one being used for the forward call
4. **API key missing `forward` scope** — calls fail at auth but can surface as "not allowed"

Ask: "Which entity are you using and which environment?" Then check.

---

## Known current limitations

- **Request/response data from destinations** — stored in S3 (`forward-api-logs-archive-{env}`) but table is unpartitioned, making queries unreliable. Breakglass access required. Improvement in progress.

---

## Onboarding guides (Confluence)
- Hub: [Forward API How to](https://checkout.atlassian.net/wiki/spaces/CLSE/pages/8441364741/Forward+API+How+to)
- SE-facing: [Onboarding Guide Forward API for SE](https://checkout.atlassian.net/wiki/spaces/CLSE/pages/7396327473/Onboarding+Guide+Forward+API+for+SE)
- Engineer-facing (Retool): [Onboarding Guide Forward API for engineers (retool)](https://checkout.atlassian.net/wiki/spaces/CLSE/pages/8441463018/Oboarding+Guide+Forward+API+for+engineers+retool)
- Legacy/general: [Onboarding Guide for Forward APIs](https://checkout.atlassian.net/wiki/spaces/CLSE/pages/6385598914/Onboarding+Guide+for+Forward+APIs)

---

## Internal onboarding via Retool (engineer steps)

For staging — the Forward API Configuration retool app:

1. **Get retool access if you don't have it.** If you can't see the "Forward Api Configuration" app in retool (staging), request access to the retool group via CKO Technology service desk under enhanced payments env: https://checkout.atlassian.net/servicedesk/customer/portal/111/group/296/create/274
2. **Add the destination** (if it doesn't exist yet) in retool.
3. **Register the merchant**: add `client_id`, `entity_id`, and choose the destination.

Retool works for both sbox and prod. For Standalone Vault setup that still needs to go through the Forward API team, use this Postman request: [Standalone Vault Postman request](https://checkout-com.postman.co/workspace/Credential-Lifecycle~58950456-71e0-490f-9a5a-b3da42a8a7d1/request/1913195-08454ca6-bd5a-420f-85ad-5f78318ba3b1?action=share&source=copy-link&creator=19941929&ctx=documentation)

---

## Useful links

- **PCI compliance gate (new merchants):** https://checkout.atlassian.net/servicedesk/customer/portal/111/group/296/create/274
- **Standalone Vault setup (Postman):** https://checkout-com.postman.co/workspace/Credential-Lifecycle~58950456-71e0-490f-9a5a-b3da42a8a7d1/request/1913195-08454ca6-bd5a-420f-85ad-5f78318ba3b1
- **Dashboard visibility flag (forward credentials display):** https://app.launchdarkly.com/projects/dashboard/flags/temp-credentials-lifecycle-forward-dashboard-310824/targeting
- **Dashboard visibility flag (permissions):** https://app.launchdarkly.com/projects/dashboard/flags/tempPermissionsCredentialsLifecycleForwardCredentials/targeting
- **API access key request guide:** https://checkout.atlassian.net/wiki/spaces/ATLAS/pages/5091000419

---

## Onboarding checklist for high-traffic merchants
Before a large merchant goes live, gather:
1. **Traffic & Load** — avg RPS, peak RPS, spike events (e.g. Black Friday + estimated RPS), traffic pattern (steady / burst / batch)
2. **API & Feature Usage** — which endpoints, features needed (NT replacement, encryption, secrets)
3. **End-to-End Flows** — full flow description, fallback flows
4. **Testing** — confirm full flows tested in sbox
5. **Go-Live date**
