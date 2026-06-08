# Stage 2: Lead Scraping — Config & Implementation Record

## Status: Complete

---

## Final Node Sequence

```
Tally Webhook
      ↓
Edit Fields
      ↓
IF (Stage 1 validation)
  True ↓
Outscraper - Search
      ↓
Count Websites & Normalize
      ↓
Fallback Check (IF)
  False ↓                         True ↓
Split Primary Leads         Outscraper - Fallback
      ↓                                ↓
      └──── Deduplicate ← Combine Primary & Fallback Leads
                  ↓
      Filter - Discard No Contact (Code)
                  ↓
         Cap Lead Count (Limit)
                  ↓
         [Stage 3 placeholder]
```

---

## Outscraper - Search (Primary)

- **Endpoint:** `GET https://api.app.outscraper.com/maps/search-v3`
- **Auth:** n8n native `Outscraper API` credential type
- **Query params:**
  - `query`: `{{ $('Edit Fields').item.json.businessType }} in {{ $('Edit Fields').item.json.location }}`
  - `limit`: `{{ $('Edit Fields').item.json.leadCount }}`
  - `async`: `false`
  - `fields`: `name,website,address,phone`
- **Response structure:** Results nested at `data[0]` (array within array)
- **Confirmed field names:** `website` (not `site`), `address` (not `full_address`). `emails` not available on base tier — removed.

## Outscraper - Fallback

- Same endpoint and auth as primary
- Query uses `near` instead of `in`: `{{ $('Edit Fields').item.json.businessType }} near {{ $('Edit Fields').item.json.location }}`
- Everything else identical

---

## Node Details

### Count Websites & Normalize
- **Language:** Python | **Mode:** Run Once for All Items
- Normalizes raw Outscraper response into `{ businessName, websiteUrl, location, phoneNumber, email, source }`
- Counts leads with non-null `websiteUrl`
- Outputs single item: `{ normalizedLeads: [...], websiteCount: N }`

### Fallback Check (IF)
- `{{ $json.websiteCount }}` less than `{{ Math.floor($('Edit Fields').item.json.leadCount * 0.5) }}`
- True → Outscraper - Fallback
- False → Split Primary Leads

### Split Primary Leads
- **Language:** Python | **Mode:** Run Once for All Items
- Splits `normalizedLeads` array into individual items
- Connects to Deduplicate

### Combine Primary & Fallback Leads
- **Language:** JavaScript | **Mode:** Run Once for All Items
- Single input: `Outscraper - Fallback`
- References `$('Count Websites & Normalize').first().json.normalizedLeads` for primary leads (no direct connection — avoids bypassing IF node)
- Normalizes fallback results, combines with primary, outputs individual items
- Connects to Deduplicate

### Deduplicate
- **Language:** Python | **Mode:** Run Once for All Items
- Dedup key: `websiteUrl` if present, else `businessName|location` (lowercased)
- Connects from both `Split Primary Leads` and `Combine Primary & Fallback Leads`

### Filter - Discard No Contact
- **Language:** Python | **Mode:** Run Once for All Items
- Keeps only leads where `websiteUrl` or `email` is present
- Replaced IF node approach — n8n IF node did not reliably handle `null` values
- Code: `return [item for item in _items if item["json"].get("websiteUrl") or item["json"].get("email")]`

### Cap Lead Count (Limit)
- Max items: `{{ $('Edit Fields').item.json.leadCount }}`
- Keep: First items

---

## Key Decisions

| Decision | Reason |
|---|---|
| Dropped Apify as fallback | Burns credits too fast; second Outscraper call with `near` query is free within quota |
| Removed `emails` field from Outscraper | Not available on base tier — requires paid enrichment. Stage 3 handles email finding. |
| Replaced Fallback Check IF node with Code node for filter | n8n IF node does not reliably evaluate `null` as empty |
| `Combine Primary & Fallback Leads` uses `$()` reference, not direct connection | Direct connection to the gate node bypasses the IF node, breaking fallback logic |
| Proceed with fewer leads if website count << leadCount | SOP: quality over quantity. Stage 4's 80% gate is the real safety net. |

---

## Normalized Lead Shape (Output Contract for Stage 3)

```json
{
  "businessName": "string",
  "websiteUrl": "string | null",
  "location": "string",
  "phoneNumber": "string | null",
  "email": null,
  "source": "outscraper"
}
```

`email` is always `null` at this stage. Stage 3 handles all email extraction.


