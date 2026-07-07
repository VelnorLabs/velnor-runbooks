# Graph Report - .  (2026-07-06)

## Corpus Check
- 22 files · ~25,000 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 71 nodes · 144 edges · 9 communities
- Extraction: 92% EXTRACTED · 6% INFERRED · 2% AMBIGUOUS · INFERRED: 8 edges (avg confidence: 0.69)
- Token cost: 0 input · 0 output

## Community Hubs (Navigation)
- [[_COMMUNITY_Atlas Migrations & DB Validation|Atlas Migrations & DB Validation]]
- [[_COMMUNITY_Bedrock Outage Response|Bedrock Outage Response]]
- [[_COMMUNITY_Tenant Isolation Auditing|Tenant Isolation Auditing]]
- [[_COMMUNITY_Repo CI & Contribution|Repo CI & Contribution]]
- [[_COMMUNITY_Tenant Deletion Procedure|Tenant Deletion Procedure]]
- [[_COMMUNITY_Aurora Cluster & Failover|Aurora Cluster & Failover]]
- [[_COMMUNITY_Jekyll Site & Indexes|Jekyll Site & Indexes]]
- [[_COMMUNITY_Deletion Governance & Sign-off|Deletion Governance & Sign-off]]
- [[_COMMUNITY_Deletion Authorization Gates|Deletion Authorization Gates]]

## God Nodes (most connected - your core abstractions)
1. `Tenant Deletion runbook (operations/, Architect draft)` - 23 edges
2. `Dev DB Validation runbook (bastion + Atlas)` - 14 edges
3. `Tenant Deletion runbook (published, runbooks/operations/)` - 14 edges
4. `Bedrock Outage runbook (published, runbooks/incidents/)` - 11 edges
5. `Tenant Isolation Alarm Response (incidents/)` - 10 edges
6. `Aurora Failover runbook (operations/)` - 10 edges
7. `Tenant Isolation Alarm Response (published, runbooks/incidents/)` - 10 edges
8. `Bedrock Outage / Model Unavailability (incidents/)` - 9 edges
9. `Atlas Migration Failure runbook` - 9 edges
10. `DEC-0035 rev.2 — schema-per-tenant` - 9 edges

## Surprising Connections (you probably didn't know these)
- `Aurora Failover runbook (operations/)` --semantically_similar_to--> `Tenant Isolation Alarm Response (incidents/)`  [INFERRED] [semantically similar]
  operations/aurora-failover.md → incidents/isolation-alarm-response.md
- `Step 0 — three independent authorization checks` --semantically_similar_to--> `Tenant Isolation Alarm Response (incidents/)`  [INFERRED] [semantically similar]
  operations/tenant-deletion.md → incidents/isolation-alarm-response.md
- `Security Policy` --conceptually_related_to--> `Bedrock Outage runbook (published, runbooks/incidents/)`  [INFERRED]
  SECURITY.md → runbooks/incidents/bedrock-outage.md
- `Security Policy` --conceptually_related_to--> `Tenant Isolation Alarm Response (published, runbooks/incidents/)`  [INFERRED]
  SECURITY.md → runbooks/incidents/isolation-alarm-response.md
- `Tenant Deletion runbook (published, runbooks/operations/)` --references--> `velnor-dev-uploads S3 bucket`  [EXTRACTED]
  runbooks/operations/tenant-deletion.md → operations/tenant-deletion.md

## Import Cycles
- None detected.

## Hyperedges (group relationships)
- **Three-layer tenant isolation pattern applied across breach response and deletion** — dec_0035, incidentsisolation_runbook, opstenantdeletion_runbook [INFERRED 0.75]
- **CloudWatch to SNS to SES alert routing pipeline (DEC-0046)** — dec_0046, sns_ops_alert, alarm_bedrock_error_rate, alarm_aurora_writer_unavailable [EXTRACTED 0.90]
- **Tenant deletion Step 0 authorization gate (three checks + safe_workflow)** — tenantdeletion_step0_three_checks, task_t_w3_022, dec_0039, tenantdeletion_idempotency_rationale [EXTRACTED 0.90]

## Communities (9 total, 0 thin omitted)

### Community 0 - "Atlas Migrations & DB Validation"
Cohesion: 0.17
Nodes (13): CloudWatch alarm: velnor-atlas-runner-error, DEC-0057 (referenced, unelaborated in this chunk), DEC-0061 — RDS Proxy replaces PgBouncer, atlas-runner ECS job, SSM bastion (accounts/dev/bastion), Atlas Migration Failure runbook, Rationale: stale PGPASSWORD causes false auth failures, Dev DB Validation runbook (bastion + Atlas) (+5 more)

### Community 1 - "Bedrock Outage Response"
Cohesion: 0.39
Nodes (9): CloudWatch alarm: velnor-bedrock-error-rate-high, DEC-0022 — AWS as cloud provider, DEC-0046 — CloudWatch to SNS to SES alert routing, DEC-0049 — AI provider fallback paths (Anthropic-direct / OpenAI), Bedrock Outage / Model Unavailability (incidents/), velnor-agent-gateway ECS service, Bedrock Outage runbook (published, runbooks/incidents/), SNS topic: velnor-ops-alert (+1 more)

### Community 2 - "Tenant Isolation Auditing"
Cohesion: 0.39
Nodes (8): CloudWatch alarm: velnor-tenant-isolation-breach, DEC-0051 (referenced, unelaborated in this chunk), Tenant Isolation Alarm Response (incidents/), Weekly isolation audit (VELNOR/IsolationAudit), Tenant Isolation Alarm Response (published, runbooks/incidents/), Security Policy, SNS topic: velnor-isolation-alert, Spike S-07 — isolation audit design

### Community 3 - "Repo CI & Contribution"
Cohesion: 0.36
Nodes (8): CONTRIBUTING guide, docs/index.md (Jekyll home mirror), CI GitHub Actions Workflow, Deploy GitHub Pages CI Job, Markdownlint CI Job, Repo root index.md, markdownlint configuration, pre-commit configuration

### Community 4 - "Tenant Deletion Procedure"
Cohesion: 0.25
Nodes (8): DEC-0047 — three-layer tenant document storage, velnor-dev-uploads S3 bucket, Tenant Deletion runbook (operations/, Architect draft), velnor-action-gate repo, velnor-admin-api repo, velnor-schemas repo, velnor-workers repo, Rationale: DR preservation window gives a definitive undo deadline

### Community 5 - "Aurora Cluster & Failover"
Cohesion: 0.48
Nodes (7): CloudWatch alarm: velnor-aurora-writer-unavailable, DEC-0023 — Kotlin/Spring Boot (HikariCP), DEC-0031/0032 — Aurora Postgres 16 Serverless v2, DEC-0035 rev.2 — schema-per-tenant, velnor-dev-aurora (Aurora Postgres Serverless v2 cluster), Aurora Failover runbook (operations/), Aurora Failover runbook (published, runbooks/operations/)

### Community 6 - "Jekyll Site & Indexes"
Cohesion: 0.38
Nodes (7): Jekyll / GitHub Pages config, DEC-0042 — wildcard subdomain / reserved slugs, DEC-0056 — GitHub org is VelnorLabs, Incident Response Playbooks index, Onboarding Guides index, Operations Runbooks index, T-W0-031 — runbook content seeding

### Community 7 - "Deletion Governance & Sign-off"
Cohesion: 0.47
Nodes (6): DEC-0050 — audit retention (90-day hot / 13-month Glacier), DEC-0065 — visibility/RLS (schema-scoped isolation), Tenant Deletion runbook (published, runbooks/operations/), velnor-runbooks README, T-W3-021 — tenant deletion runbook (Architect draft), T-W3-023 — tenant deletion dry-run

### Community 8 - "Deletion Authorization Gates"
Cohesion: 0.40
Nodes (5): DEC-0039 — safe_workflow / Step Functions Express, T-W3-022 — automated tenant deletion endpoint, Rationale: slug must be allow-list validated before DDL (injection hazard), Rationale: Idempotency-Key prevents duplicate destructive workflow invocation, Step 0 — three independent authorization checks

## Ambiguous Edges - Review These
- `Bedrock Outage / Model Unavailability (incidents/)` → `DEC-0051 (referenced, unelaborated in this chunk)`  [AMBIGUOUS]
  incidents/bedrock-outage.md · relation: references
- `Tenant Isolation Alarm Response (incidents/)` → `DEC-0051 (referenced, unelaborated in this chunk)`  [AMBIGUOUS]
  incidents/isolation-alarm-response.md · relation: references
- `Dev DB Validation runbook (bastion + Atlas)` → `DEC-0057 (referenced, unelaborated in this chunk)`  [AMBIGUOUS]
  runbooks/operations/dev-db-validation.md · relation: references

## Knowledge Gaps
- **18 isolated node(s):** `DEC-0042 — wildcard subdomain / reserved slugs`, `DEC-0049 — AI provider fallback paths (Anthropic-direct / OpenAI)`, `DEC-0056 — GitHub org is VelnorLabs`, `DEC-0057 (referenced, unelaborated in this chunk)`, `DEC-0061 — RDS Proxy replaces PgBouncer` (+13 more)
  These have ≤1 connection - possible missing edges or undocumented components.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **What is the exact relationship between `Bedrock Outage / Model Unavailability (incidents/)` and `DEC-0051 (referenced, unelaborated in this chunk)`?**
  _Edge tagged AMBIGUOUS (relation: references) - confidence is low._
- **What is the exact relationship between `Tenant Isolation Alarm Response (incidents/)` and `DEC-0051 (referenced, unelaborated in this chunk)`?**
  _Edge tagged AMBIGUOUS (relation: references) - confidence is low._
- **What is the exact relationship between `Dev DB Validation runbook (bastion + Atlas)` and `DEC-0057 (referenced, unelaborated in this chunk)`?**
  _Edge tagged AMBIGUOUS (relation: references) - confidence is low._
- **Why does `Tenant Deletion runbook (operations/, Architect draft)` connect `Tenant Deletion Procedure` to `Atlas Migrations & DB Validation`, `Bedrock Outage Response`, `Aurora Cluster & Failover`, `Deletion Governance & Sign-off`, `Deletion Authorization Gates`?**
  _High betweenness centrality (0.316) - this node is a cross-community bridge._
- **Why does `Dev DB Validation runbook (bastion + Atlas)` connect `Atlas Migrations & DB Validation` to `Tenant Deletion Procedure`, `Aurora Cluster & Failover`, `Jekyll Site & Indexes`?**
  _High betweenness centrality (0.231) - this node is a cross-community bridge._
- **Why does `DEC-0035 rev.2 — schema-per-tenant` connect `Aurora Cluster & Failover` to `Atlas Migrations & DB Validation`, `Tenant Isolation Auditing`, `Tenant Deletion Procedure`, `Jekyll Site & Indexes`, `Deletion Governance & Sign-off`?**
  _High betweenness centrality (0.167) - this node is a cross-community bridge._
- **Are the 2 inferred relationships involving `Tenant Isolation Alarm Response (incidents/)` (e.g. with `Aurora Failover runbook (operations/)` and `Step 0 — three independent authorization checks`) actually correct?**
  _`Tenant Isolation Alarm Response (incidents/)` has 2 INFERRED edges - model-reasoned connections that need verification._