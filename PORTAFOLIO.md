# n8n Automation Portfolio

This repository brings together automations built for real operations. These are not generic demos: each workflow solves a specific business bottleneck, connects external systems, and reduces manual work across sales, operations, content, and administrative processes.

## What this portfolio solves

- Reduces response times in sales, support, and lead handling.
- Standardizes processes that previously depended on manual follow-up.
- Applies AI where it creates real value: classification, extraction, validation, and personalization.
- Connects contact channels, CRMs, tracking sheets, and internal systems without building custom software from scratch.

## Project showcase

### 1. Real estate lead automation with EasyBroker

**Project:** `n8n-easybroker_lead_automation`

**Business problem**  
Real estate teams receive leads through the CRM, but they often lose speed on first response or capture them inconsistently across tracking sheets, the sales CRM, and channels like WhatsApp.

**How it was solved**  
A workflow was built to query EasyBroker in rolling 60-minute windows, normalize contact data, and send it to a master Google Sheets dataset. From there, the leads are routed into the internal CRM and the first WhatsApp outreach is triggered automatically.

**Operational outcome**  
The business gets a much faster capture-and-contact process, with a single source of truth for sales follow-up.

**Integrated technologies**  
EasyBroker API, Google Sheets, WhatsApp API, internal CRM, n8n.

---

### 2. Email reply tracking in Instantly

**Project:** `n8n-Instantly_CRM`

**Business problem**  
In outbound email campaigns, replies can be missed if lead status is not updated quickly. That leads to delayed follow-up or duplicate outreach.

**How it was solved**  
A nightly sync was implemented between Instantly and Google Sheets. The workflow checks recent messages, identifies the sender email, and automatically updates the lead status to `Replied` while refreshing the last-contact timestamp.

**Operational outcome**  
The sales team works from current data and avoids chasing leads that have already responded.

**Integrated technologies**  
Instantly API, Google Sheets, n8n.

---

### 3. Shopify abandoned checkout recovery with AI

**Project:** `n8n-Shopify-Abandoned-Checkout`

**Business problem**  
Abandoned checkouts represent lost revenue, and generic recovery emails usually underperform because they ignore customer context and cart details.

**How it was solved**  
The workflow receives `checkout/update` events from Shopify, deduplicates records in Airtable, enriches the customer profile, and generates a personalized recovery message with GPT-4o. A second model then scores the text and, if it does not meet the desired threshold, suggests improvements and forces a rewrite before the message is sent to Klaviyo.

**Operational outcome**  
Recovery no longer depends on static templates. It becomes a personalized messaging system with quality control and full traceability.

**Integrated technologies**  
Shopify, OpenAI, Airtable, Klaviyo, n8n.

---

### 4. Intelligent email classifier for operational inboxes

**Project:** `n8n-email-classifier`

**Business problem**  
When support, sales, finance, legal, and promotional noise all land in the same inbox, the bottleneck is not reading email. It is prioritizing correctly and acting on time.

**How it was solved**  
A workflow checks Gmail every 5 minutes, deduplicates already processed emails, and uses GPT-4o-mini to classify, prioritize, summarize, and suggest a reply. It then labels the email in Gmail, logs it in Google Sheets, and sends Slack notifications only for critical or high-priority cases. Critical cases also get a draft reply saved automatically.

**Operational outcome**  
The inbox becomes an automatically triaged queue where only messages that truly need human attention are escalated.

**Integrated technologies**  
Gmail, OpenAI, Google Sheets, Slack, n8n.

---

### 5. Immediate SMS response for Google Guaranteed leads

**Project:** `n8n-google-guaranted-leads`

**Business problem**  
Google Local Services leads are extremely sensitive to response time. If the business takes minutes or hours to respond, conversion probability drops quickly.

**How it was solved**  
The workflow automates incoming email reading, HTML parsing to extract name, phone, and requested service, phone validation, deduplication in Google Sheets, and business-hours checks. If everything is valid, it sends an immediate SMS through Twilio and notifies the client that the lead has already been contacted.

**Operational outcome**  
The company dramatically reduces first-response time and avoids losing opportunities because of operational delay.

**Integrated technologies**  
Gmail, Twilio, Google Sheets, n8n.

---

### 6. Work order execution system over WhatsApp

**Project:** `n8n-operation_system.json`

**Business problem**  
In field operations, executing service orders means coordinating technicians, supervisors, evidence, validations, and closeout steps across several systems. When that process depends on calls, scattered chat messages, and manual updates, it becomes slow and error-prone.

**How it was solved**  
A conversational WhatsApp workflow was built to validate the order, authenticate the user, query the pending activity in the internal system, and guide the technician step by step. The workflow requests photo or text evidence, validates it with AI or a human agent depending on the case, uploads it to Zoho FSM, records the result in the internal system, and automatically moves to the next activity. At the end, it distributes the final PDF report to the responsible parties.

**Operational outcome**  
Field execution is centralized in a single channel, with full traceability from work order start to documented completion.

**Integrated technologies**  
WhatsApp Business API, OpenAI GPT-4o, internal system, Zoho FSM, n8n.

---

### 7. Service order data extraction from email

**Project:** `n8n-email_automation`

**Business problem**  
Many service orders and operational requests arrive by email in poorly structured formats such as HTML tables or Excel files. Turning that into system-ready data takes time and introduces capture errors.

**How it was solved**  
A workflow was designed to monitor Gmail, detect incoming orders, extract the information from HTML or attachments, use AI to convert it into a standardized JSON payload, and insert it into the internal system through an API. If required fields are missing, the workflow summarizes the validation issue and replies to the requester by email.

**Operational outcome**  
Manual re-entry is removed and the path from unstructured email to executable operation becomes much faster.

**Integrated technologies**  
Gmail, OpenAI, internal API, n8n.

---

### 8. YouTube comment monitor with classification and assisted replies

**Project:** `n8n-youtube-comment-monitor`

**Business problem**  
Brands and creators who publish frequently cannot review every comment manually without losing context, especially when they need to distinguish real questions, collaboration opportunities, spam, or toxic messages.

**How it was solved**  
The system uses two workflows. The first checks new comments by video, deduplicates against a historical log, classifies them with AI, and generates a proposed reply. Useful comments are sent to Slack for human review. The second workflow receives the approved or edited reply and posts it to YouTube while updating the log.

**Operational outcome**  
The team can moderate and respond at scale without losing human judgment at the final publishing step.

**Integrated technologies**  
YouTube Data API, OpenAI, Google Sheets, Slack, n8n.

---

### 9. Telegram bot for CFDI invoice capture

**Project:** `n8n-telegram-invoice-bot`

**Business problem**  
Registering invoices manually from photos or PDFs slows down reconciliation, accounting validation, and tax control, especially when the documents arrive through messaging apps.

**How it was solved**  
A Telegram bot receives an invoice image or PDF, extracts fiscal fields through visual analysis, and structures them into JSON. It then performs an upsert into Google Sheets using the CFDI UUID as the unique key and confirms back to the user that the invoice has been registered.

**Operational outcome**  
Invoice capture moves from a manual, duplicate-prone process to an AI-assisted flow with UUID-based control.

**Integrated technologies**  
Telegram Bot API, Anthropic, OpenRouter, Google Sheets, n8n.

---

### 10. Telegram bot for real estate PDF sheets

**Project:** `n8n-telegrambot-pdf-maker`

**Business problem**  
Real estate agents need to produce polished property sheets quickly, but they usually depend on designers, manual templates, or separate tools for text, images, and PDF generation.

**How it was solved**  
A conversational bot lets the agent send property photos and free-form text. The system stores the session, structures the data with AI, generates a sales-ready HTML sheet, and converts it into a PDF with Gotenberg. Everything is returned in the same Telegram chat.

**Operational outcome**  
Commercial material can be produced much faster without relying on external design tools or manual layout work.

**Integrated technologies**  
Telegram Bot API, Anthropic, Gotenberg, n8n.

## Repeating value patterns

- Orchestration between intake channels and systems of record.
- Deduplication to avoid reprocessing and duplicate outreach.
- AI applied to classification, extraction, validation, or personalization.
- Logging into operational sheets or auxiliary stores for auditability.
- Automations designed for real operations, not just proof-of-concept demos.

## Covered sectors

- Real estate
- E-commerce
- Field operations
- Administration and finance
- Content marketing
- Sales and outbound prospecting

## Summary

This portfolio shows how n8n can act as a business automation layer, connecting CRMs, messaging, email, AI, operational sheets, and internal systems to turn manual processes into traceable, faster, and scalable workflows.
