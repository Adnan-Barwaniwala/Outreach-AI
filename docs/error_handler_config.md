# Error Handler — Config & Implementation Record

## Status: Complete

---

## Purpose

Catches unhandled errors from any OltaFlock sub-workflow and notifies the operator with full diagnostic details. Prevents silent failures going unnoticed.

---

## Workflow Name

`OltaFlock - Error Handler`

---

## Final Node Sequence

```
Error Trigger (n8n built-in)
      ↓
Build Error Alert (Code)
      ↓
Send Error Alert (Gmail)
```

---

## Node Details

### Error Trigger
- **Node Type:** n8n built-in Error Trigger
- Fires automatically when a linked workflow encounters an unhandled error during a **production** execution
- Does NOT fire on manual "Test workflow" runs — must be triggered via `Execute Workflow` from Main

### Build Error Alert
- **Language:** JavaScript | **Mode:** Run Once for All Items
- Reads error context from `$input.first().json`
- Outputs: `subject`, `body`

```javascript
const err = $input.first().json;
return [{
  json: {
    subject: `WORKFLOW ERROR — ${err.workflow?.name ?? 'Unknown Workflow'}`,
    body: `OltaFlock Error Alert\n\nWorkflow: ${err.workflow?.name}\nNode: ${err.execution?.lastNodeExecuted}\nError: ${err.execution?.error?.message}\n\nExecution ID: ${err.execution?.id}\nTime: ${new Date().toISOString()}\n\nCheck n8n for full details.`
  }
}];
```

### Send Error Alert
- **Node Type:** Gmail
- **Operation:** Send
- **To:** Operator email (hardcoded)
- **Subject:** `{{ $json.subject }}`
- **Message:** `{{ $json.body }}`
- **Email Type:** Plain Text

---

## Linked Workflows

| Workflow | Settings → Error Workflow |
|---|---|
| OltaFlock - Stage 4 - Email Validation | `none` (intentional — Halt Pipeline throws deliberately; double-alert suppressed) |
| Oltflock - Stage 5 - Email Sending | `OltaFlock - Error Handler` |
| Oltaflock - Stage 6 - Logging & Alerts | `OltaFlock - Error Handler` |

---

## Key Decisions

| Decision | Reason |
|---|---|
| Stage 4 Error Workflow set to `none` | `Halt Pipeline` throws intentionally after gate failure — linking Error Handler here would send a false alarm on every gate failure |
| Reads `$input.first().json` not `$input.item.json` | Error Trigger outputs a single item; `first()` is reliable in all-items mode |
| Production-only trigger | n8n Error Workflows do not fire on manual test runs — test by triggering Stage 5 or Stage 6 from Main |

---

## Acceptance Checklist

- [x] Error Trigger wired as the trigger node
- [x] Alert email includes: workflow name, failing node, error message, execution ID, timestamp
- [x] Linked to Stage 5 and Stage 6
- [x] Stage 4 deliberately excluded to avoid double-alerting on intentional gate failures
- [x] Confirmed working — error alert received during production test run
