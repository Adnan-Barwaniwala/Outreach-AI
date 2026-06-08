# Stage 5: Email Sending — Config & Implementation Record

## Status: Complete

---

## Final Node Sequence

```
Workflow Input
      ↓
Check Daily Limit
      ↓
Split in Batches (size: 1)
   ↓ [loop]                    ↓ [done]
Build Prompt               Stage 5 Output
      ↓
Generate Email (HTTP - Anthropic API)
      ↓
Parse Generated Email
      ↓
Wait — Send Delay
      ↓
Send Gmail
      ↓
Parse Send Result
   ↓ (back to Split in Batches)
```

---

## Workflow Entry

- **Workflow Name:** `Oltflock - Stage 5 - Email Sending`
- **Trigger Node:** `Workflow Input`
- **Trigger Mode:** Accept all incoming data from Main workflow via `Execute Workflow`

---

## AI Email Generation

- **Provider:** Anthropic (Claude)
- **Model:** `claude-haiku-4-5-20251001`
- **Endpoint:** `POST https://api.anthropic.com/v1/messages`
- **Headers:** `x-api-key`, `anthropic-version: 2023-06-01`, `content-type: application/json`
- **Max tokens:** 300
- **Response field:** `content[0].text`

---

## Node Details

### Check Daily Limit
- **Language:** JavaScript | **Mode:** Run Once for All Items
- Uses `$getWorkflowStaticData('global')` to persist send count across multiple workflow runs
- Stores `{ date: 'YYYY-MM-DD', sentToday: number }` — resets automatically at midnight UTC
- If cumulative sends for the day have reached 500 → throws error, halts workflow
- If batch would exceed remaining quota → trims batch to remaining allowed sends
- Records planned send count before sending (conservative — slightly over-counts if sends fail mid-batch)

### Split in Batches
- **Batch Size:** 1
- Loop output → `Build Prompt`
- Done output → `Stage 5 Output`

### Build Prompt
- **Language:** JavaScript | **Mode:** Run Once for Each Item
- Constructs the full Anthropic API request body as a JSON string (`apiBody`)
- Injects `businessName`, `businessType`, `location` into the prompt
- Signs off with sender name "Adnan"
- Passes all original lead fields through unchanged

### Generate Email
- **Node Type:** HTTP Request
- **Body:** `Using JSON` → Expression mode → `{{ $json.apiBody }}`
- Body is pre-serialised in `Build Prompt` to avoid n8n JSON validation issues with dynamic expressions
- Runs inside the loop so Claude API calls are spaced 30–90s apart by the Wait node — prevents hitting the 5 RPM free-tier rate limit

### Parse Generated Email
- **Language:** JavaScript | **Mode:** Run Once for Each Item
- Strips markdown code fences if Claude wraps the response (safety net)
- Parses `subject` and `body` from the JSON response
- Output adds: `subject`, `generatedBody`

### Wait — Send Delay
- **Resume:** After time interval
- **Amount:** `={{ Math.floor(Math.random() * 61) + 30 }}` seconds (30–90s randomised)

### Send Gmail
- **Operation:** Send
- **To:** `{{ $json.email }}`
- **Subject:** `{{ $json.subject }}`
- **Message:** `{{ $json.generatedBody }}`
- **Email Type:** Plain Text
- **On Error:** Continue (not Stop Workflow)
- **Retry on Fail:** Off

### Parse Send Result
- **Language:** JavaScript | **Mode:** Run Once for Each Item
- Recovers lead fields from `$('Split in Batches').item.json` — Gmail node overwrites item context with API response, so original fields must be recovered from the loop node
- Gmail result read from `$input.item.json` (Send Gmail output)
- Delivery status logic:
  - `id` present, no error → `Delivered`, `emailSent: Yes`
  - Error with 5xx SMTP code or delivery keywords (recipient, user unknown, does not exist, etc.) → `Bounced`, `emailSent: Yes`
  - Any other error (auth, timeout, rate limit) → `Undelivered`, `emailSent: No`
  - No status captured → `Unknown`, `emailSent: No`
- Known limitation: delayed NDR bounce emails (Gmail accepts send, bounce arrives days later) cannot be detected synchronously — these appear as `Delivered`
- Connect output back to `Split in Batches` input to close the loop

### Stage 5 Output
- **Language:** JavaScript | **Mode:** Run Once for All Items
- Pass-through — items already carry correct shape from `Parse Send Result`
- Fires from Split in Batches "done" port when all items are processed

---

## Key Decisions

| Decision | Reason |
|---|---|
| `Build Prompt` and `Generate Email` inside the loop | Claude free tier enforces a 5 RPM rate limit — generating all emails before the loop triggers it immediately; running inside the loop spaces calls 30–90s apart via the Wait node |
| Used `Build Prompt` Code node before HTTP Request | n8n's JSON body validator rejects dynamic expressions with newlines — pre-serialising in a Code node bypasses this |
| Claude Haiku over GPT-4o-mini | User preference; comparable cost and quality for simple email generation |
| Strip markdown fences in Parse Generated Email | Claude occasionally wraps JSON in code blocks despite instructions — safety net prevents parse failures |
| Split in Batches (size 1) + Wait for delay | Standard n8n pattern for per-item rate limiting |
| Plain Text email type on Gmail node | Claude generates plain text — HTML mode causes rendering issues |
| Continue on Fail on Gmail node, Retry off | One failure must not crash the batch; retry risks duplicate sends |
| Bounce vs Undelivered split on error message | Gmail delivery failures (5xx SMTP, "user unknown", etc.) → Bounced/emailSent:Yes; system errors (auth, timeout) → Undelivered/emailSent:No |
| Delayed NDR bounces appear as Delivered | Gmail accepts the send synchronously — bounce notification emails arriving days later cannot be captured in real-time; documented known limitation |
| `$getWorkflowStaticData` for daily limit | Persists across sub-workflow runs without needing an external store; scoped to Stage 5 sub-workflow |

---

## Output Contract for Stage 6

```json
{
  "batchId": "string",
  "businessType": "string",
  "businessName": "string",
  "location": "string",
  "email": "string",
  "validationStatus": "valid | catch-all",
  "emailSent": "Yes | No",
  "deliveryStatus": "Delivered | Undelivered | Unknown",
  "dateSent": "ISO timestamp",
  "subject": "string",
  "generatedBody": "string"
}
```

Both successful and failed sends exit Stage 5. Stage 6 logs all outcomes.
