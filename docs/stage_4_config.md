# Stage 4: Email Validation — Requirements & Config

## Status: Complete

---

## Purpose

Stage 4 is the most critical gate in the pipeline. No email may proceed to Stage 5 without passing validation. Every address — whether found by Hunter.io or guessed by the Pattern Generator — must be verified before any send attempt.

---

## Input (from Stage 3 via Main, after enrichment)

Each item arriving at Stage 4 Workflow Input:

```json
{
  "batchId": "string",
  "businessType": "string",
  "requestedLeadCount": 5,
  "businessName": "string",
  "websiteUrl": "string",
  "location": "string",
  "phoneNumber": "string | null",
  "email": "string",
  "emailSource": "hunter | pattern",
  "source": "outscraper"
}
```

`batchId`, `businessType`, and `requestedLeadCount` are injected by the `Prepare Stage 4 Input` Code node in Main before `Execute Stage 4` runs.

---

## Output (contract for Stage 5)

Each item that exits Stage 4 (valid and catch-all leads only):

```json
{
  "batchId": "string",
  "businessType": "string",
  "requestedLeadCount": 5,
  "businessName": "string",
  "websiteUrl": "string",
  "location": "string",
  "phoneNumber": "string | null",
  "email": "string",
  "emailSource": "hunter | pattern",
  "source": "outscraper",
  "validationStatus": "valid | catch-all",
  "validationProvider": "reoon",
  "validationRawStatus": "string"
}
```

Invalid, disposable, spamtrap, and unresolved unknown items are discarded silently inside Stage 4 and never reach Stage 5.

---

## Validation Tool: Reoon (Power Mode)

- **Endpoint:** `GET https://emailverifier.reoon.com/api/v1/verify`
- **Query params:** `email`, `key`, `mode=power`
- **Free tier:** 100 verifications/month — use sparingly during testing

### Status Mapping

| Reoon raw status | Normalized `validationStatus` | Action |
|---|---|---|
| `safe` | `valid` | Send |
| `role_account` | `valid` | Send |
| `catch_all` | `catch-all` | Send cautiously (30% cap enforced) |
| `invalid` | discard | Dropped silently |
| `disabled` | discard | Dropped silently |
| `inbox_full` | discard | Dropped silently |
| `disposable` | discard | Dropped silently |
| `spamtrap` | discard | Dropped silently |
| `unknown` | retry once → discard if still unknown | — |

---

## Validation Rules

| Result | Action |
|---|---|
| Valid | Send |
| Catch-All | Send cautiously — capped at 30% of outgoing batch |
| Invalid / Disabled / Inbox Full / Disposable / Spamtrap | Discard immediately |
| Unknown | Retry once. Still unknown → discard |

---

## The 80% Gate

Before any emails proceed to Stage 5, the workflow checks:

- `sendableCount ÷ requestedLeadCount ≥ 0.80` → proceed to Stage 5
- `sendableCount ÷ requestedLeadCount < 0.80` → STOP, alert operator, do not send

Wired as an IF node (`Quality Gate`) in the Stage 4 sub-workflow. Non-negotiable per SOP.

### Known Limitation — Pipeline Attrition

Leads drop naturally throughout Stages 2 and 3 (social domain filter, Hunter misses, no email found). By Stage 4, `stage4InputCount` may be well below `requestedLeadCount`. This means the gate can trip even when email quality is excellent.

The failure alert email includes two rates to help the operator diagnose the cause:

- **Gate rate** (`sendable ÷ requested`) — the SOP formula
- **Quality-only rate** (`sendable ÷ Stage 4 input`) — informational

If the gate trips due to attrition rather than bad emails, re-run with a higher `leadCount` to compensate for expected pipeline drop-off.

---

## Catch-All Cap Formula

`maxCatchAll = floor(3 × validCount / 7)`

This guarantees `catchAll / (valid + catchAll) ≤ 30%`. If `validCount = 0`, no catch-all emails are allowed through.

---

## Final Node Sequence

```
Workflow Input
      ↓
Validate Email (HTTP Request, per item)
      ↓
Parse Validation Result (Code, per item)
      ↓
Known Results (Code, all items)      Unknown Results (Code, all items)
      ↓                                         ↓
      |                                   Wait 5s - Retry Delay
      |                                         ↓
      |                                   Retry Unknown (HTTP, per item)
      |                                         ↓
      |                                   Parse Retry Result (Code, per item)
      |                                         ↓
      └──────────── Merge Validation Results (Merge, Append)
                             ↓
                   Apply Cap & Build Summary (Code, all items)
                             ↓
                   Quality Gate (IF)
                    True ↓                  False ↓
             Stage 4 Output            Build Fail Alert (Code)
             (Code, all items)               ↓
                                       Send Fail Alert (Gmail)
                                            ↓
                                       Halt Pipeline (Code)
```

---

## Main Workflow Integration

Two nodes added to Main after `Execute Stage 3`:

### Prepare Stage 4 Input (Code, all items)

```javascript
const batchId = `batch_${Date.now()}`;
const webhookData = $('Edit Fields').first().json;

return $input.all().map(item => ({
  json: {
    batchId,
    businessType: webhookData.businessType,
    requestedLeadCount: webhookData.leadCount,
    ...item.json
  }
}));
```

### Execute Stage 4

- Source: Database
- Workflow: `OltaFlock - Stage 4 - Email Validation`
- Mode: Run once with all items

---

## Key Decisions

| Decision | Reason |
|---|---|
| Reoon Power Mode over Quick Mode | Power Mode performs full SMTP verification — higher accuracy, same cost |
| `role_account` mapped to valid | Role accounts (info@, contact@) are sendable inboxes — raw status preserved for audit |
| `unknown` retried once with 5s delay | Transient SMTP timeouts are common — one retry significantly reduces unnecessary discards |
| Catch-all cap requires valid emails first | Sending a batch of 100% catch-all addresses carries high bounce risk — cap set to 0 if no confirmed valid emails |
| Gate measures against `requestedLeadCount` | Per SOP non-negotiable — attrition failure identified via qualityRate in alert email |
| `_summary` stripped at Stage 4 Output | Internal bookkeeping field — should not leak into Stage 5 or downstream logging |
| `Halt Pipeline` Code node throws after `Send Fail Alert` | Without this, Main's `Execute Stage 4` node completes normally (Gmail API response counts as output) and Stage 5 runs with malformed data — throwing here forces Main to stop |
| Stage 4 Error Workflow set to `none` | Prevents `OltaFlock - Error Handler` from double-firing when `Halt Pipeline` throws intentionally |

---

## Acceptance Checklist

- [x] Every email validated before any proceeds to Stage 5
- [x] Unknown retried exactly once — discarded if still unknown after retry
- [x] Catch-all capped at 30% of sendable batch
- [x] 80% gate enforced as IF node — blocks workflow when not met
- [x] Failure alert email distinguishes attrition failure from quality failure
- [x] `validationStatus`, `validationProvider`, `validationRawStatus` present on all Stage 5 items
- [x] `_summary` stripped before Stage 5 receives items
- [x] `batchId`, `businessType`, `requestedLeadCount` preserved through to Stage 5
- [x] Output contract matches Stage 5 input requirements
- [x] Stage 5 cannot execute after gate failure — `Halt Pipeline` node throws, blocking Main from proceeding