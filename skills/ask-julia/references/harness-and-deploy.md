# Harness CI/CD & Build/Deploy — Reference

## Services covered

| Repo | What it builds | Notes |
|---|---|---|
| `vault-portal` | Single image | Pushes to Gen2 and Gen3 ECR |
| `gateway-vault` | Two images: Vault API + Vault Account Configuration API | Gen3 ECR: `891377407345`. Also has a **Publish SDK** workflow for Vault.Sdk NuGet |
| `vault-tokens-api` | Single image | Standard pattern |

---

## Gen2 vs Gen3

| Generation | AWS Account ID | Region |
|---|---|---|
| Gen2 | `851392519502` | eu-west-1 (legacy) |
| Gen3 | `361769567558` | All regions (new) |

---

## mix_gen — when to use it and when not to

`schema_type: mix_gen` in a Harness project means: **eu-west-1 is on Gen2, all other regions are on Gen3**.

**Only use `mix_gen` when this is literally true for the service.**

If a service has been fully migrated to Gen3 in all regions (including eu-west-1), it must **not** use `mix_gen` — it should be in its own project with pure Gen3 config. Using `mix_gen` on a fully-Gen3 service causes deploys to fail with:

```
not a valid listener rule ARN
```

This is because `mix_gen` forces the Gen2 connector for eu-west-1 instead of the Gen3 connector.

### Fix: split into a separate project

1. Raise a PR in **`harness-projects-iac`** → `terraform/200-service-group-projects/cko-credential-lifecycle/`
2. Create a new project config for the service without `mix_gen`
3. Contact **Shahed Aziz** in **#tools-harness** to merge and apply

---

## harness-projects-iac

Repo: `harness-projects-iac`  
Path for CL services: `terraform/200-service-group-projects/cko-credential-lifecycle/`

All Harness project configuration lives here. To add a new service, split a service out, or change `schema_type` — raise a PR here.

---

## Harness project links

| Project | Link |
|---|---|
| vault_gen2 (Standalone Vault + older services) | [Harness](https://app.harness.io/ng/account/x9vTEK0gRkadO7XKkBzdTw/all/cd/orgs/credential_lifecycle/projects/vault_gen2/overview) |

---

## How to test a branch (pre-release)

1. Create and push a pre-release tag on your branch — format: `LatestVersion-alpha0.x.x` (e.g. `1.2.3-alpha0.1`)
   ```
   git tag 1.2.3-alpha0.1 && git push origin 1.2.3-alpha0.1
   ```
2. **Build & Test & Deploy** workflow triggers on tag push → builds, tests, pushes Docker image to Gen3 ECR
3. In Harness, confirm the tag and trigger a deployment to QA/Sbox

---

## How to create a production release

1. **Create a Jira Release** in [CLSE project](https://checkout.atlassian.net/projects/CLSE?selectedItem=com.atlassian.jira.jira-projects-plugin%3Arelease-page):
   - Title format: `Service Name Release Number` (e.g. `Tokens Api 1.27.0`)
   - Tag relevant tickets, add summary of changes
2. **Get approval** from a senior engineer on the team
3. **Create a GitHub release:**
   - Go to Releases → Draft a new release
   - Create a new tag with the full version (e.g. `1.2.3`)
   - Title: `Release tag` (or team's agreed format)
   - Add description (link to Jira, list of changes)
   - Publish → triggers Build & Test & Deploy → images pushed to Gen3 ECR
4. **Deploy via Harness** (links in each repo's README under Deployment/Deploying)
5. **Monitor:** Datadog dashboards + service health/error rates

---

## Quick reference

| Task | Steps |
|---|---|
| Test branch in QA/Sbox | Tag with `LatestVersion-alpha0.x.x`, push. Workflow pushes to ECR → deploy via Harness |
| Release to production | GitHub release with full version tag + Jira ticket + senior engineer approval + Harness deploy |
| Run acceptance tests locally | `docker-compose` up, then run acceptance test Docker image on compose network (see repo README) |
| Fix "not a valid listener rule ARN" | Service is on Gen3 but project uses `mix_gen`. Raise PR in `harness-projects-iac` to split it out |
| Add/change Harness project config | PR in `harness-projects-iac` → `terraform/200-service-group-projects/cko-credential-lifecycle/` |

---

## Confluence references

- [How to build, test, and deploy](https://checkout.atlassian.net/wiki/spaces/CLSE/pages/8127414277/How+to+build+test+and+deploy)
