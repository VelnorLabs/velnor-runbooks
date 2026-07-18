---
layout: default
title: "Tenant Deletion — Archive Semantics (Founder-Approved)"
nav_order: 4
parent: "Operations Runbooks"
adr_refs:
  - DEC-0035
  - DEC-0047
  - DEC-0065
  - DEC-0050
  - DEC-0046
severity: destructive-lite / reversible while archived; hard purge (future) is irreversible
last_updated: 2026-07-18
status: "ARCHIVE SEMANTICS — founder-approved 2026-07-18 (W3 design); dry-run T-W3-023 pending"
---

# Tenant Deletion (Archive + Tag)

> **Status:** Founder-approved archive semantics (2026-07-18 W3 design session). This revision
> **supersedes the T-W3-021 DROP-CASCADE draft**: tenant "deletion" is now an **archive + tag**
> operation — data is retained, renamed out of the live path, and every side effect is
> reversible. A true hard purge (dump → verify → drop) is a **future, separate procedure**
> (see "Future work — hard purge", below) and is explicitly **NOT implemented**.
>
> **Severity:** Destructive-lite. The tenant loses live access immediately, but no data is
> destroyed: the schema is renamed in place, Cognito users are disabled (not deleted), and S3
> objects are tagged (not removed). Rollback is a bounded, documented procedure (see Rollback
> semantics).
> **Who performs this:** A Velnor admin, deliberately, via
> `DELETE /api/v1/admin/tenants/{slug}` (T-W3-022) — a **synchronous** endpoint, no Step
> Function, not automated by any scheduler. Until T-W3-022 is deployed, the founder/Architect
> may perform the manual-run equivalent below by hand.
> **Expected duration:** ~5–10 min of active work. There is no DR countdown window — archived
> data is retained until a future hard-purge decision is made and executed separately.
> **Escalation chain:** Architect → Founder → AWS Support (only for AWS-side API failures)

---

## Purpose

Archives a Velnor tenant end-to-end: marks the registry row `archived`, renames the tenant's
Postgres schema out of the live namespace (DEC-0035 rev. 2), disables the tenant's Cognito
users (reversible), tags the tenant's S3 document originals `tenant_archived=true` (DEC-0047),
and records a `tenant_archived` confirmation in the shared `velnor.audit_event` ledger.

This procedure is the source of truth that **T-W3-022** (`DELETE
/api/v1/admin/tenants/{slug}` + `POST /api/v1/admin/tenants/{slug}/deletion-preview`)
automates behind the authorization guards in Step 0, and that **T-W3-023** dry-runs against a
throwaway Dev tenant before either can move to `done_verified`.

Founder decisions locked in the 2026-07-18 W3 design:

1. Deletion is an **archive + tag** operation, performed deliberately by a Velnor admin — not
   automated, no Step Function; the endpoint is synchronous.
2. Schema archive = **rename in place**: `velnor_t_<slug>` → `velnor_arch_<slug>_<yyyymmdd>`
   (slug normalized `-`→`_` per SDK convention). Data retained.
3. Users archive = **Cognito `AdminDisableUser`** (reversible), not delete.
4. MFA freshness = **JWT `auth_time` ≤ 5 min** (the admin pool has MFA=ON, so `auth_time` IS
   the MFA timestamp). No step-up subsystem.
5. Audit event type = **`tenant_archived`**.
6. Idempotency is **state-derived** from `velnor.tenants.state` — no dedupe ledger table.

Anchors: DEC-0035 rev. 2 (schema-per-tenant), DEC-0047 (three-layer tenant document storage),
DEC-0065 (visibility/RLS — tenant isolation is schema-scoped inside the tenant schema),
DEC-0050 (audit retention: 90-day hot / 13-month Glacier window), DEC-0046 (CloudWatch → SNS →
SES alert routing, for the escalation paths below).

---

## Scope

- **In scope:** the tenant's schema (`velnor_t_{slug_normalized}`) — renamed, not dropped; the
  tenant's Cognito users in the tenant pool (`custom:tenant_id == <slug>`) — disabled, not
  deleted; the tenant's S3 objects in the uploads bucket — tagged `tenant_archived=true`, not
  removed; the tenant's row in the shared registry (`velnor.tenants`) — `state` set to
  `archived`; the tenant's pending `action_request` rows — declined before archive; the
  tenant's Stripe subscription, if any — cancelled before archive.
- **Out of scope:** the shared `velnor` schema itself and its other tenants' rows;
  `velnor.golden_template`; Glacier-archived `tool_invocation_log` history for this tenant
  (retained for the full 13-month DEC-0050 audit window regardless of archival — see Edge
  cases); **any data destruction whatsoever** — dump/verify/drop is the future hard-purge
  procedure, not this one.

---

## Step 0 — Authorization guards (T-W3-022 evaluates these in this exact order)

**T-W3-022 implements all of the following in the endpoint before any archive action runs.**
They are independent — none may substitute for another — and failing any one aborts before
any state changes.

1. **Admin JWT** — via the existing admin-auth middleware
   (`velnor-admin-api/internal/middleware/admin.go`). Absent/invalid → `401` (existing
   behavior).
2. **Idempotency-Key** — the request must carry an `Idempotency-Key` header, else `400`
   `{"error":"missing Idempotency-Key header"}`. The submitted key is recorded in the audit
   payload for traceability. Replay protection itself is **state-derived**, not key-derived —
   see "Idempotency" below.
3. **confirm-slug** — the request body must include `{"confirm_slug": "<slug>"}` and it must
   equal the `{slug}` path parameter **exactly, case-sensitive**, else `422`. Purpose: stops a
   wrong-tenant archive caused by a UI misclick or a copy-pasted slug from a different tab.
4. **MFA freshness** — the JWT `auth_time` claim must be within **300 s** of now, else `401`
   `{"error":"MFA freshness expired; re-authenticate"}`. The admin pool enforces MFA=ON, so a
   fresh `auth_time` proves a fresh MFA challenge — no separate step-up subsystem exists or is
   needed. The middleware parses `auth_time` into the request context; the handler enforces
   the window so only destructive routes pay it.

### Idempotency (state-derived)

If `velnor.tenants.state = 'archived'` already, the endpoint returns **200** with the recorded
archive result, read back from the latest `tenant_archived` audit_event payload for that slug
— replay-safe within and beyond 24 h, with no dedupe ledger table. Concurrent
double-invocation is closed by the `UPDATE ... SET state='archived'` check-and-set being the
**first** execution step: the second request observes `archived` and takes the replay path.

**Manual-run equivalent (this runbook, before T-W3-022 ships):** the operator performs the
guards by hand — re-type the slug at the Step 1 confirmation prompt (do not copy-paste it from
a ticket, to force a second read), check `velnor.tenants.state` first and stop if it is
already `archived`, and confirm an MFA-backed `velnor-dev` SSO login happened within the last
5 minutes before Step 4.

### Implementation hazards for T-W3-022 (security review)

- **`slug` must be allow-list validated before it ever reaches DDL.** Step 4's
  `ALTER SCHEMA velnor_t_<slug_normalized> RENAME TO ...` is a schema-qualified identifier —
  Postgres has no parameterized-identifier form for DDL, so the endpoint must validate `slug`
  against the provisioning charset (`^[a-z][a-z0-9-]{1,40}$`, reserved-slug deny-list — reuse
  the provisioning validator in `internal/handlers/tenants.go`) *before* Step 0's checks run,
  and build the DDL via a quoted-identifier helper (`pgx.Identifier{...}.Sanitize()` or
  equivalent), never raw string interpolation. confirm-slug only proves body==path — it does
  not prove either is a safe identifier.
- **Schema ownership (accepted risk).** `ALTER SCHEMA ... RENAME` requires ownership of the
  schema. If the admin-api DB role lacks it in Dev, the T-W3-023 dry-run will surface the
  failure; the fallback is a founder-run grant via the bastion tunnel:
  `ALTER SCHEMA velnor_t_<slug_normalized> OWNER TO <admin_api_role>;` — then re-issue the
  archive request (the state-derived idempotency makes the retry safe).
- **Partial failure is reported, not hidden.** Cognito disables and S3 tagging continue on
  per-item failure, collect the failures, and report them in both the response and the
  `partial_failures` array of the audit payload. An operator must re-drive only the failed
  items (both operations are idempotent) — never re-run the whole procedure blindly.

---

## Pre-flight checklist

All boxes must be checked before Step 1. If any is unchecked, stop.

- [ ] **Consent obtained** — written confirmation from the tenant (or the Founder, for a
      ToS/compliance-driven removal) that archival is authorized. Attach to the task ticket.
      Note: because this is an archive, consent to *archive* is NOT consent to *purge* —
      a future hard purge needs its own consent record.
- [ ] **Backups verified** — the archive itself destroys nothing, but confirm Aurora's
      automated backup is current anyway (it is the recovery path if anything unexpected
      happens mid-procedure):

  ```bash
  aws rds describe-db-clusters --region us-east-1 --profile velnor-dev \
    --db-cluster-identifier velnor-dev \
    --query 'DBClusters[0].{Retention:BackupRetentionPeriod,LatestRestorableTime:LatestRestorableTime}'
  # expect Retention=7 (Dev, per velnor-iac/accounts/dev/aurora/main.tf) and
  # LatestRestorableTime within the last few minutes.
  ```

- [ ] **State checked** — `SELECT state FROM velnor.tenants WHERE slug='<slug>';` — if already
      `archived`, stop: the archive has already happened (state-derived idempotency; see
      Step 0). Re-running the manual procedure against an archived tenant is a no-op at best
      and a double-rename hazard at worst.
- [ ] **Deletion preview captured** — run the preview (or its manual SQL equivalent, Step 1)
      and attach the row counts / user count / object count to the ticket. This is the
      before-picture the post-archive verification compares against.
- [ ] **Subscription status checked** — see Step 2 / Edge cases: active subscription.
- [ ] **Pending action_requests checked** — see Step 3 / Edge cases: pending action_requests.

---

## Ordered procedure

Open the SSM bastion tunnel first — see `runbooks/operations/dev-db-validation.md` §0 for the
full walkthrough (fetching the master password, `127.0.0.1` vs `localhost`, etc.):

```bash
cd velnor-iac
./scripts/bastion.sh up
./scripts/bastion.sh tunnel      # Aurora-direct 5432 -> localhost:5432
```

Capture the tenant's identifiers up front — you need both (`tenant_id` keys the S3 prefix and
the audit payload; the schema name is derived from `slug` with `-`→`_`):

```sql
SELECT tenant_id, slug, display_name, plan, state
FROM velnor.tenants
WHERE slug = '<slug>';
-- record tenant_id (UUID) and confirm state is NOT already 'archived'.
```

### Step 1 — Authorization + pre-flight + preview

Complete Step 0's guards and the full pre-flight checklist above. Capture the deletion
preview — via the endpoint once T-W3-022 is deployed:

```bash
curl -s -X POST "$ADMIN_API_BASE_URL/api/v1/admin/tenants/<slug>/deletion-preview" \
  -H "Authorization: Bearer $ADMIN_JWT"
# 200 {slug, row_counts:{<table>:n,...}, s3_object_count, user_count}
# 404 if tenant unknown; 409 if already archived.
```

or manually (per-table counts for the tenant schema):

```sql
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'velnor_t_<slug_normalized>';
-- then, for each: SELECT count(*) FROM velnor_t_<slug_normalized>.<table>;
```

**Abort point:** nothing has changed yet; stopping here is a no-op.

### Step 2 — Cancel the Stripe subscription (if active)

> **Known gap (unchanged from the T-W3-021 draft):** `velnor.tenants` has no
> `stripe_subscription_id` column and the only subscription store is an in-memory stub
> (`velnor-workers/internal/stripe/handlers/subscription.go`). The Stripe webhook handler keys
> subscriptions by `metadata["tenant_id"]`, so that is the only reliable lookup path today.

1. In the Stripe Dashboard, search Customers/Subscriptions by metadata
   `tenant_id = <tenant_id from above>`.
2. If an active subscription exists, cancel it immediately — an archived tenant must not
   continue to bill.
3. Confirm the cancellation webhook landed (event lands in the shared ledger keyed by
   `tenant_id`):

   ```sql
   SELECT event_type, created_at FROM velnor.audit_event
   WHERE tenant_id = '<tenant_id>' AND event_type = 'subscription_cancelled'
   ORDER BY created_at DESC LIMIT 1;
   -- expect one row, created within the last few minutes
   ```

**Abort point:** if no active subscription is found, or cancellation is confirmed, continue.
If cancellation fails or the webhook doesn't land within a few minutes, stop and escalate to
the Architect — do not archive a tenant whose subscription is still live.

### Step 3 — Decline pending action_requests

```sql
-- inspect what's pending in the tenant's own schema
SELECT id, state, payload FROM velnor_t_<slug_normalized>.action_request
WHERE state IN ('proposed', 'pending')
ORDER BY id;

-- decline them all; record the reason inside payload (no dedicated reason column —
-- velnor-action-gate/migrations/001_action_request.sql)
UPDATE velnor_t_<slug_normalized>.action_request
SET state = 'declined',
    payload = payload || jsonb_build_object('decline_reason', 'tenant_archived', 'declined_at', now())
WHERE state IN ('proposed', 'pending');
```

**Abort point:** if any `action_request` is in `confirmed` state (mid-execution), **stop** — a
write is in flight against this tenant's data. Wait for it to reach a terminal state
(`completed` / `failed` / `expired`) before continuing; do not force-decline a `confirmed`
request.

### Step 4 — Mark archived + rename the schema in place

This is the moment the tenant loses live access. It is **reversible** (see Rollback
semantics) — the schema and all its data survive under the archive name.

```sql
-- 4a. Check-and-set the registry row FIRST (this is the idempotency anchor —
--     T-W3-022 does exactly this as its first execution step)
UPDATE velnor.tenants
SET state = 'archived', updated_at = now()
WHERE slug = '<slug>' AND state <> 'archived';
-- expect: UPDATE 1. If UPDATE 0, the tenant is already archived — STOP and
-- take the replay path (read the recorded result back from the latest
-- tenant_archived audit_event; do not re-run Steps 4b–6).

-- 4b. Last look before renaming — confirm this is the schema you mean to archive
SELECT schema_name FROM information_schema.schemata
WHERE schema_name = 'velnor_t_<slug_normalized>';

-- 4c. Rename in place. Nothing is dropped; every table
--     (entity_doc, entity_embedding, action_request, conversation_turn, ...)
--     moves with the schema.
ALTER SCHEMA velnor_t_<slug_normalized> RENAME TO velnor_arch_<slug_normalized>_<yyyymmdd>;
-- <yyyymmdd> = today's UTC date, e.g. velnor_arch_test_delete_me_20260718
```

**Record the exact timestamp and the full archive schema name** — both go into the audit
payload in Step 6, and the archive name is what a rollback renames back.

> **If 4c fails with "must be owner of schema":** the admin-api DB role (or your operator
> role) does not own the schema. Founder-run fallback via the tunnel:
> `ALTER SCHEMA velnor_t_<slug_normalized> OWNER TO <role>;` then re-run 4c. This is the
> accepted DB-permission risk from the W3 design; the dry-run surfaces it if present.

### Step 5 — Disable Cognito users + tag S3 objects

Both operations are reversible and idempotent. Partial failure: continue, collect, report —
then re-drive only the failed items.

```bash
# 5a. Disable every tenant user in the tenant pool (filter client-side on
#     custom:tenant_id — Cognito server-side filters can't target custom attrs)
aws cognito-idp list-users --profile velnor-dev --region us-east-1 \
  --user-pool-id "$COGNITO_TENANT_POOL_ID" \
  --query 'Users[].{u:Username,attrs:Attributes}' --output json \
  | jq -r '.[] | select(.attrs[]? | select(.Name=="custom:tenant_id" and .Value=="<slug>")) | .u' \
  | while read -r u; do
      aws cognito-idp admin-disable-user --profile velnor-dev --region us-east-1 \
        --user-pool-id "$COGNITO_TENANT_POOL_ID" --username "$u"
    done
# AdminDisableUser, NOT AdminDeleteUser — disabled users are re-enable-able.

# 5b. Tag every object under the tenant's uploads prefix (objects are NOT
#     deleted, NOT lifecycle-expired — tagged only)
aws s3api list-objects-v2 --profile velnor-dev --region us-east-1 \
  --bucket "$UPLOADS_BUCKET" --prefix "<tenant_prefix>/" \
  --query 'Contents[].Key' --output json | jq -r '.[]' \
  | while read -r key; do
      aws s3api put-object-tagging --profile velnor-dev --region us-east-1 \
        --bucket "$UPLOADS_BUCKET" --key "$key" \
        --tagging 'TagSet=[{Key=tenant_archived,Value=true}]'
    done
# <tenant_prefix>: T-W3-022 tags tenant/<slug>/ (the upload-path convention).
# NOTE a known discrepancy: DEC-0047's declared key pattern is
# {tenant_id}/original/{doc_id}.{ext} — if any objects exist under the
# tenant_id-keyed prefix, tag that prefix too. The T-W3-023 dry-run checks
# both and records which one actually holds objects in Dev.
```

**Count what you touched** — `users_disabled` and `s3_objects_tagged` go into the Step 6
audit payload and the endpoint response, and the post-archive verification re-checks them.

### Step 6 — Audit confirmation (`tenant_archived`)

Write the archive confirmation into the shared `velnor.audit_event` ledger — this table lives
in `velnor`, not the tenant schema, so it is unaffected by the rename. Real column set is
`(id, tenant_id, event_type, actor_id, payload, created_at)` — there is **no** `tenant_slug`
or `actor_email` column (DEC-0068; the T-W3-021 draft's INSERT targeted columns that never
existed — corrected here):

```sql
INSERT INTO velnor.audit_event (tenant_id, event_type, actor_id, payload)
VALUES (
  '<tenant_id>',
  'tenant_archived',
  '<sha256:hex-token-of-operator-identity>',   -- never a literal email
  jsonb_build_object(
    'slug', '<slug>',
    'schema_renamed_to', 'velnor_arch_<slug_normalized>_<yyyymmdd>',
    'users_disabled', <n>,
    's3_objects_tagged', <n>,
    'idempotency_key', '<the-submitted-key>',
    'performed_by', '<sha256:hex-token-of-operator-identity>',
    'partial_failures', '[]'::jsonb
  )
);
```

T-W3-022 writes this via the existing `internal/audit` writer, and its 200 response mirrors
the payload: `{slug, state:"archived", schema_renamed_to, users_disabled, s3_objects_tagged,
archived_at}`. This audit row is also the **replay source**: a repeat DELETE for an
already-archived tenant returns this recorded result.

### Step 7 — Retention posture (no countdown)

There is **no DR preservation window and no close-out date**: nothing was destroyed, so
nothing ages out. The archived schema, disabled users, and tagged objects persist
indefinitely — at storage cost — until the founder explicitly decides to hard-purge (future
procedure, below) or to restore. Do not let a new tenant reuse this slug while the archive
exists: the registry row (state `archived`) still holds the slug's uniqueness, and the archive
schema still embeds it.

---

## Rollback semantics

Rollback is a first-class outcome of archive semantics — every step has a cheap inverse.

| After step | Rollback path | Cost / caveat |
|---|---|---|
| Step 1 (authorization + pre-flight + preview) | Stop. Nothing has changed. | None. |
| Step 2 (Stripe cancel) | Re-subscribe the tenant via the Stripe Dashboard; no data was lost. | Billing gap during the interval; notify the tenant. |
| Step 3 (decline action_requests) | Manually re-derive and re-submit the declined requests — the underlying entity data still exists. | `declined` state is not itself auto-restorable; someone has to redo the work. |
| Step 4 (state + schema rename) | `ALTER SCHEMA velnor_arch_<slug_normalized>_<yyyymmdd> RENAME TO velnor_t_<slug_normalized>;` then `UPDATE velnor.tenants SET state='active', updated_at=now() WHERE slug='<slug>';` | Cheap. Do it in this order (schema first, then state) so a half-rolled-back tenant never presents as active without its schema. |
| Step 5a (Cognito disable) | `aws cognito-idp admin-enable-user` for each disabled user. | Cheap; users' credentials and attributes were never touched. |
| Step 5b (S3 tagging) | Remove or overwrite the tag: `put-object-tagging` with an empty/changed TagSet. | Cheap; object bytes were never touched. |
| Step 6 (audit confirmation) | N/A — additive, not destructive. Never delete or edit a `tenant_archived` audit_event row, even after a rollback — append a `tenant_unarchived` event instead, so the ledger tells the whole story. | None. |
| Step 7 (retention posture) | N/A — nothing to roll back; no window ever closes. Full rollback of an archived tenant remains possible indefinitely, until a future hard purge (which will have its own point of no return). | — |

---

## Post-archive verification (schema renamed, no active orphans)

```sql
-- 1. Live schema name is gone, archive schema name exists
SELECT schema_name FROM information_schema.schemata
WHERE schema_name = 'velnor_t_<slug_normalized>';
-- expect: 0 rows
SELECT schema_name FROM information_schema.schemata
WHERE schema_name LIKE 'velnor_arch_<slug_normalized>_%';
-- expect: 1 row

-- 2. Registry row is marked archived, not silently vanished
SELECT slug, state, updated_at FROM velnor.tenants WHERE slug = '<slug>';
-- expect: 1 row, state = 'archived'

-- 3. Audit confirmation exists (keyed by tenant_id; slug lives in the payload)
SELECT event_type, payload, created_at FROM velnor.audit_event
WHERE event_type = 'tenant_archived' AND payload->>'slug' = '<slug>'
ORDER BY created_at DESC LIMIT 1;
-- expect: 1 row; payload.partial_failures should be [] — if not, re-drive the
-- failed items from Step 5 and verify again.

-- 4. No active orphan rows in the shared schema — nothing still references the
--    tenant as if it were live. tool_invocation_log history is EXPECTED to
--    remain (DEC-0050 13-month audit window); retained history is not an orphan.
SELECT count(*) FROM velnor.tenants WHERE slug = '<slug>' AND state <> 'archived';
-- expect: 0
SELECT count(*) FROM velnor.tool_invocation_log WHERE tenant_slug = '<slug>';
-- expect: >= 0 — retained by design, attributable, not an orphan. Do NOT delete.
```

```bash
# 5. Cognito users all disabled (Enabled=false for every custom:tenant_id match)
aws cognito-idp list-users --profile velnor-dev --region us-east-1 \
  --user-pool-id "$COGNITO_TENANT_POOL_ID" \
  --query 'Users[].{u:Username,enabled:Enabled,attrs:Attributes}' --output json \
  | jq '[.[] | select(.attrs[]? | select(.Name=="custom:tenant_id" and .Value=="<slug>")) | .enabled] | all(. == false)'
# expect: true

# 6. S3 objects all carry the archive tag (spot-check each key from Step 5b)
aws s3api get-object-tagging --profile velnor-dev --region us-east-1 \
  --bucket "$UPLOADS_BUCKET" --key "<one-of-the-keys>" \
  --query "TagSet[?Key=='tenant_archived'].Value"
# expect: ["true"]
```

**Pass criteria:** live schema absent + archive schema present; registry row present with
`state = 'archived'`; exactly one fresh `tenant_archived` audit_event with empty
`partial_failures`; every tenant user disabled; every tenant object tagged; no shared-table
row presenting the tenant as active. If any check fails, stop and escalate — do not mark the
archival task complete.

---

## Edge cases

- **Active subscription** (Step 2) — must be cancelled before Step 4. No durable DB record of
  the tenant's Stripe subscription exists yet (see Known gap in Step 2); follow-up work should
  add a queryable subscription reference or perform the metadata-filtered Stripe API lookup
  programmatically.
- **Pending action_requests** (Step 3) — anything `proposed`/`pending` is auto-declined with
  `decline_reason: tenant_archived` recorded in its `payload`. Anything `confirmed`
  (mid-execution) blocks Step 4 until it reaches a terminal state.
- **Repeat archive request** — state-derived idempotency (Step 0): a second DELETE for an
  already-archived tenant returns 200 with the recorded result from the latest
  `tenant_archived` audit_event. It does not re-rename, re-disable, or re-tag anything.
- **Partial failures in Step 5** — recorded in the audit payload's `partial_failures` array
  and the endpoint response. Re-drive only the failed items (both operations are idempotent),
  then re-run the post-archive verification.
- **Glacier-archived audit data** — `tool_invocation_log` rows archived to S3 Glacier are
  retained for the full 13-month admin audit window (DEC-0050) **regardless of tenant
  archival**. Do not interpret retained audit history as an incomplete archival.
- **Slug reuse** — blocked while the archive exists: the `velnor.tenants` row keeps the slug
  (UNIQUE) in state `archived`. Freeing a slug requires the future hard purge.

---

## Future work — hard purge (dump → verify → drop) — NOT IMPLEMENTED

A true, irreversible purge of an **already-archived** tenant is a separate future runbook and
endpoint. It is deliberately **not implemented** in T-W3-022 and must never be improvised from
this document. Its shape, recorded here so the archive design is legible:

1. **Pre-conditions:** tenant already `archived` via this procedure; a fresh, explicit purge
   consent record (archive consent does not carry over); founder sign-off; a verified Aurora
   backup checkpoint.
2. **Dump:** `pg_dump --schema=velnor_arch_<slug_normalized>_<yyyymmdd>` to durable, encrypted
   off-cluster storage; capture the S3 object-version listing for the tenant prefix.
3. **Verify:** restore the dump into a scratch database and reconcile row counts per table
   against the live archive schema; verify the S3 listing is complete. The purge does not
   proceed until the backup verification passes.
4. **Drop:** `DROP SCHEMA velnor_arch_<slug_normalized>_<yyyymmdd> CASCADE`; delete S3 object
   versions + delete markers under the tenant prefix (the bucket is versioned — a plain
   delete only adds markers); `AdminDeleteUser` the disabled users; append a `tenant_purged`
   audit_event. This step is the point of no return.
5. **Retention interlock:** Glacier-archived `tool_invocation_log` history is still retained
   through the DEC-0050 window even after a purge.

Open items the future purge design must resolve: no S3 lifecycle rule exists on the uploads
bucket today; there is no durable Stripe subscription reference; the purge must define its own
DR preservation window before the drop is final.

---

## Escalation contacts

| Role | When to page | Escalation order |
|---|---|---|
| Architect | Any ambiguity in the Step 0 guards, an abort-point condition (Steps 2–3), or a Step 4 ownership failure | 1st |
| Founder | A consent question, a Stripe cancellation dispute, a rollback decision, or anything touching the future hard purge | 2nd |
| AWS Support | Cognito/S3 API failures that survive retries during Step 5 | 3rd |

---

## Sign-off

This runbook reflects the **founder-approved archive semantics** (2026-07-18 W3 design
session), which supersede the T-W3-021 DROP-CASCADE draft. It gates:

- **T-W3-022** — `DELETE /api/v1/admin/tenants/{slug}` + deletion-preview, automating Step 0's
  guards and Steps 4–6 as a synchronous handler (no Step Function).
- **T-W3-023** — Engineering dry-run of this exact procedure against a throwaway Dev tenant
  (`velnor-evals/integration/tenant_deletion_dryrun.py`). Revision notes from the dry-run are
  appended below.

### Dry-run revision notes (T-W3-023)

> (appended by the dry-run task as discoveries land — empty until the dry-run executes)
