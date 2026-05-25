---
layout: default
title: "Operations Runbooks"
nav_order: 1
---

# Operations Runbooks

Routine operational procedures for the Velnor V1 platform.

**Owning personas:** Engineering, Architect

---

## Available runbooks

_Content seeded in T-W0-031. This index is a placeholder._

| Runbook | Description | Status |
|---------|-------------|--------|
| _(T-W0-031)_ | First 3 runbooks: Bedrock outage · Aurora failover · tenant onboarding | Pending |

---

## Conventions

- Each runbook follows the standard template: **Purpose · Trigger · Steps · Rollback · Contacts**
- All runbooks are tested against the staging environment before landing on `main`
- SLA targets are noted inline per DEC-0046 (CloudWatch → SNS → SES alert routing)
