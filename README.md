# n8n Automation Projects

A collection of production n8n workflows built to automate real estate lead management, field service operations, and email outreach tracking. Each project targets a specific business process and integrates multiple external services through a centralized architecture.

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

---

## Shared Infrastructure

All four workflows share a common **Google Sheets master dataset** (`dataset_emails.csv`) as the central record of email leads. Each workflow interacts with it differently:

| Workflow | Operation | Match Key |
|---|---|---|
| Lead Automation | Append new leads | — |
| Instantly Sheets | Update reply status | `email` |

The **Internal System** (staging API) is used by both the Email Automation and Operation System workflows for authentication, order retrieval, and status recording.

---

## Tech Stack

| Layer | Tools |
|---|---|
| Automation platform | n8n |
| AI / LLM | OpenAI GPT-4.1-mini, GPT-4o, GPT-5-mini |
| Communication | WhatsApp Business API, Gmail |
| CRM / Outreach | EasyBroker, Instantly |
| Field Service | Zoho FSM |
| Data storage | Google Sheets |
| Internal backend | Internal System API (staging) |
