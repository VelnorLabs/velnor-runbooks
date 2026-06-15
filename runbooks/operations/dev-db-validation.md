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
# 1. ARN of the RDS-managed master-user secret
SECRET_ARN=$(aws rds describe-db-clusters \
  --region us-east-1 --profile velnor-dev \
  --db-cluster-identifier velnor-dev \
  --query 'DBClusters[0].MasterUserSecret.SecretArn' --output text)

# 2. Pull the password out of the secret JSON {"username":...,"password":...}
export PGPASSWORD=$(aws secretsmanager get-secret-value \
  --region us-east-1 --profile velnor-dev --secret-id "$SECRET_ARN" \
  --query 'SecretString' --output text \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["password"])')

export PGSSLMODE=require    # TLS is mandatory (proxy/Aurora reject plaintext)
PSQL='psql -h localhost -p 5432 -U velnor_admin -d postgres -v ON_ERROR_STOP=1'
```

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

The earlier failure ("migrate demands `Tenant slug (required)` / `-to-version
(required)`") was a **missing-required-flag invocation**, not a bug. Correct
usage (`velnor-plane-api/cmd/migrate`):

```bash
# Build
cd velnor-plane-api && go build -o ./bin/migrate ./cmd/migrate

# DSN over the open tunnel (sslmode=require)
export MIGRATE_DSN='postgres://velnor_admin:'"$PGPASSWORD"'@localhost:5432/postgres?sslmode=require'

# Apply for ONE tenant — -tenant is REQUIRED
./bin/migrate apply   -tenant acme            -dsn "$MIGRATE_DSN"

# Apply for MANY tenants in parallel — -tenants (CSV) REQUIRED
./bin/migrate apply-all -tenants acme,globex   -dsn "$MIGRATE_DSN" -workers 10

# Status
./bin/migrate status  -tenant acme            -dsn "$MIGRATE_DSN"

# Roll back — both -tenant and -to-version REQUIRED
./bin/migrate down    -tenant acme -to-version 20260101000000 -dsn "$MIGRATE_DSN" \
  -dev-url 'postgres://...localhost.../atlas_dev_schema?sslmode=require'
```

Flag notes:
- `-skip-lock` defaults `true` (S-04: required for 100-tenant parallel apply).
- `down` needs `-dev-url` pointing at `atlas_dev_schema` (created by bootstrap).

### Verify per-tenant schema apply

After `apply -tenant acme`:

```sql
-- schema-per-tenant (DEC-0035 rev.2): a tenant_<slug> schema exists
SELECT nspname FROM pg_namespace WHERE nspname = 'tenant_acme';
-- Atlas revision tracking present
SELECT count(*) FROM tenant_acme.atlas_schema_revisions;   -- > 0 after apply
```

**Pass criteria:** `migrate apply -tenant <slug>` exits 0; `tenant_<slug>`
schema + `atlas_schema_revisions` rows exist.

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
