# velnor-runbooks

> Velnor V1 · Ops documentation · Published via GitHub Pages (Jekyll)

**Owning personas:** Engineering (primary), Architect (incident response co-owner)  
**Published at:** `https://velnorlabs.github.io/velnor-runbooks/`

---

## Purpose

`velnor-runbooks` is the single source of truth for Velnor V1 operational procedures,
incident response playbooks, and onboarding guides. All content is Markdown-first and
rendered via GitHub Pages (Jekyll/minima theme).

---

## Repository layout

```
velnor-runbooks/
├── _config.yml                     # Jekyll / GitHub Pages configuration
├── runbooks/
│   ├── operations/
│   │   └── index.md                # Operations runbooks index (content via T-W0-031)
│   ├── incidents/
│   │   └── index.md                # Incident response playbooks index
│   └── onboarding/
│       └── index.md                # Tenant + engineer onboarding guides
└── README.md
```

---

## Runbook categories

| Category | Description | Content status |
|----------|-------------|----------------|
| Operations | Routine procedures (deploys, scaling, cost controls) | Index seeded; content T-W0-031 |
| Incidents | SEV-1/2/3 response playbooks (Bedrock, Aurora) | Index seeded; content T-W0-031 |
| Onboarding | Tenant provisioning + engineer setup | Index seeded; content T-W0-031 |

---

## Status

- [`operations/tenant-deletion.md`](operations/tenant-deletion.md) — Tenant deletion runbook.
  **T-W3-021**: Architect draft, awaiting founder Architect sign-off note before `done_verified`.
  Gates **T-W3-022** (deletion endpoint) and **T-W3-023** (Dev tenant dry-run). Mirrored at
  [`runbooks/operations/tenant-deletion.md`](runbooks/operations/tenant-deletion.md) for the
  published Jekyll site.

---

## Contributing

1. All runbooks follow the standard template: **Purpose · Trigger · Steps · Rollback · Contacts**
2. Conventional Commits required; signed commits enforced in CI
3. PR requires review from `@VelnorLabs/engineering` (operations/onboarding)
   or both `@VelnorLabs/engineering` + `@VelnorLabs/architecture` (incidents)
4. Test against staging environment before merging incident playbooks

---

## GitHub Pages setup

1. Go to **Settings → Pages** in the GitHub repo
2. Set source to `main` branch, `/ (root)` folder
3. Site will be available at `https://velnorlabs.github.io/velnor-runbooks/`
