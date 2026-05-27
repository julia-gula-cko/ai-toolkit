# Standalone Vault — Reference

## What it is

Standalone Vault is a merchant-facing public API for storing and managing payment instruments (cards, bank accounts, SEPA) and customers — without going through the gateway. Merchants interact with it directly via API key or JWT.

**Services involved:**
- `standalone-vault-api` — public-facing API (ECS, eu-west-1)
- `vault-portal` / `gateway-vault` / `vault-tokens-api` — internal vault services
- `cards-api` — handles card encryption (envelope encryption via KMS)
- `tokens-api` — tokenization

**Downstream dependencies (anything degrading here affects SV):**
- Vault (Vault Portal API v2 + Vault Portal API Customers)
- Network Tokens API
- FXP Validation API ← can be up to 4s for US/Philippines — this is expected, not a bug
- Card Metadata API

---

## Entity ID — when and how to send it

### The core problem

When an API key is set to **allow all processing channels**, `entity_id` is **not** included in the JWE token claims. The token alone can't identify which entity the request belongs to — you must provide it explicitly.

### Per-endpoint rules

| Endpoint | Need entity_id? | Where to send it | processing_channel_id accepted? |
|---|---|---|---|
| `POST /customers` | No | — | No |
| `GET /customers/{id}` | No | — | No |
| `PATCH /customers/{id}` | No | — | No |
| `DELETE /customers/{id}` | No | — | No |
| `POST /instruments` (card/bank) | **Yes** | **Body** `entity_id` OR **body** `processing_channel_id` | **Yes — body only** |
| `POST /instruments` (token) | No | — | No |
| `GET /instruments/{id}` | Optional (needed to get card number) | **Header** `Cko-Entity-Id` | No |
| `PATCH /instruments/{id}` | No | — | No |
| `DELETE /instruments/{id}` | No | — | No |

### Two key gotchas

1. **Creating card/bank instruments:** `entity_id` or `processing_channel_id` goes in the **JSON body**, not a header. Exactly one, not both — sending both or neither returns `422 entity_id_or_processing_channel_id_required`.

2. **Reading card number from GET:** If the response is missing the encrypted card number, the most likely cause is that the request didn't carry an `entity_id` (claim or `Cko-Entity-Id` header) tied to a PCI SAQ-D compliant, non-expired entity. No error is returned — card data is silently omitted. Metadata (BIN, last 4, expiry) is still present.

### Debugging entity ID errors in Datadog

Check **Envoy claims** in the Datadog logs for the request — look for the `entity_id` field. If it's missing from claims, the merchant must send it explicitly.

**Datadog log query:**
```
service:standalone-vault-api env:prod
```

---

## Network tokens on Standalone Vault

Network tokens are **not** provisioned or managed by Standalone Vault directly — that's the Credential Lifecycle **LE** team's scope.

If a merchant asks about network tokens on instruments:
- Escalate to **@credential-lifecycle-le-devs**
- For NT issues on Forward API (using `{{network_token_number}}` etc.) → see `forward-api-reference.md`

---

## CAT configuration

When onboarding a merchant to Standalone Vault:
- Enable **Vault** in CAT — always required
- Enable **Network Tokens** in CAT — only if the merchant will use network tokens

---

## Infrastructure

| Resource | Value |
|---|---|
| ECS cluster | `standalone-vault-api-prod` (eu-west-1) |
| DynamoDB config table | `standalone-vault-configuration-prod` |
| Harness project | [vault_gen2](https://app.harness.io/ng/account/x9vTEK0gRkadO7XKkBzdTw/all/cd/orgs/credential_lifecycle/projects/vault_gen2/overview) |
| Datadog dashboard | [mum-ze5-ds5](https://app.datadoghq.eu/dashboard/mum-ze5-ds5/standalone-vault) |
| Datadog monitor (500s) | [70782499](https://app.datadoghq.eu/monitors/70782499) |
| Datadog monitor (latency) | [98434074](https://app.datadoghq.eu/monitors/98434074) |
| Eye of Sauron dashboard | [gya-btf-f9r](https://app.datadoghq.eu/dashboard/gya-btf-f9r/credential-lifecycles-eye-of-sauron) |

---

## Vault Instruments Dashboard (LaunchDarkly)

To enable the Vault instruments dashboard for a merchant:
1. Go to LaunchDarkly → `dashboard` project
2. Find the `vault-dashboard` feature flag
3. Add the `client_id` to rule 1

[Onboarding Vault Instruments Dashboard — Confluence](https://checkout.atlassian.net/wiki/spaces/CLSE/pages/7154762313/Onboarding+Vault+Instruments+Dashboard)

---

## PCI / compliance

Forward API is **in-scope for PCI DSS** (it transmits CHD from CKO systems to PSPs). Key points:

- PAN is **not stored** in Forward API — it enters memory when retrieved from Vault and is used for template substitution, then discarded
- DynamoDB stores the pre-enriched request (before PAN is retrieved) and the scrubbed response — both are out of PCI scope
- CVV can be included via single-use `tok_` token; token expires after 15 minutes or on first use
- TLS 1.2+ enforced; destination URLs must use HTTPS
- Destination URL validation uses **Visa Global Registry of Service Providers** check
- New merchants: check PCI compliance level via [service desk](https://checkout.atlassian.net/servicedesk/customer/portal/111/group/296/create/274) before onboarding (one-time gate)

---

## Confluence references

- [Standalone Vault - Resolving Entity Id](https://checkout.atlassian.net/wiki/spaces/CLSE/pages/8403550299/Standalone+Vault+-+Resolving+Entity+Id)
- [Standalone Vault API: Documentation](https://checkout.atlassian.net/wiki/spaces/CLSE/pages/6784942713/Standalone+Vault+Api+Documentation)
- [Onboarding Guide Forward API for SE](https://checkout.atlassian.net/wiki/spaces/CLSE/pages/7396327473/Onboarding+Guide+Forward+API+for+SE)
