# Shopify Abandoned Checkout Recovery

An n8n workflow that intercepts Shopify `checkout/update` webhook events, deduplicates against Airtable, enriches customer data, generates a personalized recovery email with **GPT-4o**, evaluates its persuasiveness with **GPT-4o-mini**, rewrites it if the score falls below 7, and fires the final message to **Klaviyo** for delivery — with a full audit trail in Airtable.

---

## The problem this solves

Abandoned carts are one of the highest-ROI recovery opportunities in e-commerce, but generic recovery emails underperform. This workflow replaces template-based emails with AI-generated, customer-aware messages that reference the specific items in the cart, the total, any available discount code, and the customer's purchase history — then self-evaluates and rewrites until the message meets a persuasiveness threshold before sending.

---

## What it does

1. Receives a Shopify `checkout/update` webhook when a cart is abandoned
2. Responds to Shopify immediately to prevent webhook retries
3. Checks Airtable to avoid re-processing the same checkout twice
4. Logs the new checkout to Airtable with status `PROCESSING`
5. Enriches the customer profile with order history and spend data from Shopify REST API
6. Sends the full context to **GPT-4o** to generate a warm, personalized recovery message
7. Sends the message to **GPT-4o-mini** for persuasiveness evaluation (score 1–10)
8. If score ≥ 7 → sends to Klaviyo and marks as `SENT_TO_KLAVIYO`
9. If score < 7 and attempts < 3 → rewrites with the AI's own improvement suggestion, re-evaluates
10. After 3 attempts → sends the best version regardless of score

---

## Workflow

**File:** `n8n-Shopify-Abandoned-Checkout-Recovery.json`

### Flow diagram

```
Shopify Checkout Update Webhook
  --> Respond to Shopify (immediately — avoids 5s timeout)
  --> Check Airtable — Already Messaged?
  --> Is New Checkout?
  --> Log New Checkout to Airtable          (status: PROCESSING)
  --> Enrich Customer Data (Shopify REST)
  --> Merge Checkout + Customer Data
  --> GPT-4o: Write Recovery Message
  --> Capture Generated Message
  --> GPT-4o-mini: Evaluate Persuasiveness  (score 1–10)
  --> Parse Evaluation & Check Threshold
  --> Update Airtable — Suggestion & Attempt Count
         |
         |-- score >= 7 OR attempts = 3 ---------> Merge Final Message
         |                                              --> Send to Klaviyo
         |                                              --> Update Airtable (SENT_TO_KLAVIYO)
         |
         |-- score < 7 AND attempts < 3 -------> GPT-4o: Rewrite with Feedback
                                                       --> Update State for Re-Evaluation
                                                       --> GPT-4o-mini: Re-Evaluate
                                                       --> Parse Re-Evaluation
                                                       --> Update Airtable — Retry Attempt
                                                       --> Merge Final Message
                                                              --> Send to Klaviyo
                                                              --> Update Airtable (SENT_TO_KLAVIYO)
```

### Node-by-node breakdown

#### 1. Shopify Checkout Update Webhook (`webhook-trigger`)
Webhook trigger that listens for POST requests from Shopify on the path `shopify-checkout-update`. Shopify sends a `checkout/update` event when a cart is abandoned (typically after 1 hour of inactivity with an email captured).

Configure this URL in Shopify: **Settings → Notifications → Webhooks → `checkouts/update`**

#### 2. Extract Checkout Payload
Parses the webhook body from Shopify to extract checkout fields: ID, customer email, name, line items, total price, currency, discount codes, and recovery URL.

> The webhook payload already contains all checkout data. No separate API call is needed at this stage.

#### 3. Check Airtable — Already Messaged? (`check-airtable-dedup`)
Searches the `abandonedCheckouts` Airtable table for an existing record matching `checkout_id`. If found, the checkout has already been processed and the flow stops.

#### 4. Is New Checkout? (`is-new-checkout`)
Passes through only checkouts not already in Airtable. If Airtable returns 0 records, the checkout is new and continues downstream.

#### 5. Log New Checkout to Airtable (`log-to-airtable`)
Creates a new Airtable record with status `PROCESSING` before any AI processing begins. Writing to Airtable first (before GPT calls) ensures that if any downstream step fails, the checkout won't be re-processed on the next webhook delivery.

Fields written: `checkout_id`, `checkout_name`, `customer_email`, `customer_first_name`, `customer_last_name`, `total_price`, `currency`, `abandoned_url`, `line_items`, `discount_codes`, `status`, `attempt_count`, `created_at`, `logged_at`

#### 6. Enrich Customer Data — Shopify REST (`enrich-customer`)
Calls the Shopify REST API (`/admin/api/2024-01/customers/search.json`) to fetch the customer's purchase history: `orders_count`, `total_spent`, and `tags`. This data informs the GPT prompt — a returning customer with 5+ orders gets a different tone than a first-time visitor.

#### 7. Merge Checkout + Customer Data (`merge-data`)
Combines the checkout fields, the Airtable record ID (needed for updates), and the enriched customer data into a single item that flows through the rest of the pipeline.

#### 8. GPT-4o: Write Recovery Message (`generate-message`)
Sends the full customer and cart context to `gpt-4o` with a system prompt configured for a beauty brand advisor persona. The message is written as a single warm paragraph — no excessive emojis, no hard-sell tactics.

**Prompt inputs:** customer name, cart items with quantities, total price, currency, discount codes, recovery URL, previous order count, and (on retry runs) the previous improvement suggestion.

**Temperature:** `0.7` — allows creative variation between attempts.

#### 9. Capture Generated Message (`capture-message`)
Extracts `message.content` from the OpenAI response and increments `attemptCount`. Passes all fields forward for evaluation.

#### 10. GPT-4o-mini: Evaluate Persuasiveness (`evaluate-message`)
Sends the generated message to `gpt-4o-mini` for quality evaluation. Returns a score (1–10) and a specific, actionable improvement suggestion.

**Evaluation criteria:** emotional appeal, clarity, urgency, personalization, and CTA strength.

**Temperature:** `0.3` — keeps scoring consistent and repeatable.

**Output format:**
```json
{
  "score": 8,
  "suggestion": "Add a specific time-limited urgency element to strengthen the CTA."
}
```

#### 11. Parse Evaluation & Check Threshold (`parse-eval`)
Safely parses the GPT evaluation JSON. Falls back to `score: 7` on parse failure so the workflow always continues. Computes:

- `passesThreshold`: `score >= 7`
- `hitMaxAttempts`: `attemptCount >= 3`
- `shouldSend`: `passesThreshold || hitMaxAttempts`

#### 12. Update Airtable — Suggestion & Attempt Count (`update-airtable-loop`)
Updates the Airtable record with the current attempt number, persuasion score, and improvement suggestion before routing.

#### 13. Route — Score >=7 or Max Attempts? / Score <7 Retry?
Two filter nodes route the flow:

| Condition | Path |
|---|---|
| `shouldSend === true` (score >=7 or 3 attempts) | Merge Final Message → Klaviyo |
| `shouldSend === false` (score <7, attempts <3) | GPT-4o: Rewrite with Feedback |

#### 14. GPT-4o: Rewrite with Feedback (`retry-message`)
Rewrites the message incorporating the evaluator's specific suggestion. The original message and all checkout context are included so the rewrite has full context.

#### 15. Update State for Re-Evaluation (`update-for-retry`)
Increments `attemptCount`, stores the new message as `generatedMessage`, and saves the previous suggestion as `lastSuggestion` for reference.

#### 16. GPT-4o-mini: Re-Evaluate Rewritten Message (`re-evaluate`)
Same evaluation prompt as step 10, applied to the rewritten message.

#### 17. Parse Re-Evaluation (`parse-re-eval`)
Same parsing logic as step 11. After 3 total attempts, `hitMaxAttempts` becomes true and `shouldSend` is forced to true — the best version available is sent regardless of score.

#### 18. Update Airtable — Retry Attempt (`update-airtable-retry`)
Updates the Airtable record with the retry attempt data.

#### 19. Merge Final Message (`merge-final`)
Consolidates the final message data from whichever path (first-pass or retry) reached this point, before sending to Klaviyo.

#### 20. Send to Klaviyo — AI Recovery Email Sent (`trigger-klaviyo`)
Fires a custom `AI Recovery Email Sent` event to Klaviyo via the Events API (revision `2024-02-15`). The event carries the full message, score, checkout data, and recovery URL as properties. Klaviyo's flow engine uses this event to trigger the actual email send via an automated flow configured in Klaviyo.

**Event properties sent:** `email_message`, `persuasion_score`, `checkout_id`, `total_price`, `currency`, `line_items`, `recovery_url`, `attempt_count`

#### 21. Update Airtable — Final Status (`update-airtable-final`)
Marks the Airtable record as `SENT_TO_KLAVIYO` and records the final message, score, attempt count, and timestamp.

---

## AI feedback loop logic

```
Attempt 1
  |- Score >= 7  --> Send
  |- Score < 7   --> Rewrite with evaluator suggestion

      Attempt 2
        |- Score >= 7  --> Send
        |- Score < 7   --> Rewrite again

            Attempt 3
              --> Send regardless of score
```

The evaluator's `suggestion` field feeds directly into the next rewrite prompt, creating a closed self-improvement loop. Each rewrite is a full regeneration (not a patch), with the original message provided as reference.

---

## Airtable table structure

**Table name:** `abandonedCheckouts`

| Column | Type | Description |
|---|---|---|
| `checkout_id` | Single line text | Shopify checkout ID (dedup key) |
| `checkout_name` | Single line text | Shopify checkout name (e.g. `#1234`) |
| `customer_email` | Email | Customer email address |
| `customer_first_name` | Single line text | First name |
| `customer_last_name` | Single line text | Last name |
| `total_price` | Number | Cart total value |
| `currency` | Single line text | Currency code (e.g. `USD`) |
| `abandoned_url` | URL | Shopify recovery link |
| `line_items` | Long text | Cart summary (e.g. `Serum Plus x2, Toner x1`) |
| `discount_codes` | Single line text | Comma-separated discount codes |
| `status` | Single select | `PROCESSING` → `SENT_TO_KLAVIYO` |
| `attempt_count` | Number | Number of generation attempts (1–3) |
| `last_suggestion` | Long text | GPT evaluator's improvement suggestion |
| `last_score` | Number | Most recent persuasion score |
| `final_message` | Long text | The message sent to Klaviyo |
| `final_score` | Number | Final persuasion score at send time |
| `final_attempt_count` | Number | Which attempt was ultimately sent |
| `created_at` | Date | Shopify checkout creation timestamp |
| `logged_at` | Date | When the workflow first processed it |
| `sent_at` | Date | When the Klaviyo event was fired |

---

## Klaviyo setup

The workflow fires a **custom event** (`AI Recovery Email Sent`) — it does not send the email directly. The email delivery is handled by a **Klaviyo Flow**:

1. Klaviyo → **Flows** → Create Flow → trigger: **Metric** → select `AI Recovery Email Sent`
2. Add an Email block — use `{{ event.email_message }}` as the email body
3. Additional available properties: `{{ event.persuasion_score }}`, `{{ event.recovery_url }}`, `{{ event.line_items }}`
4. Set flow filters if needed (e.g. only send if `persuasion_score >= 6`)

---

## Setup checklist

### n8n Variables required

Set these in n8n → **Settings → Variables**:

| Variable | Value |
|---|---|
| `SHOPIFY_STORE_DOMAIN` | `your-store.myshopify.com` (no `https://`) |
| `SHOPIFY_ACCESS_TOKEN` | Shopify Admin API access token |
| `KLAVIYO_PRIVATE_KEY` | Klaviyo private API key |

### Credentials required

| Service | Type | Used in |
|---|---|---|
| Shopify | HTTP Header Auth (`X-Shopify-Access-Token`) | Customer enrichment API |
| Airtable | Personal Access Token | All Airtable nodes |
| OpenAI | API Key | GPT-4o and GPT-4o-mini nodes |

### Placeholders to replace

| Placeholder | Node | What to set |
|---|---|---|
| `YOUR_AIRTABLE_BASE_ID` | All Airtable nodes | Base ID from Airtable URL |
| `YOUR_AIRTABLE_CREDENTIAL_ID` | All Airtable nodes | n8n Airtable credential ID |
| `YOUR_SHOPIFY_CREDENTIAL_ID` | Shopify HTTP nodes | n8n HTTP Header Auth credential ID |
| `YOUR_OPENAI_CREDENTIAL_ID` | All OpenAI nodes | n8n OpenAI credential ID |
| `YOUR_ERROR_WORKFLOW_ID` | Workflow settings | n8n error-handling workflow ID |

### Shopify webhook configuration

1. Shopify Admin → **Settings → Notifications → Webhooks**
2. Create webhook: Event `checkouts/update`, Format `JSON`
3. URL: `https://YOUR_N8N_INSTANCE/webhook/shopify-checkout-update`
4. Shopify API version: `2024-01`

### Step-by-step activation

1. Create the Airtable base and table with all columns listed above
2. Set up credentials in n8n: Shopify Header Auth, Airtable, OpenAI
3. Set the three n8n Variables (`SHOPIFY_STORE_DOMAIN`, `SHOPIFY_ACCESS_TOKEN`, `KLAVIYO_PRIVATE_KEY`)
4. Import `n8n-Shopify-Abandoned-Checkout-Recovery.json`
5. Replace all placeholder IDs
6. Activate the workflow and copy the webhook URL
7. Register the webhook URL in Shopify
8. Create the Klaviyo Flow triggered by the `AI Recovery Email Sent` metric
9. Test with a real abandoned cart or use Shopify's webhook tester

---

## Error handling

- **Dedup safety:** Airtable is written *before* GPT calls. If OpenAI fails mid-flow, the checkout is already logged — the next webhook delivery (Shopify retries failed webhooks up to 19 times over 48 hours) will find the record and stop, preventing duplicate sends.
- **AI parse failures:** Both evaluation parse nodes fall back to `score: 7` on JSON parse error — the workflow always continues and a message is always sent.
- **Max attempts safety net:** After 3 attempts, `shouldSend` is forced true regardless of score — no checkout is silently dropped.
- **Error workflow:** The `errorWorkflow` setting routes unhandled node failures to a dedicated handler. Configure it to send a Slack or email alert with the execution ID.

---

## Known bugs — fix before production

| # | Severity | Node | Issue |
|---|---|---|---|
| 1 | Critical | Fetch Abandoned Checkouts | `abandonedCheckouts` does not exist in Shopify GraphQL Admin API — replace with REST endpoint or use webhook body directly |
| 2 | Critical | (architecture) | Webhook payload is completely ignored — Shopify sends all checkout data in the webhook body; the separate GraphQL call is unnecessary and broken |
| 3 | Critical | Is New Checkout? | Filter reads `$json.records.length` which does not exist in n8n Airtable v2.1 output — dedup never works correctly |
| 4 | Critical | Enrich Customer Data | Uses `$('Extract & Format Checkouts').first()` instead of `.item` — always queries the first checkout's email when multiple checkouts are processed |
| 5 | Critical | Update Airtable connections | Two-output connection format on a single-output node — `Score <7 Retry?` never receives items; retry path is unreachable |
| 6 | Critical | Send to Klaviyo | `generatedMessage` embedded in raw JSON string without escaping — any AI message with quotes or newlines produces invalid JSON |
| 7 | Major | Merge Final Message | `try/catch` on `$('nodeName').first()` never throws — always returns first-attempt data even when the retry path ran |
| 8 | Major | Webhook trigger | `responseMode: lastNode` waits 20–40s before responding to Shopify — exceeds Shopify's 5s timeout causing webhook retries and duplicate processing |

---

## Recommended future additions

### Recovery window check
Before generating the message, verify the checkout `createdAt` timestamp is within the recovery window (e.g. 1–24 hours old). Checkouts older than 24 hours have significantly lower recovery rates and could be excluded or routed to a discount-heavy fallback.

### Dynamic discount code generation
For high-value carts (e.g. total > $150), auto-generate a one-time discount code via the Shopify Admin API (`POST /admin/api/2024-01/price_rules/{id}/discount_codes.json`) and include it in the GPT prompt — incentive scaled to cart value.

### Multi-touch sequence
Extend to a 3-touch sequence: immediate message, 24h follow-up, 48h final offer — each tracked in Airtable with its own `status` and `sent_at` field. The Klaviyo flow can handle the scheduling once each event is fired with the correct touch number.

### Recovery confirmation loop
Add a second n8n workflow that listens for Shopify `orders/create` events, checks if the order corresponds to a previously abandoned checkout, and updates the Airtable record to `RECOVERED` — closing the loop on actual conversion tracking and ROI measurement.
