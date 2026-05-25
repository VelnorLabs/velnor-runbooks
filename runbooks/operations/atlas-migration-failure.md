---
layout: default
title: "Atlas Migration Failure"
nav_order: 2
parent: "Operations Runbooks"
adr_refs:
  - DEC-0031
  - DEC-0032
  - DEC-0035
  - DEC-0046
severity: SEV-2 (partial) / SEV-1 (all tenants blocked)
last_updated: 2026-05-25
---

# Atlas Migration Failure

> **Severity:** SEV-2 for partial failure (one or a few tenants); SEV-1 if all tenants are blocked or production data is at risk  
> **Audience:** On-call engineer  
> **Who is paged:** Architect (primary), Founder (escalation for SEV-1 or > 10 tenants affected)  
> **Expected duration:** 10–30 min for single-tenant remediation; 5 min for hash-mismatch fix; up to 60 min for down-migration  
> **Escalation chain:** Architect → Founder → Platform team

---

## Purpose

Step-by-step response for Atlas schema migration failures in the Velnor per-tenant schema pattern
(DEC-0035 rev. 2). Covers detection, triage, and three remediation paths: partial (single-tenant)
failure, hash-mismatch blocking all tenants, and down-migration when a schema rollback is required.

---

## Triggering alarm

| Field | Value |
|-------|-------|
| **CloudWatch alarm name** | `velnor-atlas-runner-error` |
| **Namespace / metric** | `VELNOR/Migrations` / `atlas_runner_exit_nonzero` |
| **Threshold** | Any non-zero exit in a 5-minute window |
| **SNS topic** | `arn:aws:sns:us-east-1:<ACCOUNT_ID>:velnor-ops-alert` |
| **Alert delivery** | SNS → Lambda subscriber → SES email to Architect (DEC-0046) |
| **Dashboard** | `velnor-dev-migrations` — runner exit codes, revision counts per tenant, error rate |

---

## Symptoms

Any of the following indicate an Atlas migration failure:

- Atlas runner job exits non-zero in CI or in production ECS
- `atlas_schema_revisions` table shows incomplete or dirty state for one or more tenants
- Service pods (ECS tasks) fail to start with **"schema version mismatch"** errors in their logs
- `atlas migrate status` output shows tenants at different revisions after a deploy

---

## Triage

### Step 1 — Locate the failing runner logs (< 3 min)

**Kubernetes / ECS (production):**

```bash
# ECS / CloudWatch (preferred in AWS environments)
aws logs filter-log-events \
  --log-group-name /velnor/dev/ecs/atlas-runner \
  --start-time $(date -v-30M +%s000) \
  --filter-pattern "ERROR" \
  --query 'events[*].{Time:timestamp,Message:message}' \
  --max-items 30

# Kubernetes (if running k8s)
kubectl logs -l job=atlas-runner --tail=100
```

**CI job logs:**

Check the most recent `atlas-runner` job in the GitHub Actions run. The exit code and error message
are printed as the last lines of the `atlas migrate apply` step.

### Step 2 — Check current migration state

```bash
# Replace $DATABASE_URL with the cluster writer DSN from Secrets Manager
# /velnor/dev/db/writer_url

atlas migrate status \
  --url "$DATABASE_URL" \
  --dir ./migrations
```

Expected healthy output: all tenants listed at the same revision, `Status: OK`.

Any of the following in the output indicate a problem:

| Output | Meaning |
|--------|---------|
| `Status: PENDING` | Migrations not yet applied to this tenant |
| `Status: DIRTY` | A previous migration run failed mid-apply |
| `Error: checksum mismatch` | Migration file was edited after the last `atlas migrate hash` run |
| `Error: unknown revision` | Revision table references a file that no longer exists |

### Step 3 — Identify the failing tenant

Look for the tenant slug in the error output. Atlas logs include the `search_path` or schema name:

```
applying migration to schema "velnor_t_acme-corp": ...
```

Note the tenant slug (e.g., `acme-corp`) for use in the remediation steps below.

---

## Remediation — Partial failure (single tenant)

Use this path when only one (or a small number of) tenants is blocked while others are healthy.

### Step 1 — Isolate the failing tenant

Confirm other tenants are at the expected revision before touching the failing one:

```bash
# Check all tenant schemas — output should show the same version for healthy tenants
psql "$DATABASE_URL" -c "
  SELECT schema_name
  FROM information_schema.schemata
  WHERE schema_name LIKE 'velnor_t_%'
  ORDER BY schema_name;
"
```

### Step 2 — Inspect the failing tenant's revision state

```bash
# Scope the status check to the specific tenant schema
atlas migrate status \
  --url "${DATABASE_URL}&search_path=velnor_t_${TENANT_SLUG}" \
  --dir ./migrations
```

### Step 3A — Checksum mismatch: regenerate hash and retry

If `atlas migrate status` reports a checksum mismatch, a migration file was edited after the last
`atlas migrate hash` run. Fix:

```bash
# In the velnor-plane-api repo (where migrations/ lives)
atlas migrate hash --dir ./migrations

# Commit the updated atlas.sum file
git add migrations/atlas.sum
git commit -m "fix: regenerate atlas migration hash after file edit"
git push
# Let CI re-run the atlas-runner job — it will apply to all affected tenants
```

If you need to immediately unblock a single tenant without waiting for CI:

```bash
# Re-run atlas runner for the specific tenant after the hash is regenerated
atlas migrate apply \
  --url "${DATABASE_URL}&search_path=velnor_t_${TENANT_SLUG}" \
  --dir ./migrations
```

### Step 3B — Dirty state: roll back to last clean revision

If `atlas migrate status` reports `Status: DIRTY`, the previous run failed mid-migration and left
the schema in an inconsistent state. Roll back to the last confirmed clean revision:

```bash
# 1. Find the last clean revision from the revisions table
psql "${DATABASE_URL}" -c "
  SELECT version, description, applied_at, error_stmt
  FROM velnor_t_${TENANT_SLUG}.atlas_schema_revisions
  ORDER BY applied_at DESC
  LIMIT 5;
"

# 2. Roll back to the last clean revision
#    Replace <previous_revision> with the version from the query above
atlas migrate set <previous_revision> \
  --url "${DATABASE_URL}&search_path=velnor_t_${TENANT_SLUG}" \
  --dir ./migrations

# 3. Re-run the migration for just this tenant
atlas migrate apply \
  --url "${DATABASE_URL}&search_path=velnor_t_${TENANT_SLUG}" \
  --dir ./migrations
```

### Step 4 — Verify the tenant is recovered

```bash
atlas migrate status \
  --url "${DATABASE_URL}&search_path=velnor_t_${TENANT_SLUG}" \
  --dir ./migrations
# Expected: Status: OK, all revisions applied
```

---

## Remediation — Hash mismatch (all tenants blocked)

**Cause:** A migration file in `migrations/` was edited after `atlas migrate hash` was last run.
Atlas refuses to apply any migration until the `atlas.sum` checksum file is regenerated.

**This affects all tenants simultaneously** — it is the most common cause of a full deploy blockage.

### Fix

```bash
# In the velnor-plane-api repo
cd /path/to/velnor-plane-api

# 1. Regenerate the checksum file
atlas migrate hash --dir ./migrations

# 2. Verify the sum file changed
git diff migrations/atlas.sum

# 3. Commit and push — CI will re-run the atlas-runner job against all tenants
git add migrations/atlas.sum
git commit -m "fix: regenerate atlas migration checksum after file edit"
git push
```

**CI will automatically apply the corrected migrations to all tenants** when the atlas-runner job
re-runs. Monitor the `velnor-dev-migrations` CloudWatch dashboard or the GitHub Actions run to
confirm all tenants reach the expected revision.

---

## Remediation — Down-migration required

Use this path when a migration must be reversed (e.g., a schema change causes application errors
and must be rolled back while a fix is prepared).

**Prerequisites:**

- `$DATABASE_URL` — writer DSN from Secrets Manager (`/velnor/dev/db/writer_url`)
- `$ATLAS_DEV_URL` — dev-database URL pointing at `atlas_dev_schema` (see atlas-runner ECS task
  environment: `ATLAS_DEV_URL`). This is required for Atlas to compute the diff for reversible migrations.

### Step 1 — Identify the target version to roll back to

```bash
# List applied revisions in reverse chronological order
atlas migrate status \
  --url "$DATABASE_URL" \
  --dir ./migrations
# Note the <target_version> — the last known-good revision before the bad migration
```

### Step 2 — Run the down-migration

```bash
atlas migrate down \
  --url "$DATABASE_URL" \
  --dev-url "$ATLAS_DEV_URL" \
  --dir ./migrations \
  --to-version <target_version>
```

**Note:** Down-migration rewrites schema state. Confirm with the Architect before running against
a tenant schema that contains real user data.

### Step 3 — Verify rollback

```bash
atlas migrate status \
  --url "$DATABASE_URL" \
  --dir ./migrations
# All tenants should now show <target_version> as the applied revision
```

---

## Post-recovery verification

After any of the three remediation paths, confirm the following before closing the incident:

```bash
# 1. All tenants at the same revision
atlas migrate status \
  --url "$DATABASE_URL" \
  --dir ./migrations
# Expected: all schemas at the same version, Status: OK

# 2. Smoke query — confirms the tenants table is accessible (basic schema health)
psql "$DATABASE_URL" -c "SELECT COUNT(*) FROM velnor.tenants;"

# 3. Verify ECS services have restarted successfully (if tasks failed to start)
aws ecs describe-services \
  --cluster velnor-dev \
  --services velnor-plane-api \
  --query 'services[*].{Name:serviceName,Running:runningCount,Desired:desiredCount,Status:status}'

# 4. Check the velnor-dev-migrations CloudWatch dashboard
# atlas_runner_exit_nonzero should return to 0
```

---

## Escalation

**Page the platform team immediately if any of the following are true:**

- More than **10 tenants** are blocked or in dirty state
- **Production data** may have been partially written by a failed migration
- Down-migration is required on a schema that has live tenant data
- The `atlas.sum` regeneration fix does not resolve the hash mismatch after CI reruns

Escalation path: Architect → Founder → Platform team (page via PagerDuty).

---

## Rollback criteria

- If `atlas migrate apply` fails a second time for the same tenant after a dirty-state rollback,
  **stop retrying** and escalate — do not loop on `migrate set` + `migrate apply` more than twice.
- If a down-migration removes a column or table that production code is already writing to,
  **halt ECS services immediately** before the migration completes to prevent data inconsistency:
  ```bash
  aws ecs update-service --cluster velnor-dev --service velnor-plane-api --desired-count 0
  ```

---

## Post-incident record

File within 48 hours of incident close.

```markdown
## Atlas Migration Failure Incident — <date>

**Failure type:** partial (single tenant) | hash mismatch (all tenants) | down-migration required
**Detected at:** <timestamp>
**Remediation started:** <timestamp>
**All tenants recovered:** <timestamp>
**Total blocked window:** <duration>

### Failing migration version: <version>
### Tenants affected: <count or "all">

### Timeline

| Time | Event |
|------|-------|
| ...  | ...   |

### Root cause

<hash mismatch | dirty state mid-migration | bad migration file | other>

### Tenant impact

- Tenants blocked: <count>
- Services unable to start: <list>
- Data loss or inconsistency: <none | describe>

### What went well

### What went poorly

### Action items

| Item | Owner | Due |
|------|-------|-----|
| Add atlas migrate hash check to PR lint step | Engineer | Next sprint |
| ...  | ...   | ... |
```

---

## Contacts

| Role | Contact | Escalation order |
|------|---------|-----------------|
| Architect (primary) | See PagerDuty / Slack | 1st |
| Founder | See PagerDuty / Slack | 2nd (mandatory for SEV-1 or > 10 tenants) |
| Platform team | See PagerDuty / Slack | 3rd (production data at risk) |
