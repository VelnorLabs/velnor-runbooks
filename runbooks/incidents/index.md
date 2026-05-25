---
layout: default
title: "Incident Response Playbooks"
nav_order: 2
---

# Incident Response Playbooks

Structured playbooks for diagnosing and resolving production incidents.

**Owning personas:** Engineering, Architect (jointly — incident response requires both)

---

## Severity levels

| Severity | Definition | Target MTTR |
|----------|------------|-------------|
| SEV-1 | Full tenant outage or data integrity risk | < 1h |
| SEV-2 | Degraded service, single tenant affected | < 4h |
| SEV-3 | Non-critical degradation, workaround available | < 24h |

---

## Available playbooks

_Content seeded in T-W0-031. This index is a placeholder._

| Playbook | Severity | Status |
|----------|----------|--------|
| Bedrock outage / model unavailability | SEV-2 | Pending (T-W0-031) |
| Aurora failover | SEV-1 | Pending (T-W0-031) |

---

## Escalation contacts

Contacts and on-call rotation are managed in PagerDuty.  
Alert routing per DEC-0046: CloudWatch alarm → SNS topic → SES email → PagerDuty webhook.
