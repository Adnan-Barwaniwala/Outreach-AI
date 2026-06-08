# Stage 6: Logging & Alerts — Config & Implementation Record

## Status: Complete

---

## Final Node Sequence

```
Workflow Input
      ↓
Append to Google Sheets (per item)
      ↓
Compute Bounce Rate (Code, all items)
      ↓
Bounce Rate Check (IF)
   ↓ [true > 5%]              ↓ [false ≤ 5%]
Build Bounce Alert      Build Completion Summary
   ↓                          ↓
Send Bounce Alert        Send Completion Summary
```

---

## Workflow Entry

- **Workflow Name:** `Oltaflock - Stage 6 - Logging & Alerts`
- **Trigger Node:** `Workflow Input`
- **Trigger Mode:** Accept all incoming data from Main workflow via `Execute Workflow`

---

## Node Details

### Append to Google Sheets
- **Node Type:** Google Sheets
- **Operation:** Append Row
- **Authentication:** OAuth2
- **Run mode:** Once for Each Item
- **On Error:** Continue
- **Column mapping:**

| Sheet Column | Expression |
|---|---|
| Business Name | `{{ $json.businessName }}` |
| Location | `{{ $json.location }}` |
| Business Type | `{{ $json.businessType }}` |
| Email Address | `{{ $json.email }}` |
| Validation Status | `{{ $json.validationStatus }}` |
| Email Sent | `{{ $json.emailSent }}` |
| Delivery Status | `{{ $json.deliveryStatus }}` |
| Date Sent | `{{ $json.dateSent }}` |

### Compute Bounce Rate
- **Language:** JavaScript | **Mode:** Run Once for All Items
- Reads from `$('Workflow Input').all()` — not from Sheets node output (Sheets output uses column header names with spaces, not camelCase field names)
- Computes: `bounceRate`, `deliveryRate`, and all counts
- Outputs a single summary item

### Bounce Rate Check
- **Node Type:** IF
- **Condition:** `{{ $json.bounceRate }}` > `0.05` (Number)
- True → `Build Bounce Alert`
- False → `Build Completion Summary`

### Build Bounce Alert / Build Completion Summary
- **Language:** JavaScript | **Mode:** Run Once for All Items
- Uses `$input.first().json` to access the single summary item from Compute Bounce Rate
- Outputs: `subject`, `body`

### Send Bounce Alert / Send Completion Summary
- **Node Type:** Gmail
- **Operation:** Send
- **To:** Operator email (hardcoded)
- **Subject:** `{{ $json.subject }}`
- **Message:** `{{ $json.body }}`
- **Email Type:** Plain Text
- **On Error:** Continue

---

## Key Decisions

| Decision | Reason |
|---|---|
| `Compute Bounce Rate` reads from `$('Workflow Input').all()` | Google Sheets Append Row output uses column header names with spaces — camelCase field checks would fail |
| `Build Bounce/Summary` uses `$input.first().json` | Compute Bounce Rate outputs a single aggregated item — `first()` is reliable; `item.json` fails in all-items mode |
| Sheets node On Error: Continue | A logging failure on one row must not stop the rest of the batch from being logged |
| `dateSent` mapped from `$json.dateSent` | Must reflect when the email was sent (set in Stage 5), not when the Sheets node ran |

---

## Pre-Requisite: Google Sheet Setup

Sheet must exist with these headers in row 1 before first run:

| A | B | C | D | E | F | G | H |
|---|---|---|---|---|---|---|---|
| Business Name | Location | Business Type | Email Address | Validation Status | Email Sent | Delivery Status | Date Sent |

---

## Output Contract for Stage 6

No data output — Stage 6 is terminal. All outcomes written to Google Sheets and operator notified via Gmail.

---

## Acceptance Checklist

- [x] Google Sheet appends one row per lead automatically — zero manual entry
- [x] All 8 required columns present and populated correctly
- [x] `dateSent` populated from Stage 5 — not from the logging node timestamp
- [x] Bounce rate computed across the full batch after all rows are logged
- [x] Bounce rate > 5% triggers an alert email to operator
- [x] Campaign completion summary sent to operator on every successful run
- [x] Both delivered and undelivered sends are logged with correct status
- [x] No item from Stage 5 is dropped before logging
