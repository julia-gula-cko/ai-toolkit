# Standalone Vault — OC Runbooks

## Quick reference: who to contact

| Issue | Contact |
|---|---|
| Standalone Vault internals / 500s / latency | **@credential-lifecycle-se-devs** |
| DynamoDB / KMS / Lambda issues | **@credential-lifecycle-se-devs** |
| Vault API, Cards API, Tokens API, Wallets | **@credential-lifecycle-se-devs** |
| Network Tokens or Card Metadata issues | **@credential-lifecycle-le-devs** |
| Bank Validation (FXP) | FXP team |
| Bank Validation (APM/Bank Payouts) | APM team |
| Merchant-specific traffic spike | CSM for that merchant |

---

## OC runbook: 500s exceeded

**Datadog monitors:** [70782499](https://app.datadoghq.eu/monitors/70782499)

### Step 1: Gather initial data (5 min)

1. Open [500s log query](https://app.datadoghq.eu/logs?query=service%3Astandalone-vault-api%20env%3Aprod%20%40StatusCode%3A500) — group by `@VaultAccountId`
   - How many merchants are affected?
   - Which endpoints are getting 500s?
   - What's the rough volume?

2. Open [Standalone Vault dashboard](https://app.datadoghq.eu/dashboard/mum-ze5-ds5/standalone-vault)
   - Is the 500 rate spiking?
   - Any correlated deploys or traffic events?

3. Check known issues below before escalating.

### Step 2: Investigate (10–15 min)

**A. Look for error patterns**
- Single recurring exception, or multiple different ones?
- Timeout / validation failure / downstream error (Vault, FXP, DynamoDB, Card Metadata)?
- Widen time window — did this just start?

**B. Check for traffic spike**
- In the dashboard → "Request by Client" — is one merchant dominating?
- If yes → contact CSM for that merchant

**C. Check downstream dependencies**

| Dependency | Expected | Dashboard |
|---|---|---|
| DynamoDB | < 30ms | [Standalone Vault](https://app.datadoghq.eu/dashboard/mum-ze5-ds5/standalone-vault) |
| Vault API | Low error rate | [Eye of Sauron](https://app.datadoghq.eu/dashboard/gya-btf-f9r/credential-lifecycles-eye-of-sauron) |
| Network Tokens API | Low error rate | [Eye of Sauron](https://app.datadoghq.eu/dashboard/gya-btf-f9r/credential-lifecycles-eye-of-sauron) |
| Card Metadata API | ~1–2ms | [Eye of Sauron](https://app.datadoghq.eu/dashboard/gya-btf-f9r/credential-lifecycles-eye-of-sauron) |
| FXP / Bank Validation | Up to 4s for US/PH | APM/FXP dashboards |

Use **CorrelationId** from a 500 log entry to trace across downstream logs.

**D. Check for recent deployment**
- [Harness vault_gen2](https://app.harness.io/ng/account/x9vTEK0gRkadO7XKkBzdTw/all/cd/orgs/credential_lifecycle/projects/vault_gen2/overview) or #credential-lifecycle-release
- Does the spike start right after a deploy? → likely root cause → consider rollback after escalating

### Escalation template (500s)

```
🔴 Escalating: Standalone Vault API - 500 Errors
**Summary:**
- Affected merchants (Vault Accounts): [number/list]
- Affected endpoints: [list]
- Error volume: [X requests/min, peak value]
- Duration: [how long]
**Investigation findings:**
- Traffic spike: [Yes/No]
- Dominant merchant: [Yes/No — VaultAccountId if yes]
- Downstream issues: [Yes/No — which service]
- Recent deployment: [Yes/No — version if available]
- Error pattern: [recurring exception / timeouts / downstream error]
**Example error:**
- Exception: [message]
- Endpoint: [resource]
- CorrelationId: [id]
**Links:**
- 500s query: https://app.datadoghq.eu/logs?query=service%3Astandalone-vault-api%20env%3Aprod%20%40StatusCode%3A500
- Dashboard: https://app.datadoghq.eu/dashboard/mum-ze5-ds5/standalone-vault
- Eye of Sauron: https://app.datadoghq.eu/dashboard/gya-btf-f9r/credential-lifecycles-eye-of-sauron
```

---

## OC runbook: increase in 4xx responses

**Datadog monitor:** [70782499](https://app.datadoghq.eu/monitors/70782499)

### Step 1: Gather initial data (5 min)

1. Open [4xx log query](https://app.datadoghq.eu/logs?query=service%3Astandalone-vault-api%20%40http.status_code%3A4%2A%2A) — group by `@http.status_code,@VaultAccountId`
   - Which merchants are affected?
   - Which status codes dominate — 401 / 404 / 422 / 400?
   - Which endpoints?

2. Open [Standalone Vault dashboard](https://app.datadoghq.eu/dashboard/mum-ze5-ds5/standalone-vault) — check status code breakdown

### Step 2: Investigate (10–15 min)

**A. 401 — Authentication failures**
- Traffic reaching SV has already been authenticated by Edge Gateway; 401s here mean SV rejected it at its own auth layer
- Check if this is new behaviour for this merchant

**B. 404 — Not found**
- Typically GET requests for non-existent or deleted entities
- Use the [GET 4xx query](https://app.datadoghq.eu/logs?query=service%3Astandalone-vault-api%20%40http.status_code%3A4**%20%40http.method%3AGET)
- Same instrument ID repeated? → merchant is requesting a deleted entity (known issue, not a bug)
- Cycling over many different IDs? → potential probing/abuse → escalate immediately

**C. 422 — Validation errors**
- Usually a merchant integration issue; fix on their side
- Isolated to one merchant + clearly invalid input → integration issue, not platform

**D. Traffic spike** — same as 500s runbook

**E. Recent deployment** — check Harness / #credential-lifecycle-release

### Escalation template (4xx)

```
🟠 Escalating: Standalone Vault API – 4xx Errors
**Summary:**
- Dominant 4xx types: [401/404/422/400]
- Affected merchants (Vault Accounts): [number/list]
- Affected endpoints: [list]
- Error volume: [X requests/min]
- Duration: [how long]
**Investigation findings:**
- Pattern: [single merchant / many / single endpoint / many endpoints]
- Traffic spike: [Yes/No]
- Suspicious behaviour (probing/rotating auth keys): [Yes/No]
- Recent deployment: [Yes/No]
**Example error:**
- Status code: [401/404/422]
- Endpoint: [resource]
- VaultAccountId: [id]
- CorrelationId: [id if present]
**Links:**
- 4xx query: https://app.datadoghq.eu/logs?query=service%3Astandalone-vault-api%20%40http.status_code%3A4%2A%2A
- Dashboard: https://app.datadoghq.eu/dashboard/mum-ze5-ds5/standalone-vault
```

---

## OC runbook: increase in latency

**Datadog monitor:** [98434074](https://app.datadoghq.eu/monitors/98434074)

### Step 1: Gather initial data (5 min)

1. Open [latency log query](https://app.datadoghq.eu/logs?query=service%3Astandalone-vault-api%20env%3Aprod) — group by `@VaultAccountId`
   - Which merchants and endpoints are affected?

2. Open [Standalone Vault dashboard](https://app.datadoghq.eu/dashboard/mum-ze5-ds5/standalone-vault)
   - Check HTTP Request Duration for p95/p99 on affected endpoints

3. Check known issues first.

### Step 2: Investigate (10 min)

**A. Traffic spike** — same as above

**B. Downstream dependencies** — check Eye of Sauron, trace by CorrelationId + @Elapsed column

**C. Recent deployment** — check Harness / #credential-lifecycle-release

### Escalation template (latency)

```
🔴 Escalating: Standalone Vault API - High Latency
**Summary:**
- Affected merchants: [number/list]
- Affected endpoints: [list]
- Current p95 latency: [value]ms
- Duration: [how long]
**Investigation findings:**
- Traffic spike: [Yes/No]
- Downstream issues: [Yes/No — which service]
- Recent deployment: [Yes/No]
**Links:**
- Datadog query: [link]
- Dashboard: https://app.datadoghq.eu/dashboard/mum-ze5-ds5/standalone-vault
```

---

## Known issues (don't panic)

1. **FXP Validation timeouts** — Can legitimately take up to ~4 seconds for US and Philippines flows. If you see timeout logs like `EventName: 'OnTimeout'` on Vault → FXP calls, and the rate isn't spiking, this is expected. Only escalate to FXP if the rate suddenly increases or spreads to many merchants.

2. **Isolated / sporadic 500s** — Single infrequent timeouts without a pattern are not necessarily systemic. Prioritise: clusters of errors, multiple merchants, or sustained high error rate.

3. **Repeated 404s for same ID** — Merchants sometimes keep requesting instruments they've already deleted. Not a bug.

4. **Expected 422s during rollout** — Merchants sending invalid payloads during testing/rollout. Isolated to one merchant + clearly invalid input → integration issue.

---

## Confluence references

- [OC OP: STANDALONE VAULT 500 exceeded](https://checkout.atlassian.net/wiki/spaces/CLSE/pages/7943029044/OC+OP+STANDALONE+VAULT+500+exceeded)
- [OC SOP: STANDALONE VAULT increase in 4xx Responses](https://checkout.atlassian.net/wiki/spaces/CLSE/pages/7943192987/OC+SOP+STANDALONE+VAULT+increase+in+4xx+Responses)
- [OC SOP: STANDALONE VAULT increase in latency](https://checkout.atlassian.net/wiki/spaces/CLSE/pages/7900758128/OC+SOP+STANDALONE+VAULT+increase+in+latency)
- [Standalone Vault API - OC Runbook](https://checkout.atlassian.net/wiki/spaces/CLSE/pages/6546948198/Standalone+Vault+API+-+OC+Runbook)
