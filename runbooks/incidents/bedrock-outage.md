---
layout: default
title: "Bedrock Outage / Model Unavailability"
nav_order: 1
parent: "Incident Response Playbooks"
adr_refs:
  - DEC-0022
  - DEC-0046
  - DEC-0051
severity: SEV-2
last_updated: 2026-05-24
---

# Bedrock Outage / Model Unavailability

> **Severity:** SEV-2 (degraded service; tenants cannot complete AI-assisted chat turns)  
> **Who is paged:** Architect (primary), Founder (escalation)  
> **Expected duration:** 15–60 min (fallback activation); full restore on AWS recovery  
> **Escalation chain:** Architect → Founder → AWS Support (P1 ticket)

---

## Purpose

Step-by-step response for when Amazon Bedrock becomes unreachable or returns sustained errors
for the Claude model used by the Velnor AI Harness chat path. Covers detection, triage,
fallback activation (Anthropic-direct API or OpenAI), and restoration.

---

## Triggering alarm

| Field | Value |
|-------|-------|
| **CloudWatch alarm name** | `velnor-bedrock-error-rate-high` |
| **Namespace / metric** | `VELNOR/Providers` / `bedrock_error_count` |
| **Threshold** | error rate >= 10% over 5-minute window |
| **SNS topic** | `arn:aws:sns:us-east-1:<ACCOUNT_ID>:velnor-ops-alert` |
| **Alert delivery** | SNS → Lambda subscriber → SES email to Architect (DEC-0046) |
| **Dashboard** | `velnor-dev-providers` — Bedrock call count, fallback count, error rate |

---

## Detection

### Step 1 — Confirm the alarm is real (< 2 min)

1. Open CloudWatch → Alarms → search `velnor-bedrock-error-rate-high`.
2. Confirm state is **ALARM** (not a transient blip — check the 5-minute metric graph).
3. Open the `velnor-dev-providers` dashboard and check:
   - `bedrock_error_count` — is it sustained, or a single spike?
   - `bedrock_call_count` — are calls being made at all, or is the SDK failing before reaching Bedrock?
4. Check the [AWS Service Health Dashboard](https://health.aws.amazon.com/health/status) for an open Bedrock incident in `us-east-1`.

### Step 2 — Rule out local causes (< 3 min)

Check the following before assuming AWS-side outage:

- **VPC endpoint health.** Bedrock traffic routes through the `com.amazonaws.us-east-1.bedrock-runtime` VPC endpoint. Console → VPC → Endpoints → confirm state is `available`.
- **IAM / credentials.** Check CloudTrail for `bedrock:InvokeModel` `AccessDenied` events in the last 10 minutes.
- **Model ID mismatch.** Confirm the model ID in SSM Parameter Store (`/velnor/dev/bedrock/model_id`) matches an active Bedrock model. Claude model IDs: `anthropic.claude-3-5-sonnet-20241022-v2:0` (primary).
- **Throttling vs. hard error.** HTTP 429 = throttle (quota exhaustion); HTTP 500/503 = Bedrock-side error. CloudWatch Logs → `/velnor/dev/agent-gateway` — look for the error code in structured log entries.

---

## Triage

### Step 3 — Classify the outage

| Symptom | Classification | Action |
|---------|---------------|--------|
| HTTP 503 from Bedrock + AWS Health event | AWS-side full outage | Proceed to fallback (Step 4A) |
| HTTP 429 on all calls | Quota exhaustion — throttle | Proceed to Step 4B |
| HTTP 401 / AccessDenied | IAM issue | Fix credentials; no fallback needed |
| HTTP 400 / ValidationException | Model ID invalid | Fix SSM param; no fallback needed |
| < 20% error rate, intermittent | Transient blip | Monitor for 5 min; no action yet |

### Step 4A — Full Bedrock outage: activate provider fallback

The Velnor AI Harness supports two fallback paths (DEC-0022 / DEC-0049):

1. **Anthropic-direct API** — uses the `anthropic` Python SDK pointing to `api.anthropic.com`. API key stored in Secrets Manager at `/velnor/dev/anthropic/api_key`.
2. **OpenAI** — fallback for embedding / narration if Anthropic-direct also fails. Key at `/velnor/dev/openai/api_key`.

**To activate Anthropic-direct fallback:**

```bash
# 1. Confirm the Anthropic API key is present
aws secretsmanager get-secret-value \
  --secret-id /velnor/dev/anthropic/api_key \
  --query SecretString --output text

# 2. Update the provider-mode SSM param (flips the SDK routing at runtime)
aws ssm put-parameter \
  --name /velnor/dev/ai_harness/provider_mode \
  --value "anthropic-direct" \
  --overwrite

# 3. Trigger a rolling ECS redeploy so all tasks pick up the new param
aws ecs update-service \
  --cluster velnor-dev \
  --service velnor-agent-gateway \
  --force-new-deployment

# 4. Monitor — watch for successful Anthropic calls in:
#    CloudWatch Logs → /velnor/dev/agent-gateway
#    Dashboard: velnor-dev-providers → bedrock_call_count should drop; fallback_count should rise
```

**Expected time to fallback-active:** 3–5 minutes (ECS rolling update).

**To activate OpenAI fallback** (only if Anthropic-direct also fails):

```bash
aws ssm put-parameter \
  --name /velnor/dev/ai_harness/provider_mode \
  --value "openai" \
  --overwrite

aws ecs update-service \
  --cluster velnor-dev \
  --service velnor-agent-gateway \
  --force-new-deployment
```

**Note:** OpenAI responses will have different formatting characteristics. Inform the founder.
Embedding model falls back to `text-embedding-3-small`; ensure the model dimension
(1536) matches what is stored in pgvector — if it does not, disable semantic cache lookup
temporarily (set `/velnor/dev/cache/enabled` = `false`).

### Step 4B — Quota exhaustion (HTTP 429)

1. Check current Bedrock quota in Service Quotas console: `us-east-1 → Bedrock → Invocations per minute per model`.
2. If quota is near the limit: submit a Bedrock quota-increase request via AWS Support (reference U-12 from the implementation plan).
3. Short-term mitigation: enable request queuing by setting `/velnor/dev/ai_harness/rate_limit_mode` = `queue` in SSM. This holds excess requests in SQS rather than returning errors, at the cost of increased latency.
4. Alert the founder of the quota situation and expected timeline for increase (typically 2–5 business days for Bedrock, per U-12).

---

## Communicate

### Step 5 — Internal communication (< 5 min after classification)

Send a brief message to the Velnor ops channel (Slack / email per the team's current comms tool):

```
[INCIDENT ACTIVE] Bedrock outage detected at <time>.
Severity: SEV-2. AI-assisted chat is degraded for all tenants.
Fallback provider: <anthropic-direct | openai | pending>.
ETA to restoration: <estimate or "monitoring AWS Health for recovery ETA">.
Next update in 30 min.
```

---

## Restore

### Step 6 — Restore Bedrock as primary provider

Monitor AWS Health Dashboard for the Bedrock incident to close. Once Bedrock is confirmed healthy:

```bash
# 1. Validate Bedrock is responding
aws bedrock-runtime invoke-model \
  --model-id anthropic.claude-3-5-sonnet-20241022-v2:0 \
  --body '{"anthropic_version":"bedrock-2023-05-31","max_tokens":10,"messages":[{"role":"user","content":"ping"}]}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/bedrock-health-check.json && cat /tmp/bedrock-health-check.json

# 2. Revert the provider-mode SSM param to bedrock
aws ssm put-parameter \
  --name /velnor/dev/ai_harness/provider_mode \
  --value "bedrock" \
  --overwrite

# 3. Rolling redeploy to pick up the change
aws ecs update-service \
  --cluster velnor-dev \
  --service velnor-agent-gateway \
  --force-new-deployment

# 4. Re-enable semantic cache if it was disabled during OpenAI fallback
aws ssm put-parameter \
  --name /velnor/dev/cache/enabled \
  --value "true" \
  --overwrite
```

### Step 7 — Confirm alarm cleared

1. CloudWatch → `velnor-bedrock-error-rate-high` should return to `OK`.
2. Dashboard `velnor-dev-providers` → `bedrock_call_count` rising, `fallback_count` back to zero.
3. Run a smoke test: call the `/chat` endpoint with a test tenant and confirm a valid AI response.

---

## Rollback criteria

- If Bedrock is restored but the platform does not recover within 10 minutes of the SSM param revert, roll back the ECS task definition to the previous revision and page the Architect.

---

## Postmortem template

File within 48 hours of incident close.

```markdown
## Bedrock Outage Postmortem — <date>

**Incident start:** <timestamp>
**Fallback activated at:** <timestamp>
**Bedrock restored at:** <timestamp>
**Total SEV-2 window:** <duration>

### Timeline

| Time | Event |
|------|-------|
| ...  | ...   |

### Root cause

<AWS-side or local — describe>

### Impact

- Tenants affected: <count or "all">
- Chat requests failed / queued: <count>
- Semantic cache behavior during fallback: <describe>

### What went well

### What went poorly

### Action items

| Item | Owner | Due |
|------|-------|-----|
| ...  | ...   | ... |
```

---

## Contacts

| Role | Contact | Escalation order |
|------|---------|-----------------|
| Architect (primary) | See PagerDuty / Slack | 1st |
| Founder | See PagerDuty / Slack | 2nd |
| AWS Support | https://console.aws.amazon.com/support | 3rd (P1 ticket for confirmed AWS-side outage) |
