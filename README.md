# Outreach-AI - Automated Lead Generation & Cold Outreach Pipeline

A fully automated, end-to-end cold outreach system built in n8n. Submit a single form with a business type, location, and lead count — the pipeline scrapes leads, finds and validates emails, generates a personalised AI email per lead, sends it, and logs every outcome with real-time monitoring. Zero manual work after form submission.

> **Status:** Tested in production. 9/9 emails delivered in initial dental clinic campaign. 80% quality gate successfully blocked a low-quality tech company batch before any sends occurred.

---

## Architecture

7 modular sub-workflows chained by n8n's Execute Workflow node. Each stage has a defined input/output contract and fails independently without taking down the pipeline.

```
Tally Form Submission
        ↓
  Stage 1 — Orchestrator (validates input, chains all stages)
        ↓
  Stage 2 — Lead Scraping (Outscraper · fallback query if coverage < 50%)
        ↓
  Stage 3 — Email Extraction (Hunter.io · pattern fallback · social domain filter)
        ↓
  [Inject batchId + metadata]
        ↓
  Stage 4 — Email Validation (Reoon · 80% quality gate · catch-all cap)
        ↓
  Stage 5 — AI Email Generation + Sending (Claude Haiku · Gmail · rate limiting)
        ↓
  Stage 6 — Logging & Alerts (Google Sheets · bounce monitoring · operator alerts)

  Error Handler — side-channel alert on any unhandled failure in Stage 5 or 6
```

---

## Tech Stack

| Layer | Tool |
|---|---|
| Orchestration | n8n (self-hosted / cloud) |
| Form intake | Tally |
| Lead scraping | Outscraper (Google Maps API) |
| Email finding | Hunter.io domain search |
| Email validation | Reoon (Power Mode — full SMTP verification) |
| AI email generation | Anthropic Claude Haiku (`claude-haiku-4-5`) |
| Email sending | Gmail (OAuth2) |
| Logging | Google Sheets |
| Operator alerts | Gmail |

---

## Pipeline Stages

### Stage 1 — Form Submission & Orchestration
Receives a Tally webhook, validates all three required fields (business type, location, lead count), and chains Stages 2–6 sequentially. A `Prepare Stage 4 Input` code node injects `batchId`, `businessType`, and `requestedLeadCount` into every item before Stage 4 runs — these are needed for the 80% gate calculation.

### Stage 2 — Lead Scraping
Queries the Outscraper Google Maps API. Normalises results into a consistent lead shape. If fewer than 50% of requested leads have websites (low coverage signal), fires a secondary fallback query using `near` instead of `in` and merges both result sets. Deduplicates by website URL (or business name + location as fallback). Discards leads with no website and no email.

### Stage 3 — Email Extraction
Extracts a clean domain from each lead's website URL. Filters out social media domains before any lookup (prevents Hunter.io being queried against `instagram.com`, `facebook.com`, etc.). Calls Hunter.io domain search, sorts candidates by email type (personal first) and confidence score. Leads where Hunter fails fall through to a pattern generator (`info@{domain}`). Both paths merge into a single output — only leads with a non-null email exit Stage 3.

### Stage 4 — Email Validation *(most critical stage)*
Every email — whether found by Hunter or pattern-generated — is verified by Reoon in Power Mode (full SMTP handshake, not syntax-only). Status mapping:

| Reoon result | Action |
|---|---|
| `safe`, `role_account` | Send |
| `catch_all` | Send (30% cap enforced) |
| `invalid`, `disabled`, `disposable`, `spamtrap` | Discard silently |
| `unknown` | Retry once after 5s → discard if still unknown |

**Catch-all cap:** `floor(3 × validCount / 7)` — guarantees catch-alls never exceed 30% of the outgoing batch. If there are zero confirmed valid emails, zero catch-alls are allowed through.

**80% quality gate:** `sendableCount ÷ requestedLeadCount ≥ 0.80`. If it fails, a detailed failure alert is sent to the operator (showing both gate rate and quality-only rate to distinguish attrition from bad emails), then the pipeline halts. Stage 5 never runs on a failed batch. Stage 4's n8n error workflow is deliberately set to `none` to prevent the Error Handler from double-firing on intentional gate failures.

### Stage 5 — AI Email Generation & Sending
Checks a daily send limit (500/day) persisted via `$getWorkflowStaticData` — no external store needed, resets at midnight UTC. Loops through leads one at a time (batch size 1). For each lead, builds a prompt with business name, type, and location, calls Claude Haiku, parses the response, waits a randomised 30–90 second delay, then sends via Gmail. The delay serves two purposes: stays under Claude's free-tier 5 RPM rate limit, and makes sends look human. Both successful and failed sends exit Stage 5 and are logged.

### Stage 6 — Logging & Alerts
Appends every lead to a Google Sheet (8 columns). Computes bounce rate across the full batch after all rows are logged. Over 5% → sends a bounce alert to the operator. Under 5% → sends a campaign completion summary. Reads from `Workflow Input` (not the Sheets node output) to avoid field name conflicts caused by Google Sheets renaming camelCase fields to spaced column headers.

### Error Handler
Three-node side-channel workflow: Error Trigger → Build Error Alert → Send Error Alert. Fires automatically on any unhandled error in Stage 5 or 6. Alert includes workflow name, failing node, error message, execution ID, and ISO timestamp.

---

## Key Design Decisions

| Decision | Why |
|---|---|
| 80% gate blocks send rather than degrades it | Blasting a low-quality batch damages sender reputation — better to alert and re-run |
| Catch-all cap at 30% | Unverifiable addresses carry real bounce risk; cap enforced mathematically not heuristically |
| Fallback scraping query (`near` vs `in`) | Second Outscraper call is free within quota; cheaper than Apify and avoids credit burn |
| Social domain filter before Hunter.io | Prevents Hunter being queried against `instagram.com` from a business that listed its social as a website |
| Pattern fallback: `info@{domain}` only | Generating multiple guesses per domain multiplies validation cost without proportional gain |
| `unknown` retried once with 5s delay | Transient SMTP timeouts are common — one retry handles the majority without over-spending |
| AI calls inside the loop, not before it | Generating all emails before the loop would hit Claude's 5 RPM free-tier limit immediately |
| Error Handler excluded from Stage 4 | `Halt Pipeline` throws intentionally after gate failure — linking would create false alarms |
| `$getWorkflowStaticData` for daily limit | Persists across runs without an external store; scoped to Stage 5 sub-workflow |
| Stage 6 reads `Workflow Input` for bounce rate | Google Sheets Append Row output renames camelCase fields — reading from Sheets would break field access |

---

## Sample Output

See [`samples/sample_emails.md`](samples/sample_emails.md) for:
- 9 AI-generated personalised emails from a live dental clinic campaign in Austin, TX
- A real 80% gate failure alert (tech companies batch, 40% sendable rate)
- A real bounce rate alert email
- A real campaign completion summary email

All emails were generated uniquely per business — no template was used.

---

## Known Limitations

See [`samples/known_limitations.md`](samples/known_limitations.md) for full details. Summary:

1. **Async bounce emails** — Gmail sometimes accepts sends that later bounce (NDR arrives days later). These log as `Delivered`. Reoon validation reduces but cannot eliminate this.
2. **80% gate can trip due to pipeline attrition** — leads drop in Stages 2 and 3 naturally (social filter, Hunter misses). The gate measures against the original requested count, not Stage 4 input count. Re-run with a higher `leadCount` to compensate.
3. **Transient SMTP timeouts** — `unknown` results retried once, then discarded. In rare cases a valid address may be dropped due to a temporarily unreachable mail server.
4. **n8n free plan footer** — Emails sent on the free tier include an n8n branding footer visible to recipients. Resolved by upgrading to a paid n8n plan.

---

## Setup

### Prerequisites
- n8n instance (cloud or self-hosted)
- Accounts + API keys for: Outscraper, Hunter.io, Reoon, Anthropic
- Gmail account with OAuth2 configured in n8n
- Google Sheets with headers set up per Stage 6 config
- Tally form connected to the Stage 1 webhook URL

### Import
1. Import each JSON file from `workflows/` into n8n in numerical order
2. Configure credentials for each service in n8n's credential manager
3. Update the webhook URL in the Tally form settings to match your n8n instance
4. Activate the Error Handler workflow first (it must be active to catch errors from Stage 5/6)
5. Activate Stage 1 last

### Google Sheet Setup
Create a sheet with these headers in row 1 before the first run:

| A | B | C | D | E | F | G | H |
|---|---|---|---|---|---|---|---|
| Business Name | Location | Business Type | Email Address | Validation Status | Email Sent | Delivery Status | Date Sent |

---

## Docs

Detailed per-stage implementation records, node-level configs, and design decision logs are in [`/docs`](docs/).
