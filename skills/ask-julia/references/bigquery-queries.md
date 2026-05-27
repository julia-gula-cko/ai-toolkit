# Forward API — BigQuery Queries

## Table

`cko-data-cl-prod-5361.source_flink.vault_network_tokens_forwardapirequest`

---

## Latency column reference

| Column | What it measures |
|---|---|
| `duration_ms` | Total time in Forward API (full request lifecycle) |
| `forward_duration_ms` | Outbound call to destination only. Present only when `forward_started_at` is defined. |
| **Internal latency** | `duration_ms - COALESCE(forward_duration_ms, 0)` — matches Datadog metric `forward_api.internal_latency_duration` |

Use `COALESCE(forward_duration_ms, 0)` so rows where the forward never happened are still counted.

---

## All percentiles in one query (recommended)

```sql
SELECT
  TIMESTAMP_TRUNC(started_at, MONTH) AS month,
  APPROX_QUANTILES(duration_ms - COALESCE(forward_duration_ms, 0), 100)[OFFSET(50)] AS p50_internal_ms,
  APPROX_QUANTILES(duration_ms - COALESCE(forward_duration_ms, 0), 100)[OFFSET(90)] AS p90_internal_ms,
  APPROX_QUANTILES(duration_ms - COALESCE(forward_duration_ms, 0), 100)[OFFSET(99)] AS p99_internal_ms,
  COUNT(*) AS row_count
FROM `cko-data-cl-prod-5361.source_flink.vault_network_tokens_forwardapirequest`
WHERE started_at >= TIMESTAMP(DATE_SUB(DATE_TRUNC(CURRENT_DATE(), MONTH), INTERVAL 12 MONTH))
  AND started_at <  TIMESTAMP(DATE_TRUNC(CURRENT_DATE(), MONTH))
  AND duration_ms IS NOT NULL
GROUP BY month
ORDER BY month;
```

---

## Monthly p99 — last 12 complete months

```sql
SELECT
  TIMESTAMP_TRUNC(started_at, MONTH) AS month,
  APPROX_QUANTILES(duration_ms - COALESCE(forward_duration_ms, 0), 100)[OFFSET(99)] AS p99_internal_ms,
  COUNT(*) AS row_count
FROM `cko-data-cl-prod-5361.source_flink.vault_network_tokens_forwardapirequest`
WHERE started_at >= TIMESTAMP(DATE_SUB(DATE_TRUNC(CURRENT_DATE(), MONTH), INTERVAL 12 MONTH))
  AND started_at <  TIMESTAMP(DATE_TRUNC(CURRENT_DATE(), MONTH))
  AND duration_ms IS NOT NULL
GROUP BY month
ORDER BY month;
```

## Monthly p90

```sql
SELECT
  TIMESTAMP_TRUNC(started_at, MONTH) AS month,
  APPROX_QUANTILES(duration_ms - COALESCE(forward_duration_ms, 0), 100)[OFFSET(90)] AS p90_internal_ms,
  COUNT(*) AS row_count
FROM `cko-data-cl-prod-5361.source_flink.vault_network_tokens_forwardapirequest`
WHERE started_at >= TIMESTAMP(DATE_SUB(DATE_TRUNC(CURRENT_DATE(), MONTH), INTERVAL 12 MONTH))
  AND started_at <  TIMESTAMP(DATE_TRUNC(CURRENT_DATE(), MONTH))
  AND duration_ms IS NOT NULL
GROUP BY month
ORDER BY month;
```

## Monthly p50

```sql
SELECT
  TIMESTAMP_TRUNC(started_at, MONTH) AS month,
  APPROX_QUANTILES(duration_ms - COALESCE(forward_duration_ms, 0), 100)[OFFSET(50)] AS p50_internal_ms,
  COUNT(*) AS row_count
FROM `cko-data-cl-prod-5361.source_flink.vault_network_tokens_forwardapirequest`
WHERE started_at >= TIMESTAMP(DATE_SUB(DATE_TRUNC(CURRENT_DATE(), MONTH), INTERVAL 12 MONTH))
  AND started_at <  TIMESTAMP(DATE_TRUNC(CURRENT_DATE(), MONTH))
  AND duration_ms IS NOT NULL
GROUP BY month
ORDER BY month;
```

---

## Outbound (forward) call latency only

```sql
SELECT
  TIMESTAMP_TRUNC(started_at, MONTH) AS month,
  APPROX_QUANTILES(forward_duration_ms, 100)[OFFSET(50)] AS p50_forward_ms,
  APPROX_QUANTILES(forward_duration_ms, 100)[OFFSET(90)] AS p90_forward_ms,
  APPROX_QUANTILES(forward_duration_ms, 100)[OFFSET(99)] AS p99_forward_ms,
  COUNT(*) AS row_count
FROM `cko-data-cl-prod-5361.source_flink.vault_network_tokens_forwardapirequest`
WHERE started_at >= TIMESTAMP(DATE_SUB(DATE_TRUNC(CURRENT_DATE(), MONTH), INTERVAL 12 MONTH))
  AND started_at <  TIMESTAMP(DATE_TRUNC(CURRENT_DATE(), MONTH))
  AND forward_duration_ms IS NOT NULL
GROUP BY month
ORDER BY month;
```

---

## Tips

- **Why BigQuery and not Datadog?** Datadog retains metrics for 3 months and logs for 15 days. For 12-month P99 data, BigQuery is the only option.
- `APPROX_QUANTILES(col, 100)[OFFSET(N)]` is BigQuery's idiomatic percentile — fast and cheap on large tables.
- `TIMESTAMP_SUB` does **not** support `MONTH` — use `DATE_SUB` on a `DATE` and cast back to `TIMESTAMP` (as shown above).
- The `WHERE` window above covers the **last 12 complete months** (excludes current partial month). Change upper bound to `CURRENT_TIMESTAMP()` to include current month.
- Filtering on `started_at` with a literal range enables **partition pruning** if the table is partitioned on it.

---

## Confluence reference

[Forward API BigQuery - Useful Queries](https://checkout.atlassian.net/wiki/spaces/CLSE/pages/8401191536/Forward+API+BigQuery+-+Useful+Queries)
