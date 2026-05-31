---
layout: default
title: Velnor Runbooks
---

# Velnor Runbooks

> Operational runbooks, incident response playbooks, and onboarding guides for Velnor V1.

**Published by:** Velnor Engineering
**Source:** [github.com/VelnorLabs/velnor-runbooks](https://github.com/VelnorLabs/velnor-runbooks)

---

## Categories

### [Operations](../runbooks/operations/)

Routine procedures: deployments, scaling, cost controls, and maintenance tasks.

### [Incidents](../runbooks/incidents/)

SEV-1/2/3 response playbooks: Bedrock outages, Aurora failovers, tenant isolation alarms.

### [Onboarding](../runbooks/onboarding/)

Tenant provisioning guides and engineer setup procedures.

---

## Quick reference

| Severity | Response time | Runbook |
|----------|---------------|---------|
| SEV-1 | 15 min | [Aurora Failover](../runbooks/operations/aurora-failover.md) · [Isolation Alarm](../runbooks/incidents/isolation-alarm-response.md) |
| SEV-2 | 1 hour | [Bedrock Outage](../runbooks/incidents/bedrock-outage.md) |
| SEV-3 | 4 hours | [Atlas Migration Failure](../runbooks/incidents/atlas-migration-failure.md) |

---

## Contributing

See [CONTRIBUTING.md](https://github.com/VelnorLabs/velnor-runbooks/blob/main/CONTRIBUTING.md)
for runbook template, commit conventions, and PR requirements.
