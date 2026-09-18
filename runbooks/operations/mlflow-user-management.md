---
layout: default
title: "MLflow User Management (agent trace store)"
nav_order: 5
parent: "Operations Runbooks"
adr_refs:
  - DEC-0050
  - LE-013
severity: maintenance
last_updated: 2026-09-18
---

# MLflow User Management — agent trace store

| Field | Value |
|-------|-------|
| Severity | N/A (maintenance) |
| Owner | @VelnorLabs/engineering |
| Last reviewed | 2026-09-18 |

## Purpose

Create, promote, and remove accounts on the self-hosted MLflow trace store in
dev. MLflow runs with `--app-name basic-auth`, so **every route authenticates
the caller** against its own user database (`mlflow_auth` in Postgres) — this is
not AWS IAM, and an AWS login grants nothing here.

The service has no public path. It is reached only through the SSM bastion, so
every step below assumes an open tunnel.

## Trigger conditions

- A team member needs to view agent traces and has no MLflow account
- Someone leaves and their account must be removed
- The bootstrap admin password needs rotating

---

## Step 0: Open the tunnel

MLflow listens on **8200** in the VPC and forwards to **localhost:5050**.

> **Not 5000.** macOS AirPlay Receiver binds `*:5000` on every Mac and answers
> `403 Forbidden, Server: AirTunes`, so a 5000 forward silently lands on AirPlay.
> The server port was moved off 5000 for the same reason. Override the local
> port with `MLFLOW_LOCAL_PORT` if 5050 is taken.

```bash
cd velnor-iac
./scripts/bastion.sh up            # apply + wait for SSM (skip if already up)
./scripts/bastion.sh tunnel mlflow # 8200 -> localhost:5050; leave running
```

> **The bastion is shared.** A second `up` is harmless — it re-applies the same
> workspace. But `down` destroys the instance and **every** tunnel through it,
> including other people's. Agree who tears it down, or leave it up (~$3/month).

If the tunnel fails, the script names the cause: either MLflow's security group
is absent, or the bastion has not been applied since MLflow appeared. It does
not fail silently.

## Step 1: Authenticate as the bootstrap admin

The `velnor-admin` account is created on first boot from Secrets Manager. It is
a **bootstrap credential, not a personal login**.

```bash
MLFLOW=http://localhost:5050
ADMIN_PW=$(aws secretsmanager get-secret-value \
  --secret-id velnor/dev/mlflow/admin-password \
  --query SecretString --output text --profile velnor-dev --region us-east-1)

# Sanity-check both variables without printing the password
echo "MLFLOW=$MLFLOW  pw_length=${#ADMIN_PW}"
```

> Set both in the **same shell** you run the calls from. Environment variables
> do not cross terminals, and `curl -s` suppresses its own errors — an unset
> `$MLFLOW` produces *no output at all* (`curl: (3) URL rejected: No host part
> in the URL`), which reads as a hang rather than a mistake. Use `-sS` so errors
> stay visible.

Verify before making changes:

```bash
curl -sS -u "velnor-admin:$ADMIN_PW" \
  "$MLFLOW/api/2.0/mlflow/users/get?username=velnor-admin"
```

Expect `{"user":{"id":1,"is_admin":true,"username":"velnor-admin"}}`.

> Requires `secretsmanager:GetSecretValue` on that secret plus `kms:Decrypt` on
> the CMK encrypting it. The `velnor-dev` profile has both. **Never paste the
> password into a file or this runbook.**

## Step 2: Create accounts

One account per person. Passwords are generated locally and never committed.

```bash
umask 077; OUT=~/mlflow-creds.txt; : > "$OUT"
for u in <username1> <username2>; do
  PW=$(openssl rand -base64 18)
  curl -sS -u "velnor-admin:$ADMIN_PW" -X POST "$MLFLOW/api/2.0/mlflow/users/create" \
    -H 'Content-Type: application/json' \
    -d "{\"username\":\"$u\",\"password\":\"$PW\"}" -w " <- create $u [%{http_code}]\n"
  printf '%s\t%s\n' "$u" "$PW" >> "$OUT"
done
```

Distribute each line to its owner over a private channel, then **delete the
file**: `rm ~/mlflow-creds.txt`. Each person should change their password
immediately (Step 5).

## Step 3: Grant access — this step is not optional

A newly created user can authenticate and will then see **nothing**. The auth
config sets:

```ini
default_permission = NO_PERMISSIONS
grant_default_workspace_access = false
```

That is deliberate least privilege — *nobody sees an experiment until granted
access*. Without this step the UI looks identical to the "no traces yet" state,
and the two are indistinguishable to the person hitting the blank page.

Two ways to grant it:

**A — make them an admin** (simplest; admins bypass per-experiment checks, so
nothing needs repeating when new experiments appear):

```bash
curl -sS -u "velnor-admin:$ADMIN_PW" -X PATCH "$MLFLOW/api/2.0/mlflow/users/update-admin" \
  -H 'Content-Type: application/json' \
  -d '{"username":"<username>","is_admin":true}' -w " [%{http_code}]\n"
```

**B — grant a scoped permission** (least privilege; repeat per resource):

Note the **`/api/3.0/`** prefix — these are v3 routes, unlike the user CRUD
endpoints above which are v2. MLflow 3.x replaced the older
`experiments/permissions/*` endpoints with a unified per-user grant API; those
older paths do **not** exist in this version and will 404.

```bash
curl -sS -u "velnor-admin:$ADMIN_PW" -X POST \
  "$MLFLOW/api/3.0/mlflow/users/permissions/grant" \
  -H 'Content-Type: application/json' \
  -d '{"username":"<username>","permission":"EDIT"}' \
  -w " [%{http_code}]\n"
```

Permission values are `READ`, `EDIT`, `MANAGE`, `NO_PERMISSIONS`.

Check what a user's effective permission actually resolves to — this evaluates
it the same way a real request would, rather than reporting what was granted:

```bash
curl -sS -u "velnor-admin:$ADMIN_PW" \
  "$MLFLOW/api/3.0/mlflow/users/permissions/get?username=<username>"
```

> Grants write to the user's synthetic `__user_<id>__` role in the active
> workspace. MLflow 3.x also ships a full RBAC role system
> (`/api/3.0/mlflow/roles/create`, `roles/permissions/add`, `assign-role`) —
> worth preferring over blanket admin once more than a handful of people need
> differentiated access.
>
> Choose deliberately. Option A is appropriate for a small team on an internal
> dev tool and is what dev uses today, but it does step away from the
> least-privilege posture the config sets. Option B preserves it at the cost of
> a grant per user per resource.

## Step 4: Verify

```bash
for u in <username1> <username2>; do
  curl -sS -u "velnor-admin:$ADMIN_PW" "$MLFLOW/api/2.0/mlflow/users/get?username=$u"; echo
done
```

Each line should show the expected `is_admin` value. A `404` means the create
did not land.

List every account, which is the reliable way to spot one nobody is tracking.
User ids are sequential, so a gap may mean a deleted account or simply a
rolled-back insert — the list tells you which:

```bash
curl -sS -u "velnor-admin:$ADMIN_PW" "$MLFLOW/api/2.0/mlflow/users/list"
```

> Do **not** try to infer whether a route or an account exists from an HTTP
> status code here. MLflow's auth middleware rejects before routing, so every
> path — real, mistyped, or invented — returns `401` when unauthenticated. Only
> an authenticated call distinguishes them.

## Step 5: Password changes

Each person, on first login:

```bash
curl -sS -u "<username>:<their-current-password>" -X PATCH \
  "$MLFLOW/api/2.0/mlflow/users/update-password" \
  -H 'Content-Type: application/json' \
  -d '{"username":"<username>","password":"<new-password>"}' -w " [%{http_code}]\n"
```

**Rotating `velnor-admin`:** change it in MLflow *and* update the Secrets
Manager value in the same maintenance window. The entrypoint reads
`MLFLOW_ADMIN_PASSWORD` at boot, so if the two diverge the next task replacement
recreates the admin from the stale secret.

## Step 6: Remove an account

```bash
curl -sS -u "velnor-admin:$ADMIN_PW" -X DELETE "$MLFLOW/api/2.0/mlflow/users/delete" \
  -H 'Content-Type: application/json' \
  -d '{"username":"<username>"}' -w " [%{http_code}]\n"
```

Do this when someone leaves. An AWS account being disabled does **not** disable
their MLflow login — the two identity systems are unrelated.

---

## Rollback

| Action | Rollback |
|---|---|
| Created a user in error | Step 6 (delete) |
| Granted admin in error | `update-admin` with `"is_admin":false` |
| Granted a permission in error | `DELETE /api/3.0/mlflow/users/permissions/revoke` with the same `username` |
| Changed a password and lost it | Reset it as `velnor-admin` via Step 5 |
| Lost the `velnor-admin` password | Fetch it again from Secrets Manager (Step 1); it is not derived from anything local |

No action here touches trace data. Deleting a user removes the login, not the
traces they recorded.

## Escalation contacts

- **Owner:** `@VelnorLabs/engineering`
- **Infrastructure:** `velnor-iac/accounts/dev/ecs/mlflow` (service),
  `accounts/dev/mlflow` (database, bucket, secrets),
  `accounts/dev/bastion` (tunnel)
- **Bastion reference:** `velnor-iac/accounts/dev/bastion/README.md`

## Notes

- **Usernames do not have to match AWS IAM users, but consider making them
  match.** Dev currently uses generic accounts. MLflow's audit trail records
  whatever username acted, so a generic name means you cannot tell who deleted a
  trace without a side mapping.
- **An empty UI is expected until instrumentation lands.** Nothing is wired to
  send traces yet; the store is deployed ahead of its producers. A blank
  experiment list is not a fault, and is also what Step 3 prevents being
  mistaken for a permissions problem.
- `/health` is unauthenticated by design so the container health check needs no
  credentials. Every other route requires authentication.
