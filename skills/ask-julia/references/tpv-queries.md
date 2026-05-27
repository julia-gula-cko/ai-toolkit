# Forward API — TPV (Total Payment Volume)

TPV is computed from the **raw destination request/response logs** archived to S3, not from BigQuery. This means: Athena queries, breakglass access required, and a small amount of regex work to extract amount/currency from heterogeneous merchant payload formats.

---

## Prereqs

1. **Breakglass access** to the `forward-api-logs-analytics-prod` AWS account (logs are PII-adjacent — access is gated).
2. Athena workgroup in that account.
3. The S3 bucket is partitioned by year/month: `s3://forward-api-logs-analytics-prod/YYYY/MM/`.

---

## Step 1 — Create an external table for the period

Pick the period you want (single month is typical). Replace `<period>` (e.g. `april_2026`, `q2_2026`) and the S3 `LOCATION` accordingly.

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS forward_api_logs_analytics_<period> (
  request_id STRING,
  forward_request_body STRING,
  forward_request_headers MAP<STRING, STRING>,
  forward_request_query_parameters MAP<STRING, STRING>,
  forward_response_body STRING,
  forward_response_headers MAP<STRING, ARRAY<STRING>>
)
PARTITIONED BY (dt STRING)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
WITH SERDEPROPERTIES ('ignore.malformed.json' = 'true')
LOCATION 's3://forward-api-logs-analytics-prod/<YYYY>/<MM>/'
TBLPROPERTIES ('classification' = 'json');
```

Example `LOCATION` values:
- April 2026 → `s3://forward-api-logs-analytics-prod/2026/04/`
- May 2026 → `s3://forward-api-logs-analytics-prod/2026/05/`

> If you want a multi-month range, point the LOCATION at `s3://forward-api-logs-analytics-prod/<YYYY>/` and partition pruning will handle it.

---

## Step 2 — Extract amount + currency per request

Merchant destination payloads aren't uniform — some are JSON, some XML, some form-encoded. This query handles the common shapes seen on Forward API.

```sql
WITH extracted_payloads AS (
    SELECT
        request_id,
        forward_request_body,
        regexp_extract(forward_request_body, '(?:<amount>|"(?:amount|value|amount_total)"\s*:\s*"?)([0-9]+\.?[0-9]*)', 1) AS raw_amount_str,
        COALESCE(
            regexp_extract(forward_request_body, '(?:<currency>|"(?:currency|currencyCode|currency_code)"\s*:\s*")([A-Z]{3})', 1),
            regexp_extract(forward_request_body, '<merchant-account-id>.*_([A-Z]{3})<', 1)
        ) AS currency,
        CASE
            WHEN forward_request_body LIKE '%payment_method%' THEN 'Card Management / Migration'
            WHEN forward_request_body LIKE '%<transaction>%' THEN 'XML Transaction'
            WHEN forward_request_body LIKE '%"payment":%' AND forward_request_body LIKE '%"amount_total"%' THEN 'JSON Payment Object'
            WHEN forward_request_body LIKE '%"order":%' AND forward_request_body LIKE '%"action"%' THEN 'JSON Purchase Order'
            WHEN forward_request_body LIKE '%"order":%' THEN 'JSON Order'
            ELSE 'Other / Unknown'
        END AS payload_category
    FROM
        forward_db.forward_api_logs_analytics_<period>
    WHERE
        (forward_request_body LIKE '%amount%'
         OR forward_request_body LIKE '%value%'
         OR forward_request_body LIKE '%currency%'
         OR forward_request_body LIKE '%merchant-account-id%'
         OR forward_request_body LIKE '%payment_method%'
         OR forward_request_body LIKE '%currencyCode%'
         OR forward_request_body LIKE '%amount_total%'
         OR forward_request_body LIKE '%currency_code%')
)

SELECT
    request_id,
    currency,
    CASE
        WHEN raw_amount_str IS NULL OR raw_amount_str = '' THEN NULL
        WHEN raw_amount_str NOT LIKE '%.%' THEN CAST(raw_amount_str AS DOUBLE) / 100.0
        ELSE CAST(raw_amount_str AS DOUBLE)
    END AS amount
FROM
    extracted_payloads
```

### Amount handling
- If the extracted amount has **no decimal point**, it's assumed to be **minor units** (cents) and divided by 100.
- If it already has a decimal, it's used as-is.
- Currency-specific minor units (e.g. JPY has no minor units) are **not** corrected here — bear in mind when totalling by currency.

---

## Step 3 — Aggregate to TPV

To get TPV totals per currency from the extraction:

```sql
SELECT
    currency,
    COUNT(*) AS tx_count,
    SUM(amount) AS tpv
FROM (
    -- paste the Step 2 query here, or save it as a view
) extracted
WHERE amount IS NOT NULL
  AND currency IS NOT NULL
GROUP BY currency
ORDER BY tpv DESC;
```

---

## Gotchas

- **The bucket name is `forward-api-logs-analytics-prod`**, not the older `forward-api-logs-archive-prod` mentioned elsewhere. The analytics bucket is the partitioned one — use it.
- **Payload heterogeneity** — the regex set above covers the common shapes but won't catch everything. Spot-check the `Other / Unknown` category and extend if a meaningful chunk lands there.
- **Currency assumption (minor units)** — the `/100` correction assumes 2-decimal currencies. JPY, KRW etc. will be off by 100x. Filter or correct per currency if accuracy matters.
- **Breakglass time-boxed** — you only have a limited window once access is granted. Create the table and run extraction in the same session, or persist results into a non-PII table for later analysis.
- **One table per period** — keep the period suffix in the table name (`_april_2026`, `_q2_2026`) so old extractions don't get stale. Drop the table when done.

---

## Reference ticket

Original example for April: [CLSE-2321](https://checkout.atlassian.net/browse/CLSE-2321).
