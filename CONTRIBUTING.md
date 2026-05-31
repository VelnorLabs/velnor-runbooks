# Contributing to velnor-runbooks

Thank you for contributing to Velnor's operational runbooks.

## Who should contribute

- **Engineering team** — operations, onboarding, and general runbooks
- **Architecture team** — incident response playbooks (co-owns with Engineering)

## Runbook structure

Every runbook must follow this template:

```markdown
# <Title>

| Field | Value |
|-------|-------|
| Severity | SEV-1 / SEV-2 / SEV-3 / N/A |
| Owner | @VelnorLabs/engineering |
| Last reviewed | YYYY-MM-DD |

## Purpose

## Trigger conditions

## Steps

1. Step one
2. Step two

## Rollback

## Escalation contacts
```

## Commit conventions

- Use [Conventional Commits](https://www.conventionalcommits.org/)
  - `docs(runbooks):` for new/updated runbook content
  - `fix(runbooks):` for corrections
  - `feat(runbooks):` for new runbook categories
- Signed commits are enforced in CI (`git config commit.gpgsign true`)

## Pull request requirements

| Runbook type | Required reviewers |
|---|---|
| Operations | `@VelnorLabs/engineering` |
| Onboarding | `@VelnorLabs/engineering` |
| Incidents | `@VelnorLabs/engineering` + `@VelnorLabs/architecture` |

## Lint

Run markdownlint before opening a PR:

```bash
npx markdownlint-cli2 "**/*.md" --ignore node_modules
```

Or use pre-commit:

```bash
pre-commit run --all-files
```

## GitHub Pages

Content is published at `https://VelnorLabs.github.io/velnor-runbooks/` via Jekyll.
Preview locally with `bundle exec jekyll serve` (requires Ruby + Bundler).
