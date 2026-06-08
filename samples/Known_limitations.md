# Known Limitations

---

## 1. Async Bounce Emails Logged as Delivered

**Affects:** Delivery status logging · Bounce rate alert

**What happens**
When an email is sent to a non-existent address, Gmail sometimes accepts the message immediately and returns a success response. The bounce notification (NDR) only arrives hours or days later, in a separate email — not as part of the original send response. The pipeline only reads the immediate response, so these cases are logged as `Delivered` instead of `Bounced`. The bounce rate alert may therefore underreport the true bounce rate.

**Why it happens**
Some receiving mail servers accept all incoming mail first and only check whether the specific inbox exists afterwards. This is a fundamental behaviour of SMTP — there is no way to detect it synchronously at send time.

**How to mitigate**
Monitor the Gmail sent account for incoming NDR emails. Cross-reference against the Google Sheet periodically. Reoon's validation step significantly reduces this risk by filtering out addresses likely to bounce before they are ever sent to.

---

## 2. The 80% Gate Can Trip Due to Pipeline Attrition

**Affects:** 80% quality gate (Stage 4)

**What happens**
The gate checks: `emails validated ÷ leads requested ≥ 80%`. Leads naturally drop throughout Stages 2 and 3 — social media URLs are filtered out, Hunter.io may not find an email for every domain, and some businesses have no contactable email at all. By the time leads reach Stage 4, the count may already be well below the original requested number, causing the gate to trip even when every email that was found is perfectly valid.

**Why it happens**
The SOP formula measures against `requested count`, not `Stage 4 input count`. This is intentional — it holds the pipeline accountable to the original request, not just what survived extraction.

**How to resolve**
If the gate trips and the failure alert email shows a high quality rate alongside a low gate rate, the cause is attrition rather than bad emails. Re-run the campaign with a higher lead count to compensate for expected drop-off. The alert email includes both rates to help diagnose which case applies.

---

## 3. Valid Emails May Be Discarded Due to Transient SMTP Timeouts

**Affects:** Email validation (Stage 4) · Final lead count

**What happens**
When Reoon cannot reach a mail server during verification, it returns a status of `unknown`. The pipeline retries once after a 5-second delay. If the result is still `unknown`, the email is discarded and the lead does not proceed to Stage 5. In rare cases, this may discard a genuinely valid address whose mail server was temporarily unreachable.

**Why it happens**
Some mail servers are slow, rate-limited, or briefly offline during the verification window. SMTP timeouts are common and not always indicative of an invalid address.

**How to mitigate**
The single retry with a delay already handles the majority of transient cases. Persistent `unknown` results are discarded by design — sending to unverified addresses risks deliverability. If a specific lead is known to be valid, it can be manually added to a future batch.

---

## 4. Emails Include an "n8n" Footer on the Free Plan

**Affects:** Email tone · SOP requirement: no automation give-away to recipients

**What happens**
On the n8n free tier, all emails sent via the Gmail node include a small footer branding the message as sent through n8n. This is visible to the recipient and contradicts the SOP requirement that emails feel human — no mention of AI or automation tools.

**Why it happens**
n8n adds this footer automatically on free tier accounts. It cannot be removed or overridden within the workflow itself.

**How to resolve**
Upgrade to an n8n paid plan. The footer is removed entirely on paid tiers. No changes to the workflow are needed — the upgrade alone resolves it.
