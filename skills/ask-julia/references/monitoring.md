# Monitoring — Dashboards, SLOs, and Spotify SLAs

## Datadog retention limits

| Data type | Retention |
|---|---|
| Metrics | **3 months** |
| Logs | **15 days** |

For anything beyond 3 months → use **BigQuery** (see `bigquery-queries.md`).

---

## Key dashboards

| Dashboard | What it covers | Link |
|---|---|---|
| Standalone Vault | HTTP status breakdown, error rates, latency, traffic by client | [mum-ze5-ds5](https://app.datadoghq.eu/dashboard/mum-ze5-ds5/standalone-vault) |
| Eye of Sauron (Credential Lifecycle) | All CL services: Vault API, Network Tokens, Card Metadata, KMS, DynamoDB | [gya-btf-f9r](https://app.datadoghq.eu/dashboard/gya-btf-f9r/credential-lifecycles-eye-of-sauron) |

## Key monitors

| Monitor | Alert condition | Link |
|---|---|---|
| Standalone Vault 500s | 500 rate exceeded | [70782499](https://app.datadoghq.eu/monitors/70782499) |
| Standalone Vault latency | Latency threshold exceeded | [98434074](https://app.datadoghq.eu/monitors/98434074) |

---

## Downstream service baselines (Standalone Vault)

| Dependency | Expected behaviour |
|---|---|
| DynamoDB | Latency typically < 30ms |
| Vault API | Low error rate, stable latency |
| Network Tokens API | Low error rate, stable latency |
| Card Metadata API | ~1–2ms latency, low error rate |
| FXP / Bank Validation | **Up to 4s for US / Philippines — this is expected** |

---

## Spotify SLAs

Spotify has contractual SLAs for both uptime and latency on the Vault and Forward API.

### Uptime

**99.99%** — across all endpoints: `/tokens`, `/instruments`, `/customers`, `/network-tokens`, `/forward`

Measured **monthly**.

### P99 latency

| Endpoint | P99 SLA |
|---|---|
| `/forward` | **525ms** |
| All other Vault endpoints | **750ms** |

Measured **monthly**.

> These are measured against CKO's internal processing time (not the destination PSP round-trip for `/forward`).

---

## Useful log queries

**All Standalone Vault prod logs:**
```
service:standalone-vault-api env:prod
```

**500s by merchant:**
```
service:standalone-vault-api env:prod @StatusCode:500
```
Group by `@VaultAccountId` in Datadog to see which merchants are affected.

**4xx breakdown:**
```
service:standalone-vault-api @http.status_code:4**
```
Group by `@http.status_code,@VaultAccountId`.

**Trace a specific request by CorrelationId:**
```
@CorrelationId:<id>
```
Check the `@Elapsed` column across services to see where latency is.

---

## AWS resources

| Resource | Link |
|---|---|
| ECS cluster | [standalone-vault-api-prod](https://eu-west-1.console.aws.amazon.com/ecs/v2/clusters/standalone-vault-api-prod/clusterMetrics?region=eu-west-1) |
| DynamoDB config table | [standalone-vault-configuration-prod](https://eu-west-1.console.aws.amazon.com/dynamodbv2/home?region=eu-west-1#table?name=standalone-vault-configuration-prod) |

---

## Slack channels

- **#credential-lifecycle-release** — deployment notifications
- **#ask-forward-api** — merchant questions on Forward API
- **#credential-lifecycle-secure-exchange-devs** — team channel
