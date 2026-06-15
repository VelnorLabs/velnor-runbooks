---
layout: default
title: "Dev DB Validation (bootstrap + migrate) via bastion"
nav_order: 3
parent: "Operations Runbooks"
adr_refs:
  - DEC-0035
  - DEC-0057
  - DEC-0061
severity: maintenance
last_updated: 2026-06-14
---

# Dev DB Validation — Aurora bootstrap + Atlas migrate

How to validate the dev Aurora cluster from a laptop, using the SSM bastion
(`accounts/dev/bastion/` — LE-013). Covers the two Wave-0 checks that were
blocked on DB reachability:

- **LE-022 / T-W0-024** — Aurora bootstrap: extensions, shared schema, registry
  tables, Atlas dev schema, RLS.
- **LE-020 / T-W0-025** — Atlas migrate runner: correct CLI invocation +
  per-tenant schema apply.

> **Reachability:** Aurora and the RDS Proxy sit on private-data subnets with no
> public path. All commands below run over an SSM port-forward through the
> bastion — see `velnor-iac/accounts/dev/bastion/README.md` for the two tunnel
> modes (Aurora-direct password / RDS-Proxy IAM token, DEC-0061).

## 0. Open the tunnel

Use **Aurora-direct** for these checks (DDL + inspection; pooler-free is
preferred):

```bash
BID=$(cd velnor-iac/accounts/dev/bastion && tofu output -raw bastion_instance_id)
AURORA=$(cd velnor-iac/accounts/dev/aurora && tofu output -raw cluster_endpoint)

aws ssm start-session --region us-east-1 --profile velnor-dev \
  --target "$BID" \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters "{\"host\":[\"$AURORA\"],\"portNumber\":[\"5432\"],\"localPortNumber\":[\"5432\"]}"
```

Leave running. In another terminal, fetch the master password. The cluster runs
with `manage_master_user_password = true`, so AWS owns a rotating Secrets
Manager secret (named `rds!cluster-…`); resolve its ARN from the cluster rather
than guessing the name:

```bash
# Clear any stale/whitespace-corrupted password from a prior attempt — a
# lingering PGPASSWORD silently overrides the fetch and causes a confusing
# "password authentication failed" even with correct creds.
unset PGPASSWORD

# 1. ARN of the RDS-managed master-user secret
SECRET_ARN=$(aws rds describe-db-clusters \
  --region us-east-1 --profile velnor-dev \
  --db-cluster-identifier velnor-dev \
  --query 'DBClusters[0].MasterUserSecret.SecretArn' --output text)

# 2. Print the password (one-off) and confirm the secret's user is velnor_admin
aws secretsmanager get-secret-value \
  --region us-east-1 --profile velnor-dev --secret-id "$SECRET_ARN" \
  --query 'SecretString' --output text \
  | python3 -c 'import sys,json; d=json.load(sys.stdin); print("user:", d["username"]); print("password:", d["password"])'
```

Then connect with a **conninfo string** and paste the password at the prompt —
do NOT rely on `PGPASSWORD` (env-var contamination is the #1 cause of a false
"password authentication failed" here):

```bash
export PGSSLMODE=require    # TLS is mandatory (proxy/Aurora reject plaintext)
PSQL='psql "host=127.0.0.1 port=5432 dbname=postgres user=velnor_admin sslmode=require" -v ON_ERROR_STOP=1'

# interactive: paste the password from step 2 at the "Password:" prompt
psql "host=127.0.0.1 port=5432 dbname=postgres user=velnor_admin sslmode=require"
```

> Use `127.0.0.1`, not `localhost` — `localhost` can resolve to the Unix socket
> (`/tmp/.s.PGSQL.5432`) and bypass the tunnel. The RDS Proxy is `iam_auth =
> REQUIRED` and rejects passwords entirely, so this password path only works
> against the **Aurora cluster endpoint** tunnel, not the proxy (see bastion
> README for the proxy IAM-token path).
>
> Requires `secretsmanager:GetSecretValue` on that secret **and** `kms:Decrypt`
> on the key encrypting it (RDS-managed secrets are KMS-encrypted) — the account
> admin / `velnor-dev` profile has both. Never paste the password into a file or
> the runbook; it rotates.

## 1. LE-022 — Aurora bootstrap verification

The authoritative bootstrap is `velnor-iac/accounts/dev/rds/bootstrap.sql`
(idempotent; applied by the bootstrap Lambda). Expected state:

### Extensions (3)

```sql
SELECT extname FROM pg_extension
WHERE extname IN ('vector','pg_trgm','pg_stat_statements')
ORDER BY extname;
-- expect 3 rows: pg_stat_statements, pg_trgm, vector
```

> Note: the pgvector **extension** is named `vector` (not `pgvector`) — that
> naming mismatch was an earlier validation false-negative.

### Schemas (2)

```sql
SELECT nspname FROM pg_namespace
WHERE nspname IN ('velnor','atlas_dev_schema') ORDER BY nspname;
-- expect 2 rows: atlas_dev_schema, velnor
```

### Registry + audit tables (4) with RLS

```sql
SELECT relname, relrowsecurity FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname = 'velnor'
  AND relname IN ('tenants','golden_template','tool_invocation_log','audit_event')
ORDER BY relname;
-- expect 4 rows; relrowsecurity = t for tenants, tool_invocation_log, audit_event
```

### Seed data — expected EMPTY at Wave-0

```sql
SELECT count(*) FROM velnor.golden_template;   -- expect 0 at Wave-0
SELECT count(*) FROM velnor.tenants;           -- expect 0 until first tenant provisioned
```

> **The "64 seed entries return 0" finding is EXPECTED, not a defect.**
> `bootstrap.sql` deliberately seeds no rows. The golden vertical catalogs
> (Gym v1, Dance v1 — the ~64 entries) live in `velnor-schemas`
> (`postgres/golden_templates/`) and are **Wave-1** work (T-W1-003), loaded via
> the per-tenant template fork at registration, not by the Wave-0 bootstrap.

**Pass criteria (Wave-0):** 3 extensions + 2 schemas + 4 tables (3 with RLS) present; seed tables empty. If extensions/schemas/tables are missing, re-run the bootstrap Lambda (`accounts/dev/rds/` — invoke `bootstrap_runner`) and re-check.

## 2. LE-020 — Atlas migrate runner

Two things to know before running:

- The earlier "migrate demands `Tenant slug (required)`" was a
  **missing-required-flag invocation**, not a bug — every sub-command needs
  `-tenant`/`-tenants`.
- The migrate runner does **not create the tenant schema** — it applies the
  shared catalog into an *existing* schema (the slug **is** the schema name,
  via Atlas `--schema`; no `tenant_` prefix is added). So at Wave-0, with no
  tenant provisioned, you create a throwaway schema first.
- The runner's dir bug (it looked for a non-existent per-slug sub-dir) is fixed
  in plane-api PR #11 — pull `main` before building.

### Self-contained validation (no real tenant needed)

```bash
cd velnor-plane-api && git pull origin main      # include PR #11
go build -o ./bin/migrate ./cmd/migrate

# DSN over the open tunnel (127.0.0.1, sslmode=require). Use a conninfo-style
# DSN; the password is the master secret (see §0) or omit and rely on a prompt.
export MIGRATE_DSN="postgres://velnor_admin@127.0.0.1:5432/postgres?sslmode=require"
export PGPASSWORD='<paste master password>'      # libpq picks this up for the DSN

# 1. Create a throwaway tenant schema (the runner expects it to exist).
psql "host=127.0.0.1 port=5432 dbname=postgres user=velnor_admin sslmode=require" \
  -c 'CREATE SCHEMA IF NOT EXISTS val_smoke;'

# 2. Apply the shared catalog into it (-tenant = the schema name).
./bin/migrate apply -tenant val_smoke -dsn "$MIGRATE_DSN"   # expect exit 0

# 3. Status
./bin/migrate status -tenant val_smoke -dsn "$MIGRATE_DSN"
```

### Verify

```sql
-- the catalog's tables landed in the schema (not the public/velnor schema)
SELECT count(*) FROM information_schema.tables WHERE table_schema = 'val_smoke';
-- Atlas revision tracking present in THIS schema (slug == schema name)
SELECT count(*) FROM val_smoke.atlas_schema_revisions;   -- > 0 after apply
```

**Pass criteria:** `migrate apply -tenant val_smoke` exits 0; `val_smoke`
schema has the catalog tables + `atlas_schema_revisions` rows.

### Clean up

```sql
DROP SCHEMA val_smoke CASCADE;
```

### Other sub-commands (reference)

```bash
# Many tenants in parallel — -tenants (CSV) REQUIRED (schemas must pre-exist)
./bin/migrate apply-all -tenants t1,t2 -dsn "$MIGRATE_DSN" -workers 10
# Roll back — -tenant + -to-version REQUIRED; -dev-url points at atlas_dev_schema
./bin/migrate down -tenant val_smoke -to-version 20260524000001 -dsn "$MIGRATE_DSN" \
  -dev-url "postgres://velnor_admin@127.0.0.1:5432/postgres?search_path=atlas_dev_schema&sslmode=require"
```

- `-skip-lock` defaults `true` (S-04: required for parallel per-schema apply).

## 3. Known separate gaps (NOT DB-reachability)

These were bundled into LE-022 but are independent of the bastion — flag/track
separately, don't expect them to pass here:

- **CloudWatch migrate metric emitting 0** (LE-020) — verify the metric is wired
  and the runner has `cloudwatch:PutMetricData`; may be never-wired, not a
  perms issue.
- **PRETOKEN Lambda absent** (T-W0-021) — genuinely missing, not unreachable.
- **velnor-iac-artifacts bucket / T-W0-012 deploy pipeline** — verify existence
  independently.

## 4. Close the tunnel

`Ctrl-C` the `start-session` terminal. Tear the bastion down at Wave-0 close:
`cd velnor-iac/accounts/dev/bastion && tofu destroy`.
