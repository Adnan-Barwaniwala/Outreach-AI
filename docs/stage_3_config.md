# Stage 3: Email Extraction — Config & Implementation Record

## Status: Complete

---

## Final Node Sequence

```
Workflow Input
      ↓
Extract Domain
      ↓
Filter Social Domains
      ↓
Hunter.io - Domain Search
      ↓
Parse Hunter.io Results
      ↓
Hunter Found (Code)               Hunter Not Found (Code)
          ↓                               ↓
                                      Pattern Generator
                                              ↓
                    Hunter Found ───────→ Merge Emails ←────── Pattern Generator
                                              ↓
                                       Stage 3 Output
```

---

## Workflow Entry

- **Workflow Name:** `OltaFlock - Stage 3 - Email Extraction`
- **Trigger Node:** `Workflow Input`
- **Trigger Mode:** Accept all incoming data from Main workflow via `Execute Workflow`
- **Main Workflow Integration:** Main passes Stage 2 output items into Stage 3 using `Execute Workflow`

---

## Hunter.io - Domain Search

- **Endpoint:** `GET https://api.hunter.io/v2/domain-search`
- **Auth:** API key passed in query params or stored in n8n credentials
- **Query params:**
  - `domain`: `{{ $json.domain }}`
  - `api_key`: Hunter API key
  - `limit`: `5`
- **Run mode:** Once per item
- **Response structure:** Hunter emails returned at `data.emails[]`

---

## Node Details

### Extract Domain
- **Language:** JavaScript | **Mode:** Run Once for Each Item
- Extracts a clean domain from `websiteUrl`
- Normalization includes:
  - strips `http://` or `https://`
  - strips leading `www.`
  - removes path, query string, fragment, and port
  - lowercases final domain
- Output adds:
  - `domain`

### Filter Social Domains
- **Language:** JavaScript | **Mode:** Run Once for All Items
- Removes leads where extracted `domain` is a known social/media platform instead of the business website
- Current blocked set includes:
  - `instagram.com`
  - `facebook.com`
  - `fb.com`
  - `twitter.com`
  - `x.com`
  - `linkedin.com`
  - `youtube.com`
  - `tiktok.com`
- Purpose:
  - prevents Hunter lookup against social profile domains
  - prevents pattern guessing like `info@instagram.com`

### Parse Hunter.io Results
- **Language:** JavaScript | **Mode:** Run Once for Each Item
- Recovers original lead fields from `Extract Domain`
- Reads `data.emails[]` from Hunter response
- Sorts candidates:
  - personal emails before generic emails
  - higher confidence first
- Outputs best match only
- Output shape includes:
  - `businessName`
  - `websiteUrl`
  - `location`
  - `phoneNumber`
  - `source`
  - `domain`
  - `email`
  - `emailSource`

### Hunter Found
- **Language:** JavaScript | **Mode:** Run Once for All Items
- Filters items where `email` is present and non-empty
- Sends successful Hunter results to merge/output path

### Hunter Not Found
- **Language:** JavaScript | **Mode:** Run Once for All Items
- Filters items where `email` is null, missing, or empty
- Sends only unresolved leads to pattern fallback

### Pattern Generator
- **Language:** JavaScript | **Mode:** Run Once for Each Item
- Generates a single fallback email guess using:
  - `info@{domain}`
- Output shape:
  - `businessName`
  - `websiteUrl`
  - `location`
  - `phoneNumber`
  - `source`
  - `email`
  - `emailSource: "pattern"`
- No multiple guesses are generated
- `firstname@domain` and `contact@domain` were deliberately not added to avoid noisy validation spend

### Merge Emails
- **Node Type:** Merge
- **Mode:** Append
- Input 1:
  - `Hunter Found`
- Input 2:
  - `Pattern Generator`
- Required because direct multi-input into `Stage 3 Output` caused separate executions rather than one combined batch

### Stage 3 Output
- **Language:** JavaScript | **Mode:** Run Once for All Items
- Normalizes final Stage 3 output shape for Stage 4
- Removes intermediate field:
  - `domain`
- Returns one combined list of leads with emails

---

## Key Decisions

| Decision | Reason |
|---|---|
| Dropped Snov.io from the waterfall | Free plan does not expose usable email results for this workflow |
| Used Code nodes instead of IF nodes for found/not-found splits | n8n empty/null checks in IF nodes were unreliable in practice |
| Added social-domain filter before Hunter | Social URLs from Stage 2 were being treated as business websites, producing useless domains |
| Kept a single pattern fallback: `info@domain` | Lowest-cost, lowest-complexity generic fallback; avoids generating excess emails for validation |
| Used Merge node in `Append` mode before final output | Direct multi-input into `Stage 3 Output` executed separately per branch instead of combining them |
| Removed `domain` from final output | `domain` is only needed internally for lookup and guessing; Stage 4 contract does not require it |

---

## Output Contract for Stage 4

```json
{
  "businessName": "string",
  "websiteUrl": "string",
  "location": "string",
  "phoneNumber": "string | null",
  "email": "string",
  "emailSource": "hunter | pattern",
  "source": "outscraper"
}
```

Only leads with a non-null email leave Stage 3.

