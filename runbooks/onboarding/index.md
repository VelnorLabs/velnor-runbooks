---
layout: default
title: "Onboarding Guides"
nav_order: 3
---

# Onboarding Guides

Step-by-step guides for onboarding new tenants and new engineers to the Velnor V1 platform.

**Owning personas:** Engineering, Architect

---

## Tenant onboarding

_Content seeded in T-W0-031. This index is a placeholder._

Tenant onboarding provisions:
- Schema namespace under `velnor_dev` (DEC-0035 rev. 2 — schema-per-tenant)
- Cognito user pool entry scoped to tenant_id
- S3 prefix allocation under the shared tenant-doc bucket (DEC-0047)
- Reserved slug registration in `velnor-iac/operations/reserved-slugs.md` (DEC-0042)

---

## Engineer onboarding

New engineers should:

1. Clone all repos listed in the `SMB-SaaS` top-level `README.md`
2. Run `pre-commit install` in each repo with a `.pre-commit-config.yaml`
3. Configure signed commits: `git config commit.gpgsign true`
4. Install `.tool-versions` dependencies via `asdf install`
5. Run `velnor-task dashboard` to see the current Wave status
