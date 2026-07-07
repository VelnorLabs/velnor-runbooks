---
layout: default
title: "Tenant Deletion (Architect Draft — Awaiting Sign-off)"
nav_order: 4
parent: "Operations Runbooks"
adr_refs:
  - DEC-0035
  - DEC-0047
  - DEC-0065
  - DEC-0050
  - DEC-0046
severity: destructive / irreversible after the DR preservation window closes
last_updated: 2026-07-06
status: "DRAFT — T-W3-021, awaiting founder Architect sign-off"
---

# Tenant Deletion

> **Status:** DRAFT. This is an **Architect draft** per T-W3-021. It has not been dry-run against
> a real (even throwaway) tenant yet — that happens in **T-W3-023**. Do not run this against any
> tenant, Dev or otherwise, until the founder (Architect hat) has signed off and T-W3-023 has
> passed.
>
> **Severity:** Destructive. Recoverable only within the DR preservation window (Step 7); no
> rollback after that window closes.
> **Who performs this:** Founder or Architect only, by hand, until T-W3-022 ships the automated
> endpoint (which must implement Step 0's three checks in code).
> **Expected duration:** ~15–20 min of active work, then a standing 7-day (Dev; see Step 7 for the
> Staging/Prod figure once those accounts exist) DR preservation window before the deletion is
> final.
> **Escalation chain:** Architect → Founder → AWS Support (only for a Step 4 rollback attempt)

---

## Purpose

Deletes a Velnor tenant end-to-end: drops the tenant's Postgres schema (DEC-0035 rev. 2), expires
the tenant's S3 document originals (DEC-0047), records an audit confirmation in the shared
`velnor` schema, and holds a DR preservation window before the deletion is considered final.

This procedure is the source of truth that **T-W3-022** (`DELETE
/api/v1/admin/tenants/{slug}`) must automate behind three independent authorization checks, and
that **T-W3-023** dry-runs against a throwaway Dev tenant before either can move to
`done_verified`.

Inputs: 2026-05-17 V1 implementation plan §2.5 Q-6 (step ordering), §2.6 Q-15 (deletion semantics).
Anchors: DEC-0035 rev. 2 (schema-per-tenant), DEC-0047 (three-layer tenant document storage),
DEC-0065 (visibility/RLS — confirms tenant isolation is schema-scoped, not a `tenant_id` column,
inside the tenant schema), DEC-0050 (audit retention: 90-day hot / 13-month Glacier window), DEC-0046
(CloudWatch → SNS → SES alert routing, for the escalation paths below).

---

## Scope

- **In scope:** the tenant's schema (`velnor_t_{slug}`); the tenant's S3 objects under
  `{tenant_id}/` in `velnor-dev-uploads` (DEC-0047 key pattern `{tenant_id}/original/{doc_id}.{ext}`);
  the tenant's row in the shared registry (`velnor.tenants`); the tenant's pending
  `action_request` rows; the tenant's Stripe subscription, if any.
- **Out of scope:** the shared `velnor` schema itself and its other tenants' rows;
  `velnor.golden_template`; Glacier-archived `tool_invocation_log` history for this tenant
  (retained for the full 13-month DEC-0050 audit window regardless of tenant deletion — see
  Edge cases).

---

## Step 0 — Authorization: three independent checks

**T-W3-022 must implement all three of the following in the endpoint before the delete workflow
is allowed to start.** They are independent — none may substitute for another — and failing any
one aborts before any destructive action runs.

1. **confirm-slug** — the request body must include `"confirm_slug": "<slug>"`, and it must be a
   byte-exact match to the `{slug}` path parameter. Purpose: stops a wrong-tenant deletion caused
   by a UI misclick or a copy-pasted slug from a different tab. Reject with `400` on any mismatch,
   before the idempotency or MFA checks even run.
2. **Idempotency-Key** — the request must carry an `Idempotency-Key` header (a UUID). The server
   persists the key (a dedupe row keyed on `(operation, idempotency_key)`) *before* triggering the
   Step Functions Express workflow (DEC-0039 `modules/safe_workflow/`), so a client retry after a
   timeout cannot re-invoke the destructive workflow a second time. A re-used key whose workflow is
   still `RUNNING` gets `409`; a re-used key whose workflow already reached a terminal state
   returns that same terminal result instead of re-running anything.
3. **MFA-within-5min** — the caller's session must show a step-up MFA challenge (TOTP or WebAuthn)
   completed within the last 5 minutes — not merely "logged in with MFA at some point." Check an
   `mfa_verified_at` session/claim; if it is absent or older than 5 minutes, return `401`/`428`
   and require re-authentication before the request can be retried.

**Manual-run equivalent (this runbook, before T-W3-022 ships):** the operator performs all three by
hand — re-type the slug at the Step 1 confirmation prompt (do not copy-paste it from a ticket, to
force a second read), treat the run as one-shot (start over from Step 1 rather than repeating a
single step after an interruption), and confirm an MFA-backed `velnor-dev` SSO login happened
within the last 5 minutes before Step 4.

### Implementation hazards for T-W3-022 (security review)

These do not block the manual runbook (an operator typing literal values by hand isn't exposed to
them) but they are real hazards for the automated endpoint and must be closed before T-W3-022
ships:

- **`slug` must be allow-list validated before it ever reaches DDL.** Step 4's `DROP SCHEMA
  velnor_t_<slug> CASCADE` and Step 3's `velnor_t_<slug>.action_request` are schema-qualified
  identifiers — Postgres has no parameterized-identifier form for DDL (unlike a `$1` value
  placeholder), so if the endpoint builds this statement by string-formatting the path parameter
  directly, an attacker-controlled `slug` containing `; DROP SCHEMA velnor; --` (or any Postgres
  identifier-breaking sequence) is a DDL injection vector. **confirm-slug (Step 0.1) checks that
  the body matches the path parameter — it does not validate that either value is a safe
  identifier.** T-W3-022 must validate `slug` against the same charset the provisioning path
  already constrains it to (schema names are derived from `slug` at tenant creation — reuse that
  validator) *before* Step 0's checks run, and build the DDL via a quoted-identifier helper (e.g.
  `pgx`'s `pgx.Identifier{...}.Sanitize()` or equivalent), never raw string interpolation.
- **Concurrent double-invocation isn't covered by the Idempotency-Key check alone.** Two requests
  for the same tenant with two *different* Idempotency-Keys both pass Step 0 independently and can
  race into Step 4 concurrently — the dedupe row only stops a retry of the *same* key. T-W3-022
  should acquire a per-tenant advisory lock (or check-and-set `velnor.tenants.state =
  'deletion_in_progress'`) as the first action inside the workflow, before Step 2, so a second
  concurrent request fails fast instead of racing another instance of this same procedure.
- **No audit trail exists if the workflow crashes between Step 4 and Step 6.** As drafted, the only
  audit write (`tenant_deleted`) happens *after* the destructive steps — if the process dies
  between the Step 4 `DROP` and the Step 6 `INSERT`, there is no record in `velnor.audit_event` that
  a deletion was even attempted. T-W3-022 must write a `tenant_deletion_started` audit_event (same
  shape as Step 6, `event_type = 'tenant_deletion_started'`) as its first workflow action, before
  Step 2, so a forensic trail exists regardless of where the workflow stops. DEC-0039's
  `safe_workflow` bounded retries + DLQ then apply to the remaining steps as usual.

---

## Pre-flight checklist

All boxes must be checked before Step 1. If any is unchecked, stop.

- [ ] **Consent obtained** — written confirmation from the tenant (or the Founder, for a
      ToS/compliance-driven removal) that deletion is authorized. Attach to the task ticket.
- [ ] **Backups verified** — confirm Aurora's automated backup is current:

  ```bash
  aws rds describe-db-clusters --region us-east-1 --profile velnor-dev \
    --db-cluster-identifier velnor-dev \
    --query 'DBClusters[0].{Retention:BackupRetentionPeriod,LatestRestorableTime:LatestRestorableTime}'
  # expect Retention=7 (Dev, per velnor-iac/accounts/dev/aurora/main.tf) and
  # LatestRestorableTime within the last few minutes.
  # Staging/Prod are not yet provisioned (Phase 2, Wk 12 per CLAUDE.md) — when they exist,
  # confirm that account's own retention value before running this against a non-Dev tenant.
  ```

- [ ] **No active workflows on the tenant** — confirm no in-flight Step Functions executions
      reference this tenant:

  ```bash
  aws stepfunctions list-executions --region us-east-1 --profile velnor-dev \
    --state-machine-arn <safe_workflow-state-machine-arn> --status-filter RUNNING \
    --query "executions[?contains(name, '<slug>')]"
  # expect: []
  ```

- [ ] **Subscription status checked** — see Step 2 / Edge cases: active subscription.
- [ ] **Pending action_requests checked** — see Step 3 / Edge cases: pending action_requests.

---

## Ordered procedure

Open the SSM bastion tunnel first — see `runbooks/operations/dev-db-validation.md` §0 for the full
walkthrough (fetching the master password, `127.0.0.1` vs `localhost`, etc.):

```bash
cd velnor-iac
./scripts/bastion.sh up
./scripts/bastion.sh tunnel      # Aurora-direct 5432 -> localhost:5432
```

Capture the tenant's identifiers up front — you need both, and `tenant_id` becomes unrecoverable
from Postgres after Step 4:

```sql
SELECT tenant_id, slug, display_name, plan, state
FROM velnor.tenants
WHERE slug = '<slug>';
-- record tenant_id (UUID) — needed for the S3 prefix in Step 5 and the audit
-- payload in Step 6.
```

### Step 1 — Authorization + pre-flight

Complete Step 0 and the full pre-flight checklist above. **Abort point:** nothing has changed yet;
stopping here is a no-op.

### Step 2 — Cancel the Stripe subscription (if active)

> **Known gap — flag for T-W3-022:** `velnor.tenants` has no `stripe_subscription_id` column
> (`velnor-schemas/postgres/velnor_shared/v0.sql:13-21`), and the only subscription store in the
> codebase today is an **in-memory stub**
> (`velnor-workers/internal/stripe/handlers/subscription.go` —
> `InMemoryMembershipStore`, explicitly commented "replace with a database-backed implementation
> before production use"). There is no durable, queryable tenant→subscription mapping yet. The
> Stripe webhook handler keys subscriptions by `metadata["tenant_id"]` on the Stripe object, so
> that is the only reliable lookup path today.

1. In the Stripe Dashboard, search Customers/Subscriptions by metadata `tenant_id = <tenant_id
   from above>`.
2. If an active subscription exists, cancel it immediately (Subscription → Cancel immediately, not
   at period end — a deleted tenant should not continue to bill).
3. Confirm the cancellation webhook landed:

   ```sql
   SELECT * FROM velnor.audit_event
   WHERE tenant_slug = '<slug>' AND event_type = 'subscription_cancelled'
   ORDER BY created_at DESC LIMIT 1;
   -- expect one row, created within the last few minutes
   ```

**Abort point:** if no active subscription is found, or cancellation is confirmed, continue. If
cancellation fails or the webhook doesn't land within a few minutes, stop and escalate to the
Architect — do not proceed to Step 4 with a live subscription still billing a schema that's about
to be dropped.

### Step 3 — Decline pending action_requests

```sql
-- inspect what's pending in the tenant's own schema
SELECT id, state, payload FROM velnor_t_<slug>.action_request
WHERE state IN ('proposed', 'pending')
ORDER BY id;

-- decline them all. There is no dedicated "reason" column on action_request
-- (velnor-action-gate/migrations/001_action_request.sql:9-49) — record the
-- reason inside payload instead.
UPDATE velnor_t_<slug>.action_request
SET state = 'declined',
    payload = payload || jsonb_build_object('decline_reason', 'tenant_deleted', 'declined_at', now())
WHERE state IN ('proposed', 'pending');
```

**Abort point:** if any `action_request` is in `confirmed` state (mid-execution, not merely
proposed/pending), **stop** — a write is in flight against this tenant's data. Wait for it to reach
a terminal state (`completed` / `failed` / `expired`) before continuing; do not force-decline a
`confirmed` request.

### Step 4 — Schema DROP CASCADE

This is the point of no return for live query access to this tenant's data. From here, the only
path back is an Aurora point-in-time restore, and only within the DR preservation window (Step 7 /
Rollback table below).

```sql
-- last look before dropping — confirm this is the schema you mean to drop
SELECT schema_name FROM information_schema.schemata WHERE schema_name = 'velnor_t_<slug>';

-- the drop. velnor_t_<slug> holds: entity_row, entity_edge, entity_embedding,
-- entity_doc, action_request, conversation_turn, audit_event, agent_run
-- (DEC-0035 rev. 2) — CASCADE removes all of them together with the schema.
DROP SCHEMA velnor_t_<slug> CASCADE;
```

**Record the exact timestamp this ran.** It is the anchor for the DR preservation window in Step 7
and goes into the audit payload in Step 6.

### Step 5 — S3 object expiration

`velnor-dev-uploads` holds this tenant's document originals under `{tenant_id}/original/{doc_id}.{ext}`
(DEC-0047; bucket defined at `velnor-iac/accounts/dev/ecs/embed-worker/uploads_bucket.tf:74-75`).
The bucket has **versioning enabled** and, as of this writing, **no lifecycle rule** — so a plain
`aws s3 rm` only adds a delete marker; prior object versions remain in the bucket (and billed,
and technically recoverable) until explicitly purged.

```bash
# 1. Confirm what's under the prefix before touching anything
aws s3 ls "s3://velnor-dev-uploads/<tenant_id>/" --recursive --profile velnor-dev

# 2. Add delete markers on current versions (recoverable up to this point —
#    see Rollback table)
aws s3 rm "s3://velnor-dev-uploads/<tenant_id>/" --recursive --profile velnor-dev

# 3. Capture every version + delete-marker under the prefix before purging —
#    this listing IS the only record of what existed; save it somewhere
#    durable (not just /tmp) if there is any chance Step 4's rollback path
#    will be exercised.
aws s3api list-object-versions --bucket velnor-dev-uploads --prefix "<tenant_id>/" \
  --profile velnor-dev --region us-east-1 \
  --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json \
  > /tmp/<tenant_id>-versions.json

aws s3api list-object-versions --bucket velnor-dev-uploads --prefix "<tenant_id>/" \
  --profile velnor-dev --region us-east-1 \
  --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}' --output json \
  > /tmp/<tenant_id>-markers.json

# 4. Purge all object versions AND delete markers under the prefix — this is
#    the irreversible step (required for real erasure, since the bucket has
#    versioning enabled).
aws s3api delete-objects --bucket velnor-dev-uploads --profile velnor-dev --region us-east-1 \
  --delete file:///tmp/<tenant_id>-versions.json
aws s3api delete-objects --bucket velnor-dev-uploads --profile velnor-dev --region us-east-1 \
  --delete file:///tmp/<tenant_id>-markers.json

# 5. Verify empty
aws s3 ls "s3://velnor-dev-uploads/<tenant_id>/" --recursive --profile velnor-dev
# expect: no output
```

> **Build note for T-W3-022:** DEC-0047's declared lifecycle (STANDARD → STANDARD-IA at 90d →
> GLACIER at 365d, cross-region replica in `us-west-2`) is **not implemented** in `velnor-iac` for
> `velnor-dev-uploads` today — only SSE-KMS + versioning exist. Until a per-prefix
> lifecycle/expiration rule exists, the automated endpoint must do the explicit
> list-then-delete-versions sequence above (steps 3–4), scoped to the tenant's own prefix only —
> never a bucket-wide operation.

### Step 6 — Audit confirmation

Write the deletion confirmation into the shared `velnor.audit_event` table — this table lives in
`velnor`, not `velnor_t_<slug>`, so it survives the Step 4 drop
(`velnor-admin-api/migrations/001_audit_event.sql:13-21`):

```sql
INSERT INTO velnor.audit_event (id, tenant_slug, actor_email, event_type, payload, created_at)
VALUES (
  gen_random_uuid(),
  '<slug>',
  '<sha256:hex-token-of-operator-email>',
  'tenant_deleted',
  jsonb_build_object(
    'tenant_id', '<tenant_id>',
    'schema_dropped_at', '<Step-4-timestamp>',
    's3_prefix_purged', true,
    'dr_preservation_window_ends', '<Step-4-timestamp + backup retention days>'
  ),
  now()
);
```

Then mark the registry row (`velnor.tenants` has no `deleted_at` column — reuse the existing
`state` column, which already carries free-text values like `'active'`):

```sql
UPDATE velnor.tenants SET state = 'deleted', updated_at = now() WHERE slug = '<slug>';
```

### Step 7 — DR preservation window

The tenant's schema is still recoverable via Aurora point-in-time restore for as long as the
automated backup retains it: **7 days in Dev**
(`backup_retention_period = 7`, `velnor-iac/accounts/dev/aurora/main.tf:233`). Staging and Prod are
not yet provisioned (Phase 2, Wk 12 per CLAUDE.md) — confirm that account's own
`backup_retention_period` before treating this figure as authoritative there.

Do not consider the deletion final, and do not let a new tenant reuse this slug, until this window
has elapsed:

```bash
# compute the close-out date (Dev, 7-day window)
date -v+7d -j -f "%Y-%m-%d" "<Step-4-date>" "+%Y-%m-%d"   # macOS
# date -d "<Step-4-date> +7 days" +%Y-%m-%d               # GNU/Linux
```

No action is required at close-out — the automated backup simply ages out. This step exists so an
operator asking "can this still be undone?" has a definitive, dated answer, not so a follow-up
task needs to be scheduled.

---

## Rollback semantics

| After step | Rollback path | Cost / caveat |
|---|---|---|
| Step 1 (authorization + pre-flight) | Stop. Nothing has changed. | None. |
| Step 2 (Stripe cancel) | Re-subscribe the tenant via the Stripe Dashboard; no data was lost. | Billing gap during the interval; notify the tenant. |
| Step 3 (decline action_requests) | Manually re-derive and re-submit the declined requests — the underlying entity data still exists (schema not yet dropped at this point). | `declined` state is not itself auto-restorable; someone has to redo the work. |
| Step 4 (schema DROP CASCADE) | **Aurora point-in-time restore only, and only within the Step 7 window.** Restore the cluster to a new instance at a timestamp just before the DROP ran, `pg_dump --schema=velnor_t_<slug>` from the restored instance, then `pg_restore` into the live cluster. | Expensive (new RDS instance, manual extraction), requires Architect + Founder sign-off to execute, and does **not** resurrect anything already purged in Step 5 — Aurora backups and S3 are independent recovery domains. |
| Step 5, item 2 (`aws s3 rm` — delete markers added) | Still recoverable: `aws s3api delete-object --key <key> --version-id <delete-marker-version-id>` removes the marker and un-hides the prior version. | Cheap, but only until item 4 runs. |
| Step 5, item 4 (`delete-objects` — versions purged) | **No rollback.** `delete-objects` on explicit version IDs is permanent — S3 does not retain a version once its version ID is deleted. | Irreversible. |
| Step 6 (audit confirmation) | N/A — this step is additive, not destructive. Never delete or edit a `tenant_deleted` audit_event row, even if the deletion is later reversed via Step 4's path — append a `tenant_deletion_reversed` event instead. | None. |
| Step 7 (DR window closes) | **No rollback.** This is the actual point of no return — after this, Step 4's restore path is gone too. | — |

---

## Post-deletion verification (no orphan rows in shared schema)

```sql
-- 1. Tenant schema is gone
SELECT schema_name FROM information_schema.schemata WHERE schema_name = 'velnor_t_<slug>';
-- expect: 0 rows

-- 2. Registry row is marked deleted, not silently vanished
SELECT slug, state, updated_at FROM velnor.tenants WHERE slug = '<slug>';
-- expect: 1 row, state = 'deleted'

-- 3. Audit confirmation exists
SELECT event_type, payload, created_at FROM velnor.audit_event
WHERE tenant_slug = '<slug>' AND event_type = 'tenant_deleted'
ORDER BY created_at DESC LIMIT 1;
-- expect: 1 row

-- 4. tool_invocation_log history still resolves — historical rows for this
--    tenant are EXPECTED to remain (retained for the DEC-0050 13-month audit
--    window); this check confirms they exist and are attributable, not that
--    they were deleted. Do NOT treat these as orphans.
SELECT count(*) FROM velnor.tool_invocation_log WHERE tenant_slug = '<slug>';
-- expect: > 0 if the tenant had any tool activity — retained by design, not an orphan
```

```bash
# 5. S3 prefix is empty
aws s3 ls "s3://velnor-dev-uploads/<tenant_id>/" --recursive --profile velnor-dev
# expect: no output
```

**Pass criteria:** schema absent; registry row present with `state = 'deleted'`; exactly one
`tenant_deleted` audit_event; S3 prefix empty. If any check fails, stop and escalate — do not mark
the deletion task complete.

---

## Edge cases

- **Active subscription** (Step 2) — must be cancelled before Step 4. No durable DB record of the
  tenant's Stripe subscription exists yet (see Known gap in Step 2); T-W3-022 should either add a
  queryable subscription reference or perform the metadata-filtered Stripe API lookup
  programmatically.
- **Pending action_requests** (Step 3) — anything `proposed`/`pending` is auto-declined with
  `decline_reason: tenant_deleted` recorded in its `payload`. Anything `confirmed` (mid-execution)
  blocks Step 4 until it reaches a terminal state.
- **Glacier-archived audit data** — `tool_invocation_log` rows archived to S3 Glacier are retained
  for the full 13-month admin audit window (DEC-0050) **regardless of tenant deletion**. This
  procedure never touches Glacier-archived audit history — see Post-deletion verification, item 4.
  Do not interpret retained audit history as an incomplete deletion.

---

## Escalation contacts

| Role | When to page | Escalation order |
|---|---|---|
| Architect | Any ambiguity in the Step 0 authorization checks, or an abort-point condition (Steps 2–3) is hit | 1st |
| Founder | A consent question, a Stripe cancellation dispute, or a Step 4/5 rollback is being considered | 2nd |
| AWS Support | An Aurora PITR restore attempt (Step 4 rollback) is failing on the RDS side | 3rd |

---

## Sign-off

This runbook is an **Architect draft** (T-W3-021). It gates:

- **T-W3-022** — `DELETE /api/v1/admin/tenants/{slug}`, automating Step 0 (three independent
  checks) plus Steps 1–7 as a Step Functions Express workflow (DEC-0039 `safe_workflow` module).
- **T-W3-023** — Engineering dry-run of this exact procedure against a throwaway Dev tenant.

**Founder Architect sign-off is required** before T-W3-021 moves from `done` to `done_verified`.
The sign-off note should confirm: the Step 0 authorization model is correct and sufficient; the
Rollback semantics table is acceptable risk; and the two flagged gaps (no durable Stripe
subscription reference, no S3 lifecycle rule on `velnor-dev-uploads`) are tracked as follow-up
work rather than blockers to drafting T-W3-022.
