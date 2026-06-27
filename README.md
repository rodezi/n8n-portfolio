# n8n Automation Projects

## Portfolio Showcase

For a business-focused portfolio version in Spanish, see [PORTAFOLIO.md](PORTAFOLIO.md).

A collection of production n8n workflows built to automate real estate lead management, field service operations, email outreach tracking, lead response, content engagement, inbox triage, and e-commerce recovery. Each project targets a specific business process and integrates multiple external services through a centralized architecture.

---

## Projects

### 1. [Email Automation — Service Order Data Extractor](email_automation/email_automation.md)

Automates the extraction of service order data from incoming emails. Incoming messages may contain HTML tables or Excel attachments — the workflow parses both, uses **GPT-4.1-mini** to structure the extracted data into a standardized JSON schema, and injects it into the **Internal System** via API. If mandatory fields are missing, the validation error is summarized by an LLM and sent back to the requester by email.

**Key integrations:** Gmail, OpenAI, Internal System API
**Trigger:** Gmail poll (every hour)
**Output:** Validated service order injected into the Internal System

---

### 2. [Lead Automation — Real Estate Lead Management](easybroker_lead_automation/lead_automation.md)

Fetches real estate leads from **EasyBroker CRM** multiple times per day using a rolling 60-minute time window, standardizes the contact data, and appends it to a **Google Sheets** master dataset. Leads are then routed for CRM injection and automated **WhatsApp outreach** to begin the sales conversation.

**Key integrations:** EasyBroker API, Google Sheets, WhatsApp API, Internal CRM
**Trigger:** Multiple daily executions (manual / scheduled)
**Output:** Lead records written to Google Sheets + CRM + WhatsApp contact initiated

---

### 3. [Operation System — Field Service Work Order Automation](operation_system.json/operation_system.md)

Orchestrates the full lifecycle of field service work orders entirely over **WhatsApp**. A technician sends an order number to initiate execution; the system validates the order and the user's authorization through the **Internal System**, then guides the technician activity by activity — collecting photo or text evidence, validating it with **GPT-4o vision** (up to 3 attempts) or a human agent, uploading results to **Zoho FSM**, and recording completion status. Once all activities are done, a final PDF report is distributed to the operational leader and dispatcher via WhatsApp.

**Key integrations:** WhatsApp Business API, OpenAI GPT-4o, Internal System API, Zoho FSM
**Trigger:** WhatsApp webhook (inbound message)
**Output:** All activities validated and recorded + PDF report delivered

---

### 4. [Instantly Sheets — Email Reply Tracker](Instantly_CRM/Instantly_sheets.md)

A lightweight nightly workflow that keeps the **Google Sheets** master dataset in sync with email reply activity from **Instantly**. It fetches the latest messages from the Instantly unibox, extracts the sender's email address, and uses it as the unique key to upsert each row — updating the `Status` to `Replied` and refreshing the `Last Contact` timestamp. No content parsing or outreach logic is involved; its only job is keeping the sheet accurate.

**Key integrations:** Instantly API, Google Sheets
**Trigger:** Daily schedule (10:00 PM)
**Output:** `Status` and `Last Contact` updated in the master dataset

### 5. [Google Guaranteed Lead → SMS Automation](n8n-google-guaranted-leads/n8n-google-guaranted-leads.md)


Monitors Gmail every minute for new **Google Local Services Ads (Google Guaranteed)** lead notifications. When a lead arrives, the workflow parses the HTML email to extract the contact's name, phone, and service type — normalizing the phone to E.164 format. It then deduplicates against a **Google Sheets** log, checks **business hours (US Eastern)**, and fires an immediate personalized **SMS via Twilio**. The lead is logged to the sheet and a confirmation email is sent to the client. If parsing fails, an alert is sent to the operator.

**Key integrations:** Gmail, Twilio, Google Sheets
**Trigger:** Gmail poll (every minute)
**Output:** SMS sent to lead + row logged in Google Sheets + client notified by email

---

### 6. [YouTube Comment Monitor & Auto-Reply System](n8n-youtube-comment-monitor/youtube-system.md)

A two-workflow system that monitors YouTube comments across multiple videos on a recurring schedule. New comments are deduplicated using **Google Sheets**, classified by **GPT-4o-mini** into categories (question, feedback, spam, collaboration, hate speech, etc.), and routed to a **Slack** channel with the classification, confidence score, and a suggested reply draft for team review. Once the team approves or edits a reply, a single POST to the second workflow's webhook posts it back to YouTube via OAuth2 and updates the sheet log. SPAM and HATE_SPEECH comments are logged but never surface in Slack.

**Key integrations:** YouTube Data API v3, OpenAI GPT-4o-mini, Google Sheets, Slack
**Trigger:** Schedule (every 15 min) + Webhook (reply approval)
**Output:** Slack notification with AI-drafted reply + approved reply posted to YouTube

### 7. [Email Triage & AI Classifier](n8n-email-classifier/email-classifier.md)

Polls a Gmail inbox every 5 minutes and runs each new unread email through **GPT-4o-mini** for classification, priority scoring, summarization, and reply drafting. Emails are deduplicated via **Google Sheets**, labeled in Gmail by category (Support, Sales, HR, Finance, Legal, Internal, Newsletter, Spam, Other), and logged to a full audit sheet. **CRITICAL** emails trigger an immediate Slack alert to the manager and save an AI-drafted reply to Gmail Drafts — threaded and ready to send. **HIGH** priority emails notify the team channel. MEDIUM and LOW emails are silently labeled and logged, keeping the inbox clean without noise.

**Key integrations:** Gmail, OpenAI GPT-4o-mini, Google Sheets, Slack
**Trigger:** Schedule (every 5 min)
**Output:** Gmail labeled + EmailLog updated + Slack alert + Draft reply saved (CRITICAL only)

### 9. [Telegram Invoice Bot — CFDI Parser](n8n-telegram-invoice-bot/README.md)

Telegram bot that receives photos or PDFs of Mexican invoices (CFDI) and extracts all fiscal data automatically. **Claude Sonnet 4.6** performs visual analysis of the document and returns a structured JSON. An **Information Extractor** node (backed by OpenRouter GPT-5.1) parses the typed fields. If the message has no attachment, the bot prompts the user to resend. On success, all data is upserted into **Google Sheets** keyed on the CFDI UUID, and a confirmation message is sent back to the user.

**Key integrations:** Telegram Bot API, Anthropic Claude Sonnet 4.6, OpenRouter, Google Sheets
**Trigger:** Telegram webhook (inbound message)
**Output:** CFDI data upserted to Google Sheets + Telegram confirmation sent to user

---

### 8. [Shopify Abandoned Checkout Recovery](n8n-Shopify-Abandoned-Checkout/n8n-Shopify-Abandoned-Checkout-Recovery.md)

Intercepts Shopify `checkout/update` webhooks and runs each abandoned cart through an AI self-improvement loop. **GPT-4o** generates a personalized recovery email referencing the customer's name, cart items, total, and discount codes. **GPT-4o-mini** then scores its persuasiveness (1–10) and provides a specific improvement suggestion. If the score is below 7 and fewer than 3 attempts have been made, the message is rewritten incorporating the feedback and re-evaluated. The best-scoring version is fired to **Klaviyo** as a custom event, which triggers the actual email delivery via a Klaviyo flow. Every attempt is logged in **Airtable** with score, suggestion, and final status.

**Key integrations:** Shopify Admin API, OpenAI GPT-4o + GPT-4o-mini, Klaviyo Events API, Airtable
**Trigger:** Shopify webhook (`checkouts/update`)
**Output:** Personalized recovery email fired to Klaviyo + full Airtable audit trail

---

## Tech Stack

| Layer | Tools |
|---|---|
| Automation platform | n8n |
| AI / LLM | OpenAI GPT-4.1-mini, GPT-4o, GPT-4o-mini, GPT-5-mini, Anthropic Claude Sonnet 4.6 |
| Communication | WhatsApp Business API, Gmail, Twilio SMS, Slack, Telegram Bot API |
| CRM / Outreach | EasyBroker, Instantly, Klaviyo |
| E-commerce | Shopify Admin API |
| Field Service | Zoho FSM |
| Data storage | Google Sheets, Airtable |
| Internal backend | Internal System API (staging) |
