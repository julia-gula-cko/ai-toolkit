# Forward API — TPV (Total Payment Volume)

TPV is computed from the **raw destination request/response logs** archived to S3, not from BigQuery. This means: Athena queries, breakglass access required, and a small amount of regex work to extract amount/currency from heterogeneous merchant payload formats.

The flow is **three steps**: (1) create an external table over the raw S3 logs for the period, (2) run a CTAS that extracts amount/currency into a partitioned PARQUET results table (~40 min), (3) aggregate that results table to TPV.

---

## Prereqs

1. **Breakglass access** to the `forward-api-logs-analytics-prod` AWS account (logs are PII-adjacent — access is gated).
2. Athena workgroup in that account.
3. The S3 bucket is partitioned by year/month: `s3://forward-api-logs-analytics-prod/YYYY/MM/`.

---

## Step 1 — Create an external table for the period

Change the `LOCATION` to the month you want to query, and give the table a new name.

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS
  forward_db.your_table_name
  ( request_id string, forward_request_body string, forward_request_headers string, forward_response_body string, forward_response_headers string )
  ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
  WITH SERDEPROPERTIES ('ignore.malformed.json' = 'true')
  LOCATION 's3://forward-api-logs-analytics-prod/2026/04/'
```

Example `LOCATION` values:
- April 2026 → `s3://forward-api-logs-analytics-prod/2026/04/`
- May 2026 → `s3://forward-api-logs-analytics-prod/2026/05/`

---

## Step 2 — Extract amount + currency into a partitioned results table

This is a CTAS (`CREATE TABLE ... AS`) that materializes the extracted amount/currency into a partitioned PARQUET table. **This query runs for around 40 minutes.**

Change:
- the name of the table being created (`your_partitioned_table_name`)
- the `external_location` to where you want the partitioned results to land
- the `FROM` to the name of the table you created in Step 1

Merchant destination payloads aren't uniform — some are JSON, some XML, some form-encoded. The regex below handles the common shapes seen on Forward API.

```sql
CREATE TABLE forward_db.your_partitioned_table_name
  WITH (
      format = 'PARQUET',
      external_location = 's3://temp-forward-results/your-locaction-folder-name/'
  ) AS
  WITH extracted_payloads AS (
      SELECT
          request_id,
          regexp_extract(forward_request_body, '(?:<amount>|"(?:amount|value|amount_total)"\s*:\s*"?)([0-9]+\.?[0-9]*)', 1) AS raw_amount_str,
          COALESCE(
              regexp_extract(forward_request_body, '(?:<currency>|"(?:currency|currencyCode|currency_code)"\s*:\s*")([A-Z]{3})', 1),
              regexp_extract(forward_request_body, '<merchant-account-id>.*_([A-Z]{3})<', 1)
          ) AS currency
      FROM forward_db.name_of_table_you_created_in_above_query
      WHERE forward_request_body LIKE '%amount%'
         OR forward_request_body LIKE '%value%'
         OR forward_request_body LIKE '%currency%'
         OR forward_request_body LIKE '%merchant-account-id%'
         OR forward_request_body LIKE '%payment_method%'
  )
  SELECT
      request_id,
      currency,
      CASE
          WHEN raw_amount_str IS NULL OR raw_amount_str = '' THEN NULL
          WHEN raw_amount_str NOT LIKE '%.%' THEN CAST(raw_amount_str AS DOUBLE) / 100.0
          ELSE CAST(raw_amount_str AS DOUBLE)
      END AS amount
  FROM extracted_payloads
```

### Amount handling
- If the extracted amount has **no decimal point**, it's assumed to be **minor units** (cents) and divided by 100.
- If it already has a decimal, it's used as-is.
- Currency-specific minor units (e.g. JPY has no minor units) are **not** corrected here — bear in mind when totalling by currency.

---

## Step 3 — Aggregate to TPV

You now have a partitioned results table. Query it for TPV totals per currency (point `FROM` at the partitioned table you created in Step 2):

```sql
SELECT
      currency,
      SUM(amount) AS total_amount,
      COUNT(*) AS transaction_count
  FROM forward_db.forward_api_logs_april_results
  GROUP BY currency
  ORDER BY total_amount DESC
```

---

## Gotchas

- **The bucket name is `forward-api-logs-analytics-prod`**, not the older `forward-api-logs-archive-prod` mentioned elsewhere. The analytics bucket is the partitioned one — use it.
- **Step 2 runs ~40 minutes** — it's a full extraction over the month's raw logs. Kick it off and don't expect an instant result.
- **Payload heterogeneity** — the regex set above covers the common shapes but won't catch everything. Spot-check requests where `amount`/`currency` came back NULL and extend the regex if a meaningful chunk is missed.
- **Currency assumption (minor units)** — the `/100` correction assumes 2-decimal currencies. JPY, KRW etc. will be off by 100x. Filter or correct per currency if accuracy matters.
- **Breakglass time-boxed** — you only have a limited window once access is granted. Because Step 2 takes ~40 min, persisting results to the PARQUET results table (rather than re-running ad-hoc) lets you do later analysis without re-acquiring access.
- **Name your tables per period** — keep the period in the table/results names (`forward_api_logs_april_results`, etc.) so old extractions don't get confused. Clean up when done.

---

## Source of truth

Confluence: [Athena Queries Forward API TPV](https://checkout.atlassian.net/wiki/spaces/CLSE/pages/8455782637/Athena+Queries+Forward+API+TPV) (CLSE space).

Original example ticket for April: [CLSE-2321](https://checkout.atlassian.net/browse/CLSE-2321).
