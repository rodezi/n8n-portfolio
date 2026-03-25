# Email Triage & AI Classifier

An n8n workflow that monitors a Gmail inbox every 5 minutes, uses GPT-4o-mini to classify and summarize each new email, applies category labels in Gmail, logs everything to Google Sheets, and routes only actionable emails to the right people via Slack — with AI-drafted replies saved automatically for critical messages.

---

## The problem this solves

A busy inbox where every email looks the same — client emergencies, newsletters, HR notices, spam, and sales opportunities all land in the same place. Someone (usually the manager) has to manually read and sort everything before any routing or response can happen. This workflow automates that entire first-pass triage layer so only emails that truly need human attention surface, already pre-classified, summarized, and with a reply draft ready.

---

## What it does

1. Polls Gmail every 5 minutes for unread inbox emails
2. Deduplicates against a Google Sheet so no email is ever processed twice
3. Sends each new email to GPT-4o-mini for classification, priority scoring, summarization, assignee suggestion, and reply drafting
4. Applies a category-specific Gmail label to every email
5. Logs the full classification result to a Google Sheet audit trail
6. For **CRITICAL** emails: sends an immediate Slack alert to the manager and saves an AI-drafted reply to Gmail Drafts (threaded, ready to send)
7. For **HIGH** priority emails: sends a Slack notification to the team channel with full context
8. For **MEDIUM/LOW**: silently labeled and logged — inbox stays clean, nothing surfaces in Slack

---

## Workflow

**File:** `email-classifier.json`

### Flow diagram

```
Schedule (every 5 min)
  --> Read Processed Emails Sheet      (once per run)
  --> Build Processed IDs Set          (aggregate all rows into 1 item)
  --> Fetch Unread Emails              (Gmail: is:unread in:inbox, up to 50)
  --> Extract & Filter New Emails      (dedup + header extraction + body cleaning)
  --> Mark as Processed (Sheet)        (written before classification — prevents re-runs on failure)
  --> Classify & Summarize (GPT)       (category, priority, summary, assignee, deadline, draft reply)
  --> Parse AI Response                (safe JSON parse + restores email fields + maps label ID)
  --> Apply Gmail Label                (category label applied to the email thread)
  --> Log to Email Log Sheet           (full audit record)
  --> Route by Priority (Switch)
         |-- CRITICAL --> Notify Manager (Slack) --> Create Draft Reply (Gmail Drafts)
         |-- HIGH     --> Notify Team (Slack)
         |-- MEDIUM / LOW --> (end — labeled and logged only)
```

### Node-by-node breakdown

#### 1. Every 5 Minutes (`schedule-trigger`)
Schedule Trigger that fires every 5 minutes. Controls the polling cadence — adjust the interval in this node to change how frequently the inbox is checked.

#### 2. Read Processed Emails Sheet (`read-processed-sheet`)
Reads the entire `ProcessedEmails` tab from Google Sheets at the start of each run. Returns all rows — one item per row.

> Runs **once per execution**, not once per email. This is intentional: avoids N redundant API calls and is the foundation of the dedup pattern.

#### 3. Build Processed IDs Set (`build-processed-ids`)
Aggregates all sheet rows into a **single output item** containing an array of every message ID already processed:

```js
const processedIds = rows.map(r => r.json.messageId).filter(Boolean);
return [{ json: { processedIds } }];
```

The resulting single item is referenced globally downstream via `$('Build Processed IDs Set').first()`. Because this node always outputs exactly one item, `.first()` is safe and correct.

#### 4. Fetch Unread Emails (`fetch-emails`)
Gmail node using the `getAll` operation with the query `is:unread in:inbox`. Fetches up to 50 emails per run in full format (headers + body included in the response).

#### 5. Extract & Filter New Emails (`extract-filter-new`)
The core dedup and extraction node. It:

1. Loads the processed IDs built in step 3 via `$('Build Processed IDs Set').first().json.processedIds`
2. Skips any email whose `id` is already in the set
3. For each new email, recursively walks `payload.parts` to find the `text/plain` body part, falling back to the raw payload body
4. Strips all HTML tags and collapses whitespace
5. Truncates the body to **3000 characters** (with a `[truncated]` marker) to stay within GPT token limits
6. Extracts: `messageId`, `threadId`, `subject`, `from`, `to`, `date`, `body`

Returning an empty array stops execution cleanly with no errors when there are no new emails.

#### 6. Mark as Processed (Sheet) (`mark-processed`)
Appends the new email's ID and metadata to the `ProcessedEmails` tab **before** the GPT classification step. This is intentional: if the OpenAI call or any subsequent step fails, the email is already recorded as seen and won't be re-processed on the next run — preventing duplicate Slack alerts.

Columns written: `messageId`, `subject`, `from`, `processedAt`

#### 7. Classify & Summarize (GPT) (`classify-gpt`)
Sends the email to `gpt-4o-mini` with a structured system prompt. The user message is built by referencing `$('Extract & Filter New Emails').item.json` directly — this bypasses the fact that `$json` after the Google Sheets append may contain sheet response data rather than email fields. n8n's 1:1 item pairing ensures the correct email is always referenced.

**Temperature:** `0.2` — keeps classification consistent and deterministic.

**AI output format:**
```json
{
  "category": "SUPPORT",
  "priority": "HIGH",
  "summary": "Client reporting login failure since yesterday's deployment. Needs urgent fix.",
  "requires_response": true,
  "response_deadline": "24h",
  "suggested_assignee": "support",
  "draft_reply": "Hi [Name], thank you for reaching out..."
}
```

**Categories:**

| Category | Description |
|---|---|
| `SUPPORT` | Customer or user support requests |
| `SALES` | Sales inquiries, proposals, deal-related |
| `HR` | Human resources, hiring, internal people matters |
| `FINANCE` | Invoices, payments, budgets, financial requests |
| `LEGAL` | Contracts, compliance, legal notices |
| `INTERNAL` | Internal team communications |
| `NEWSLETTER` | Subscriptions, marketing emails, digests |
| `SPAM` | Unsolicited bulk mail, phishing |
| `OTHER` | Anything that doesn't fit the above |

**Priority levels:**

| Priority | When assigned |
|---|---|
| `CRITICAL` | Legal risk, security incident, executive escalation, high-value client emergency |
| `HIGH` | Client request, sales opportunity, time-sensitive internal matter |
| `MEDIUM` | Standard questions, FYI updates, routine requests |
| `LOW` | Newsletters, automated notifications, low-urgency items |

#### 8. Parse AI Response (`parse-ai`)
Safely parses the raw JSON string from GPT. Falls back to `category: OTHER, priority: MEDIUM` if parsing fails — the workflow never crashes on a malformed AI response. Also:

- Restores the original email fields (`...email`) from `$('Extract & Filter New Emails').item.json` since the GPT node's output replaces `$json`
- Maps the AI category to a Gmail label ID using the `labelMap` object (user-configurable placeholders)

#### 9. Apply Gmail Label (`apply-label`)
Gmail node that calls `addLabels` using the `gmailLabelId` resolved in step 8. Labels the email thread in Gmail so the inbox is visually organized by category without manual sorting.

#### 10. Log to Email Log Sheet (`log-to-sheet`)
Appends a full classification record to the `EmailLog` tab. Serves as the audit trail and the data source for a future daily digest workflow.

Columns written: `messageId`, `subject`, `from`, `date`, `category`, `priority`, `summary`, `requires_response`, `response_deadline`, `suggested_assignee`, `processedAt`

#### 11. Route by Priority (`route-priority`)
Switch node (v3) with two named outputs:

| Output | Condition | Downstream |
|---|---|---|
| `CRITICAL` | `priority === "CRITICAL"` | Notify Manager → Create Draft |
| `HIGH` | `priority === "HIGH"` | Notify Team |
| *(fallback)* | `MEDIUM` or `LOW` | No output — execution ends here |

#### 12. Notify Manager (Slack) (`notify-manager`)
Posts a formatted alert to the manager's Slack channel. Message includes: sender, subject, category, suggested assignee, response deadline, AI summary, and the Gmail message ID. Also informs the manager that a draft reply has been saved to Gmail Drafts.

Only fires for `CRITICAL` emails.

#### 13. Create Draft Reply (Gmail) (`create-draft`)
Creates a draft in Gmail Drafts using the AI-generated `draft_reply` text. The draft is threaded to the original message via `threadId` so it appears inline in the conversation. The manager only needs to review and hit Send — no typing required.

Only fires for `CRITICAL` emails.

#### 14. Notify Team (Slack) (`notify-team`)
Posts a formatted notification to the team Slack channel for `HIGH` priority emails. Same format as the manager alert but without the draft note. No Gmail draft is created for HIGH — the team handles the response.

---

## Google Sheets structure

Create a single Google Sheet with two tabs:

### Tab 1: `ProcessedEmails`

| Column | Description |
|---|---|
| `messageId` | Gmail message ID (dedup key) |
| `subject` | Email subject |
| `from` | Sender address |
| `processedAt` | ISO timestamp of when the workflow processed it |

### Tab 2: `EmailLog`

| Column | Description |
|---|---|
| `messageId` | Gmail message ID |
| `subject` | Email subject |
| `from` | Sender address |
| `date` | Original send date from email headers |
| `category` | AI-assigned category |
| `priority` | AI-assigned priority level |
| `summary` | AI-generated one-to-two sentence summary |
| `requires_response` | Boolean — whether the email needs a reply |
| `response_deadline` | `immediate`, `24h`, `48h`, `this_week`, or `none` |
| `suggested_assignee` | Recommended team or person to handle |
| `processedAt` | ISO timestamp of classification |

---

## Gmail labels setup

Before activating the workflow, create all nine category labels in Gmail:

`SUPPORT` · `SALES` · `HR` · `FINANCE` · `LEGAL` · `INTERNAL` · `NEWSLETTER` · `SPAM` · `OTHER`

**How to find a label's ID:**
1. Go to Gmail → Settings (gear icon) → See all settings → Labels
2. Hover over any label — the label ID appears in the URL bar as something like `Label_1234567890`
3. Copy that ID and paste it into the `labelMap` object in the **Parse AI Response** node

```js
const labelMap = {
  SUPPORT:    'Label_12345678',   // replace with your actual IDs
  SALES:      'Label_12345679',
  HR:         'Label_12345680',
  // ...
};
```

---

## Setup checklist

### Credentials required

| Service | Type | Used in |
|---|---|---|
| Gmail | OAuth2 | Fetch emails, apply labels, create drafts |
| Google Sheets | OAuth2 | Both Sheets nodes |
| OpenAI | API Key | GPT classification node |
| Slack | OAuth / Bot Token | Both Slack notification nodes |

### Placeholders to replace

| Placeholder | Node | What to set |
|---|---|---|
| `YOUR_GOOGLE_SHEET_ID` | All Sheets nodes | Sheet ID from the URL |
| `YOUR_GSHEETS_CREDENTIAL_ID` | All Sheets nodes | n8n Google Sheets credential ID |
| `YOUR_GMAIL_CREDENTIAL_ID` | All Gmail nodes | n8n Gmail OAuth2 credential ID |
| `YOUR_OPENAI_CREDENTIAL_ID` | Classify & Summarize node | n8n OpenAI credential ID |
| `YOUR_SLACK_CREDENTIAL_ID` | Both Slack nodes | n8n Slack credential ID |
| `YOUR_SLACK_MANAGER_CHANNEL_ID` | Notify Manager node | Slack channel ID for manager alerts |
| `YOUR_SLACK_TEAM_CHANNEL_ID` | Notify Team node | Slack channel ID for team alerts |
| `YOUR_LABEL_ID_*` | Parse AI Response — `labelMap` | Gmail label IDs (one per category) |
| `YOUR_ERROR_WORKFLOW_ID` | Workflow settings | n8n error-handling workflow ID |

### Step-by-step activation

1. Create the Google Sheet with both tabs and the column headers listed above.
2. Create the 9 Gmail category labels and note their IDs.
3. In n8n, create credentials for Gmail (OAuth2), Google Sheets (OAuth2), OpenAI, and Slack.
4. Import `email-classifier.json` into n8n.
5. Replace all placeholders. Paste the Gmail label IDs into the `labelMap` in Parse AI Response.
6. Set up an error workflow in n8n (or remove the `errorWorkflow` setting if not needed yet).
7. Activate the workflow.

---

## Error handling

- **Dedup safety:** `ProcessedEmails` is written *before* the GPT call. If OpenAI times out or Slack is down, the email won't be re-processed and double-notified on the next run.
- **AI parse failures:** The Parse AI Response node catches JSON parse errors and falls back to `category: OTHER, priority: MEDIUM` — the workflow continues and the email still gets labeled and logged.
- **No new emails:** `Extract & Filter New Emails` returns an empty array when all fetched emails are already in the processed log. n8n stops execution cleanly — no errors, no notifications.
- **Error workflow:** Both the workflow-level `errorWorkflow` setting points to a dedicated error handler. Configure it to send a Slack or email alert when any node throws an unhandled exception.

---

## Routing logic summary

| Priority | Gmail label | Sheet log | Slack | Draft reply |
|---|---|---|---|---|
| CRITICAL | Yes (category) | Yes | Manager channel | Yes (Gmail Drafts) |
| HIGH | Yes (category) | Yes | Team channel | No |
| MEDIUM | Yes (category) | Yes | No | No |
| LOW | Yes (category) | Yes | No | No |

---

## Recommended future additions

### Daily digest workflow (separate n8n workflow)
A nightly scheduled workflow that reads the `EmailLog` sheet, filters for the current day, groups by priority and category, and sends a formatted summary email or Slack message to the manager. The `EmailLog` tab is already structured to support this without any changes.

### Per-category Slack routing
The `suggested_assignee` field returned by GPT (e.g., `support`, `sales`, `hr`) can be used in an extended Switch node to route HIGH-priority emails to category-specific Slack channels rather than a single team channel.

### Reply confirmation loop
Extend Workflow 2 (modeled after the YouTube Post Approved Reply pattern) to accept a webhook POST from Slack with an approved or edited reply, then send it as an actual Gmail reply rather than just saving to Drafts.
