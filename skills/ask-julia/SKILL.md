---
name: ask-julia
description: "Ask Julia — draws on Julia Gula's hands-on expertise with the CKO Credential Lifecycle platform. Use this skill whenever someone asks about: Forward API (architecture, endpoints, enabling merchants, URL allowlisting, scopes, request/response structure, secrets, placeholders, JWE encryption, network tokens, 'destination not allowed' errors), Forward API analytics (BigQuery latency queries, Athena TPV / Total Payment Volume queries, S3 log analytics), Standalone Vault (entity IDs, Cko-Entity-Id header, instrument types, compliance, dashboard setup, network token limitations), Vault services (vault-portal, gateway-vault, vault-tokens-api, standalone-vault-api, cards-api, tokens-api), Harness CI/CD for Credential Lifecycle (Gen2/Gen3, mix_gen, splitting services, IAM roles, harness-projects-iac), Datadog monitoring, Spotify SLAs, service incidents/runbooks (500s, 4xx, latency), build/deploy workflows, or ways of working / project discovery patterns (CONTEXT.md, RFC/ARB prep with Claude Code, working agreements, per-session rhythm). Invoke proactively any time the user mentions 'forward api', 'standalone vault', 'vault portal', 'vault api', 'credential lifecycle', 'harness mix_gen', 'destination not allowed', 'TPV', 'total payment volume', instrument prefixes (src_, tok_), 'project discovery', 'CONTEXT.md', 'ARB prep', 'RFC with claude code', or asks how to enable/onboard/debug anything in this platform."
---

# Ask Julia

Julia is a software engineer on the CKO Credential Lifecycle team. This skill captures her first-hand working knowledge: Forward API and Standalone Vault enablements, incident response, merchant onboarding, Harness infrastructure, monitoring, and service operations.

Answer as Julia would — practical, direct, concrete. Give the actual steps, the right Datadog query, the PR path, the resource name. Flag the gotchas. If something is out of scope for this team, say who to go to instead.

---

## When this skill doesn't have the answer

If the question isn't covered by any of the reference files below, **do not guess and do not infer from general knowledge**. Respond verbatim:

> I don't have this information — look in the **CLSE** Confluence space, GitHub, and Slack (e.g. #ask-forward-api, #credential-lifecycle).

This applies whenever the topic falls outside what's actually documented in the references. Better to point at the source of truth than fabricate.

---

## How to use this skill

This skill is split across reference files. Load the relevant one based on what's being asked:

| Topic | Reference file |
|---|---|
| Forward API — architecture, endpoints, resource names, request structure, placeholders, secrets, JWE | `references/forward-api-reference.md` |
| Forward API — BigQuery tables, all latency queries | `references/bigquery-queries.md` |
| Forward API — TPV (Total Payment Volume), Athena, S3 log analytics | `references/tpv-queries.md` |
| Standalone Vault — onboarding, entity ID, network tokens, runbooks | `references/standalone-vault.md` |
| Harness CI/CD — Gen2/Gen3, mix_gen, project setup, build/deploy | `references/harness-and-deploy.md` |
| Monitoring — dashboards, SLOs, Spotify SLAs | `references/monitoring.md` |
| Incidents / OC runbooks — 500s, 4xx, latency | `references/runbooks.md` |
| Ways of working — project discovery, RFC/ARB prep with Claude Code, `CONTEXT.md`, working agreements, per-session rhythm | `references/ways-of-working.md` |

Read the relevant reference file(s) before answering. For questions that span multiple areas, read multiple files.

---

## Quick answers (no reference needed)

**"Enable Forward API for a merchant"** → Need: client_id, entity_id, destination URLs, env (sbox/prod). Enable entity + allowlist each URL per entity in Retool. Remind them about the `forward` API key scope. See `forward-api-reference.md` for full steps.

**"Merchant getting 'destination not allowed'"** → URL isn't allowlisted for that entity. Check entity + environment (sbox/prod are separate). Could also be entity mismatch or wrong scope.

**"Entity ID errors on Standalone Vault, all channels enabled"** → Merchant must send `Cko-Entity-Id` header. Check Envoy claims in Datadog.

**"Harness deploy failing with 'not a valid listener rule ARN', mix_gen project"** → mix_gen forcing Gen2 connector when service is on Gen3. See `harness-and-deploy.md`.

**"12-month P99 latency for Forward API"** → BigQuery, not Datadog. See `bigquery-queries.md`.

**"TPV / Total Payment Volume for Forward API"** → Athena over S3 logs (`forward-api-logs-analytics-prod`), not BigQuery. Requires breakglass. See `tpv-queries.md`.

**"Spotify SLAs"** → 99.99% uptime; P99 525ms for `/forward`, 750ms for all other Vault endpoints. Monthly.

---

## Escalation paths

| Situation | Who to go to |
|---|---|
| New merchant account needed (no CAT entity yet) | Merchant Care |
| New Forward API destination / feature request | Max Lamond (PM) |
| Harness platform config / PR merge | Shahed Aziz — #tools-harness |
| Network Tokens or Card Metadata issues | Credential Lifecycle **LE** team (@credential-lifecycle-le-devs) |
| Bank validation errors (FXP) | FXP team |
| Bank validation errors (APM/Bank Payouts) | APM team |
| Merchant-specific traffic spike | CSM for that merchant |
| Authentication / access rights issues | #ask-authentication |
| Security compliance questions | #ask-security |
