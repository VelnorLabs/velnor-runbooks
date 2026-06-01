---
layout: default
title: "Tenant Isolation Alarm Response"
nav_order: 2
parent: "Incident Response Playbooks"
adr_refs:
  - DEC-0035
  - DEC-0046
  - DEC-0051
severity: SEV-1
last_updated: 2026-05-24
---

# Tenant Isolation Alarm Response

> **Severity:** SEV-1 — potential cross-tenant data exposure; treat as a data integrity incident until proven otherwise  
> **Who is paged:** Architect (primary), Founder (mandatory co-responder)  
> **Expected duration:** 30–90 min (triage to containment); root-cause investigation may extend 24h  
> **Escalation chain:** Architect → Founder → Legal / Data Protection Officer (if confirmed exposure)

---

## Purpose

This runbook covers the response when the `VELNOR/IsolationAudit` CloudWatch alarm fires,
indicating that the weekly isolation audit detected at least one cross-tenant write in
`tool_invocation_log`. A breach means a tool was invoked in the wrong tenant schema
context — the structural isolation layer (per-tenant schema + session credential + RLS)
may have a gap.

Reference: Agent Runtime rev. 7 §4.3 · Spike S-07 isolation audit design.

---

## Triggering alarm

| Field | Value |
|-------|-------|
| **CloudWatch alarm name** | `velnor-tenant-isolation-breach` |
| **Namespace / metric** | `VELNOR/IsolationAudit` / `tenant_isolation_breach_count` |
| **Threshold** | breach_count >= 1 |
| **Period** | 60 seconds · EvaluationPeriods = 1 |
| **TreatMissingData** | `notBreaching` (alarm stays OK when audit hasn't run) |
| **SNS topic** | `arn:aws:sns:us-east-1:<ACCOUNT_ID>:velnor-isolation-alert` |
| **Alert delivery** | SNS → Lambda → SES email to Architect (DEC-0046) |
| **Dashboard** | `velnor-dev-isolation-audit` — alarm state, breach count, last run timestamp |

---

## Isolation model (context)

Velnor uses **three-layer isolation** (Agent Runtime rev. 7 §4.3, DEC-0035 rev. 2):

1. **Request-time tenant header validation** — every API request carries `X-Tenant-ID`; validated at the gateway before any DB call.
2. **Connection-pool tenant scoping** — each DB connection sets `SET search_path = velnor_t_{slug}` before executing queries.
3. **Row-level deny-by-default** — `tool_invocation_log.tenant_id` must equal `replace(current_schema(), 'velnor_t_', '')`.

A breach in `tool_invocation_log` means a row was written where `tenant_id` does not match the schema it was written into — one of these three layers failed.

---

## Step 1 — Confirm the alarm (< 2 min)

1. Open CloudWatch → Alarms → `velnor-tenant-isolation-breach` → confirm state is **ALARM**.
2. Note the breach count from the metric graph (how many breaches, over what time window?).
3. Open the `velnor-dev-isolation-audit` dashboard — confirm breach count > 0 and note the `last_run_timestamp`.

> **Do not dismiss the alarm.** Even a breach_count = 1 is a SEV-1 until root cause is established.

---

## Step 2 — Page the Founder (< 3 min)

This step is mandatory. Tenant isolation is a data integrity invariant. The Founder must be
informed before any investigation conclusions are drawn.

Send to ops channel:

```
[SEV-1 ACTIVE] Tenant isolation breach detected.
CloudWatch alarm: velnor-tenant-isolation-breach
Breach count: <N>
Audit run: <timestamp>
Architect and Founder both required. Investigation in progress.
```

---

## Step 3 — Run the forensic detail view query (< 10 min)

The isolation audit runner emits only an aggregate count. To identify *which* tenant is
leaking and *which* tool is involved, run the detail view query directly.

### 3a — Connect to Aurora (writer endpoint)

```bash
psql "host=velnor-dev-aurora.cluster-<ID>.us-east-1.rds.amazonaws.com \
      port=5432 dbname=velnor user=velnor_app sslmode=require"
```

### 3b — Run per-schema forensic detail query

For each `velnor_t_*` schema, run the detail view with the schema context set:

```sql
-- Run this for each suspect schema. Replace 'velnor_t_<slug>' with the schema name.
SET search_path = velnor_t_<slug>;

-- Forensic detail view: which tenants are leaking into this schema?
SELECT
    current_schema()                                AS schema_name,
    tenant_id                                       AS logged_tenant,
    replace(current_schema(), 'velnor_t_', '')      AS expected_tenant,
    actor_id,
    tool_name,
    MIN(invoked_at)                                 AS first_breach,
    MAX(invoked_at)                                 AS last_breach,
    COUNT(*)                                        AS breach_row_count
FROM tool_invocation_log
WHERE tenant_id != replace(current_schema(), 'velnor_t_', '')
GROUP BY tenant_id, actor_id, tool_name
ORDER BY breach_row_count DESC;
```

If you do not know which schema to check, run a cross-schema sweep:

```sql
-- Find all schemas with breaches (run as a superuser / velnor_admin role)
SELECT schema_name
FROM information_schema.schemata
WHERE schema_name LIKE 'velnor_t_%';
-- Then run the detail view above for each returned schema.
```

### 3c — Record the findings

Fill in the incident record template (see Step 7) with:
- `schema_name` (the victim schema — the one that received alien rows)
- `logged_tenant` (the alien tenant whose tenant_id appears in the wrong schema)
- `tool_name` (which tool wrote the row)
- `actor_id` (which user/agent triggered the tool call)
- `first_breach` / `last_breach` timestamps

---

## Step 4 — Identify the breaching tenant

Using the detail view output:

- **`schema_name`** = the tenant whose schema was contaminated (victim tenant)
- **`logged_tenant`** = the tenant whose `tenant_id` was incorrectly written (source tenant)

Determine when those two tenants had overlapping activity:

```bash
# Pull agent-gateway logs for the source tenant around the breach window
aws logs filter-log-events \
  --log-group-name /velnor/dev/agent-gateway \
  --start-time <first_breach_epoch_ms - 300000> \
  --end-time <last_breach_epoch_ms + 300000> \
  --filter-pattern '{ $.tenant_id = "<logged_tenant>" }' \
  --query 'events[*].{Time:timestamp,Message:message}' \
  --max-items 50
```

Look for:
- Requests where `X-Tenant-ID` header did not match the `search_path` that was set.
- Concurrent requests from two tenants that may have shared a connection-pool connection.
- Any recent deployment that changed the connection-pool scoping logic.

---

## Step 5 — Containment decision

### If breach is isolated to a single tool call (single row, clear clock-skew root cause):

- **Minimal containment:** delete the offending row from the victim schema, document the root cause, and schedule a code fix.
- No tenant traffic halt required if the breach is demonstrably not ongoing.

```sql
-- Delete the confirmed breach row(s) ONLY after documented founder approval
SET search_path = velnor_t_<victim_slug>;
DELETE FROM tool_invocation_log
WHERE tenant_id = '<logged_tenant>'
  AND actor_id = '<actor_id>'
  AND tool_name = '<tool_name>'
  AND invoked_at BETWEEN '<first_breach>' AND '<last_breach>';
```

### If breach count > 1 or involves data reads (not just audit log):

- **Full containment:** halt the affected tenant's API traffic immediately.

```bash
# Set the breaching tenant to maintenance mode (blocks inbound requests at gateway)
aws ssm put-parameter \
  --name /velnor/dev/tenants/<logged_tenant>/maintenance_mode \
  --value "true" \
  --overwrite
```

- Do not resume tenant traffic until root cause is confirmed and a fix is deployed.

### If breach involves entity data tables (not just tool_invocation_log):

- **This is a data exposure event.** Escalate to Legal / Data Protection Officer immediately.
- Preserve all logs — do not delete any data without legal review.
- Notify the affected tenant per the incident response SLA agreed in their contract.

---

## Step 6 — Root cause investigation

Common root causes ranked by likelihood (from S-07 spike findings):

| Rank | Root cause | Indicator |
|------|-----------|-----------|
| 1 | `search_path` not set before tool execution | `actor_id` has concurrent sessions across two tenants |
| 2 | Connection-pool connection returned to wrong tenant context | `tool_name` involves a retry path that doesn't reset `search_path` |
| 3 | Decorator bypass — `@tenant_scoped` decorator missing on a new tool | `tool_name` is recently added (check git blame) |
| 4 | RLS policy gap — `tenant_id` column not in policy predicate | Affects multiple schemas, same tool |

To confirm root cause 1 or 2, look for overlapping timestamps in the X-Ray traces:

```bash
# Query X-Ray for agent-gateway traces around the breach window
aws xray get-trace-summaries \
  --start-time <first_breach_epoch> \
  --end-time <last_breach_epoch> \
  --filter-expression 'service("velnor-agent-gateway") AND annotation.tenant_id = "<logged_tenant>"' \
  --query 'TraceSummaries[*].{TraceID:Id,Duration:Duration,StartTime:EntryPoint.StartTime}'
```

---

## Step 7 — File the incident

Create a new file in `velnor-runbooks/runbooks/incidents/archive/` named
`YYYY-MM-DD-isolation-breach-<short-description>.md` with this template:

```markdown
## Isolation Breach Incident — <date>

**Alarm fired at:** <timestamp>
**Founder notified at:** <timestamp>
**Containment complete at:** <timestamp>
**Root cause confirmed at:** <timestamp>

### Breach details

| Field | Value |
|-------|-------|
| Victim schema | velnor_t_<slug> |
| Source tenant (alien tenant_id) | <logged_tenant> |
| Tool involved | <tool_name> |
| Actor | <actor_id> |
| First breach | <first_breach> |
| Last breach | <last_breach> |
| Breach row count | <N> |
| Data tables affected beyond tool_invocation_log | yes / no |

### Root cause

<Describe: which of the three isolation layers failed and why>

### Containment actions taken

<List actions: row deletion, tenant maintenance mode, etc.>

### Code fix

<PR link or commit SHA>

### Follow-up action items

| Item | Owner | Due |
|------|-------|-----|
| ... | ... | ... |
```

---

## Step 8 — Resolve the alarm

Once the breach rows are addressed and the root cause fix is deployed:

1. Re-run the isolation audit manually to confirm breach_count = 0:

```bash
# Trigger the CodeBuild audit job manually
aws codebuild start-build \
  --project-name velnor-isolation-audit \
  --region us-east-1
```

2. Confirm `VELNOR/IsolationAudit / tenant_isolation_breach_count` metric = 0.
3. Confirm `velnor-tenant-isolation-breach` alarm returns to **OK**.
4. Lift maintenance mode if applied:

```bash
aws ssm put-parameter \
  --name /velnor/dev/tenants/<logged_tenant>/maintenance_mode \
  --value "false" \
  --overwrite
```

---

## Rollback criteria

- If you cannot determine root cause within 2 hours: put the breaching tenant in permanent maintenance mode and escalate to a dedicated engineering investigation. Do not declare incident closed.
- If data tables beyond `tool_invocation_log` are affected: do not delete any data; escalate to Legal immediately.

---

## Contacts

| Role | Contact | Escalation order |
|------|---------|-----------------|
| Architect (primary) | See PagerDuty / Slack | 1st |
| Founder (mandatory co-responder) | See PagerDuty / Slack | 2nd |
| Legal / DPO | See internal contacts list | If confirmed data exposure |
| AWS Support | https://console.aws.amazon.com/support | If platform-level isolation failure suspected |
