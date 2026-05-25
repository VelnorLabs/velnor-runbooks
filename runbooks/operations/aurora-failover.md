---
layout: default
title: "Aurora Failover (Planned + Unplanned)"
nav_order: 1
parent: "Operations Runbooks"
adr_refs:
  - DEC-0031
  - DEC-0032
  - DEC-0046
severity: SEV-1 (unplanned) / maintenance (planned)
last_updated: 2026-05-24
---

# Aurora Failover — Planned + Unplanned

> **Severity:** SEV-1 for unplanned failover (full write outage until promotion completes)  
> **Who is paged:** Architect (primary), Founder (escalation for unplanned)  
> **Expected duration:** < 60 seconds RDS promotion + ~2 min application recovery (connection-pool reconnect)  
> **Escalation chain:** Architect → Founder → AWS Support

---

## Purpose

Step-by-step procedure for two Aurora Postgres Serverless v2 failover scenarios:

1. **Planned failover** — intentional writer-instance reboot with failover (maintenance, upgrade, AZ rebalancing).
2. **Unplanned failover** — reader promoted to writer due to writer-instance failure, AZ event, or hardware fault.

Stack context (DEC-0031 / DEC-0032):
- **Cluster:** `velnor-dev-aurora` (Aurora Postgres 16 Serverless v2, `us-east-1`)
- **Writer endpoint:** `velnor-dev-aurora.cluster-<ID>.us-east-1.rds.amazonaws.com`
- **Reader endpoint:** `velnor-dev-aurora.cluster-ro-<ID>.us-east-1.rds.amazonaws.com`
- **Multi-AZ:** Yes — one writer (AZ-a), one reader (AZ-b) minimum at V1
- **Schema pattern:** per-tenant schemas `velnor_t_{slug}` (DEC-0035 rev. 2)

---

## Triggering alarm

| Field | Value |
|-------|-------|
| **CloudWatch alarm name (unplanned)** | `velnor-aurora-writer-unavailable` |
| **Namespace / metric** | `AWS/RDS` / `DatabaseConnections` (writer drops to 0 on failover start) |
| **Threshold** | Writer connections = 0 for >= 2 evaluation periods (2 min) |
| **SNS topic** | `arn:aws:sns:us-east-1:<ACCOUNT_ID>:velnor-ops-alert` |
| **Alert delivery** | SNS → Lambda → SES email to Architect (DEC-0046) |
| **Dashboard** | `velnor-dev-aurora` — ACU, connection count, slow queries, scale-to-zero events |

---

## Part 1 — Planned Failover

Use this procedure for scheduled maintenance (e.g., minor version upgrades, instance class changes, AZ rebalancing).

### Pre-flight checklist

Run these before triggering the failover:

- [ ] Confirm maintenance window is agreed with the founder (or no active tenants expected).
- [ ] Verify all active ECS services are connection-pool-aware (they should use the cluster writer endpoint, not an instance endpoint).
- [ ] Check for any long-running transactions: `SELECT pid, now() - pg_stat_activity.query_start AS duration, query FROM pg_stat_activity WHERE state = 'active' ORDER BY duration DESC LIMIT 10;`
- [ ] Note the current writer AZ: `aws rds describe-db-clusters --db-cluster-identifier velnor-dev-aurora --query 'DBClusters[0].AvailabilityZones'`
- [ ] Open the `velnor-dev-aurora` CloudWatch dashboard in a second browser tab to monitor live.

### Step 1 — Trigger planned failover

```bash
# Option A: Console
# RDS → Clusters → velnor-dev-aurora → Actions → Failover

# Option B: CLI (preferred — auditable)
aws rds failover-db-cluster \
  --db-cluster-identifier velnor-dev-aurora \
  --region us-east-1
```

Expected response: `{"DBCluster": {"Status": "failing-over", ...}}`

### Step 2 — Monitor promotion

```bash
# Poll cluster status (runs every 10 seconds)
watch -n 10 'aws rds describe-db-clusters \
  --db-cluster-identifier velnor-dev-aurora \
  --query "DBClusters[0].{Status:Status,Members:DBClusterMembers[*].{ID:DBInstanceIdentifier,Role:DBClusterParameterGroupStatus,Writer:IsClusterWriter}}" \
  --output json'
```

Expected timeline:
- T+0: Status = `failing-over`
- T+20–60s: Former reader promoted to writer (IsClusterWriter = true flips)
- T+60s: Status = `available`

**Application connection pools** will see a brief TCP reset (~1–3 seconds) as the writer endpoint DNS record updates. Services using HikariCP (Kotlin/Spring Boot DEC-0023) or psycopg2 connection pools with retry logic should auto-reconnect within 5–10 seconds.

### Step 3 — Verify post-failover

```bash
# 1. Confirm new writer instance
aws rds describe-db-cluster-members \
  --db-cluster-identifier velnor-dev-aurora \
  --query 'DBClusterMembers[?IsClusterWriter==`true`].DBInstanceIdentifier'

# 2. Confirm the new writer is in a different AZ than before
aws rds describe-db-instances \
  --db-instance-identifier <new-writer-instance-id> \
  --query 'DBInstances[0].AvailabilityZone'

# 3. Run smoke check query via writer endpoint
psql "host=velnor-dev-aurora.cluster-<ID>.us-east-1.rds.amazonaws.com \
      port=5432 dbname=velnor user=velnor_app sslmode=require" \
  -c "SELECT current_database(), pg_is_in_recovery();"
# Expected: pg_is_in_recovery() = false (writer is not in recovery mode)

# 4. Verify application connections restored
# Check CloudWatch → velnor-dev-aurora → DatabaseConnections
# Should be back above 0 within 2 minutes of promotion
```

---

## Part 2 — Unplanned Failover

An unplanned failover fires the `velnor-aurora-writer-unavailable` alarm. The reader is automatically promoted by RDS — no manual trigger needed. This procedure covers how to monitor, validate, and recover.

### Step 1 — Assess the situation (< 3 min from alarm)

```bash
# 1. Check cluster status
aws rds describe-db-clusters \
  --db-cluster-identifier velnor-dev-aurora \
  --query 'DBClusters[0].{Status:Status,MultiAZ:MultiAZ,Endpoint:Endpoint,ReaderEndpoint:ReaderEndpoint}'

# 2. Check which instance is the current writer
aws rds describe-db-cluster-members \
  --db-cluster-identifier velnor-dev-aurora \
  --query 'DBClusterMembers[*].{ID:DBInstanceIdentifier,Writer:IsClusterWriter,Status:DBClusterParameterGroupStatus}'

# 3. Check the RDS Events for the triggering event
aws rds describe-events \
  --source-identifier velnor-dev-aurora \
  --source-type db-cluster \
  --duration 60 \
  --query 'Events[*].{Message:Message,Date:Date}'
```

### Step 2 — Multi-AZ promotion semantics

Aurora Postgres Serverless v2 unplanned failover:

1. RDS detects writer unavailability (health check failure or AZ event).
2. The replica in the remaining healthy AZ is promoted to writer automatically.
3. The cluster endpoint DNS record is updated (TTL is low, ~5 seconds).
4. The original writer instance may be replaced in the failed AZ (RDS-managed, no operator action needed).

**Operator action required:** None for the promotion itself. Your job is to:
- Confirm promotion completed.
- Verify application layers reconnected.
- File an incident record.
- Investigate root cause if the failure was unexpected.

### Step 3 — Confirm application recovery

```bash
# Check ECS task health
aws ecs describe-services \
  --cluster velnor-dev \
  --services velnor-agent-gateway velnor-chat-api velnor-plane-api \
  --query 'services[*].{Name:serviceName,Running:runningCount,Desired:desiredCount,Status:status}'

# Check for recent application errors in CloudWatch Logs
aws logs filter-log-events \
  --log-group-name /velnor/dev/agent-gateway \
  --start-time $(date -v-10M +%s000) \
  --filter-pattern "ERROR" \
  --query 'events[*].{Time:timestamp,Message:message}' \
  --max-items 20
```

If ECS tasks are stuck in unhealthy state after 5 minutes:

```bash
# Force a rolling restart of affected services
for svc in velnor-agent-gateway velnor-chat-api velnor-plane-api; do
  aws ecs update-service \
    --cluster velnor-dev \
    --service $svc \
    --force-new-deployment
done
```

### Step 4 — Tenant isolation check post-failover

After Aurora failover, verify that per-tenant schema routing is intact.

```bash
# Connect to the new writer and confirm schemas are present
psql "host=velnor-dev-aurora.cluster-<ID>.us-east-1.rds.amazonaws.com \
      port=5432 dbname=velnor user=velnor_app sslmode=require" \
  -c "SELECT schema_name FROM information_schema.schemata
      WHERE schema_name LIKE 'velnor_t_%'
      ORDER BY schema_name;"
```

If schemas are missing or the new writer endpoint is returning errors, **do not resume tenant traffic** — escalate to the Founder and open an AWS Support case.

### Step 5 — Smoke check

```bash
# 1. Confirm writer endpoint accepts connections
psql "host=velnor-dev-aurora.cluster-<ID>.us-east-1.rds.amazonaws.com \
      port=5432 dbname=velnor user=velnor_app sslmode=require" \
  -c "SELECT current_database(), pg_is_in_recovery(), now();"

# 2. Write smoke test (confirms replication not accidentally still active)
psql "..." -c "CREATE TEMP TABLE _smoke_check (id int); DROP TABLE _smoke_check;"

# 3. Check CloudWatch alarm clears
# velnor-aurora-writer-unavailable should return to OK within 5 minutes
```

---

## Rollback criteria

- If promotion does not complete within 5 minutes: open AWS Support case (P1 for production, P2 for dev).
- If schemas are missing or corrupted post-failover: **halt all tenant traffic** immediately and page the Founder.
- If the new writer is in the same AZ as the original (single-AZ degraded mode): add a new replica in the second AZ via RDS console before resuming full traffic.

---

## Post-incident record

File within 48 hours of unplanned failover close.

```markdown
## Aurora Failover Incident — <date>

**Failover type:** planned | unplanned
**Failover start (detected):** <timestamp>
**Writer promotion complete:** <timestamp>
**Application fully recovered:** <timestamp>
**Total write outage window:** <duration>

### Former writer AZ: <AZ>
### New writer AZ: <AZ>

### Timeline

| Time | Event |
|------|-------|
| ...  | ...   |

### Root cause

<AWS-side AZ event | instance hardware fault | operator-triggered | other>

### Tenant impact

- Write operations unavailable for: <duration>
- Tenants affected: <count or "all">
- Data loss: none expected (Aurora Serverless v2 synchronous replication to reader)

### Action items

| Item | Owner | Due |
|------|-------|-----|
| Add multi-AZ replica if degraded after failover | Architect | 24h |
| Review connection-pool retry config if reconnect > 10s | Engineer | Next sprint |
```

---

## Contacts

| Role | Contact | Escalation order |
|------|---------|-----------------|
| Architect (primary) | See PagerDuty / Slack | 1st |
| Founder | See PagerDuty / Slack | 2nd (mandatory for unplanned SEV-1) |
| AWS Support | https://console.aws.amazon.com/support | 3rd (P1 for confirmed AZ-level event) |
