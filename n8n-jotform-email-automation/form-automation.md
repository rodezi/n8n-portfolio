# Jotform Email Automation — Appointment Confirmation & Lead Capture

An n8n workflow that fires on **Jotform** booking-form submissions for **WebDevX**, normalizes the raw form payload, strips personally identifiable information (PII) with a **Guardrails** node, writes the sanitized lead to **Airtable**, and sends an appointment-confirmation email via **Gmail**. It also includes a **beehiiv** newsletter-subscription call for leads who opted into promotional communications.

---

## The problem this solves

A booking web form captures high-intent leads (name, contact info, service, preferred appointment time). Manually re-typing each submission into a CRM, removing sensitive data, answering "yes, my appointment is confirmed," and enrolling opt-in leads into the newsletter is slow, error-prone, and leaks PII across internal systems. This workflow turns each form submission into a sanitized Airtable record **and** an immediate confirmation email — automatically — with no human touch.

---

## What it does

1. Fires on every new submission to the configured Jotform
2. Re-maps the flatten-and-nest Jotform payload into clean, named fields (name, last name, phone, email, address, appointment, service, promotional opt-in)
3. Sanitizes a concatenated string of contact data, removing PII before it reaches storage
4. Creates a new lead record in Airtable using the sanitized phone and email plus the name, service, and appointment time
5. Branches on the promotional-notification opt-in answer
6. Sends the same appointment-confirmation email in both branches (Gmail)
7. Subscribes opted-in leads to the newsletter via the **beehiiv** Subscriptions API (node present but not yet wired into the graph)

---

## Workflow

**File:** `form-automation.json`

### Flow diagram

```
Jotform Trigger  (new form submission)
  --> Data (Set)           normalize payload into named fields
        --> Guardrails      sanitize contact string (PII)
              --> LeadRecord  Airtable create (sanitized email/phone + name/service/appt)
                    --> If    promotional opt-in === "Yes"?
                          |-- Yes --> Appt Confirmed (Gmail confirmation)
                          |-- No  --> Appt Confirmed1 (Gmail confirmation)
                                          --> Time Saved --> No Operation

(beehiiv Sub node present in the file but not connected to the graph)
```

### Node-by-node breakdown

#### 1. Jotform Trigger (`Jotform Trigger`)
Webhook node (form `262128588551868`, `jotFormTrigger`) that starts the execution the moment a visitor submits the Jotform booking form. Auth via the **JotForm API** credential.

#### 2. Data (`Data` — Set, v3.4)
Normalizes the nested Jotform payload into clean, flat output fields used everywhere downstream:

| Output field | Source expression |
|---|---|
| `Name` | `Full Name.first` |
| `Last Name` | `Full Name.last` |
| `Phone Number` | `Contact Number.full` |
| `Email` | `Email Address` |
| `Address` | `Address` (object) |
| `Date and time Appt` | `What date and time work best for you?.date` |
| `Services of interest` | `What services are you interested in?` |
| `Would you like to be notified about promotional services?` | same-named form field |

#### 3. Guardrails (`guardrails` — LangChain PII sanitize)
Runs a `sanitize` operation on a single concatenated string built from the phone, email, service, and name/last name. The `guardrails` PII check is set to type `all`, so it redacts every detected PII category. The resulting output exposes `checks` → `analyzerResults`, which downstream nodes use for the cleaned phone and email.

#### 4. LeadRecord (`leadRecord` — Airtable create)
Creates a new row in **Table 1** of the `demoJOTFORM - Web DEVAgency` Airtable base. Column mapping:

| Airtable column | Value |
|---|---|
| `Name` | from `Data` |
| `Last Name` | from `Data` |
| `phone` | `checks[0].info.analyzerResults[1].text` (sanitized) |
| `Correo electrónico` | `checks results[0].text` (sanitized) |
| `Service` | from `Data` (`Services of interest`) |
| `Appt` | from `Data` (`Date and time Appt`) |

The node has **retry on fail** enabled.

#### 5. If (`if`)
Branches on whether the form's *"Would you like to be notified about promotional services?"* field equals `Yes`:

- **True** → `Appt Confirmed`
- **False** → `Appt Confirmed1`

#### 6. Appt Confirmed / Appt Confirmed1 (Gmail, `appt-confirmed` / `appt-confirmed1`)
Two functionally identical Gmail nodes (one per IF branch). Each sends the appointment confirmation email:

- **To:** `$json.fields['Correo electrónico']`
- **Subject:** `Appt Confirmed! - WebDevx`
- **Plain-text body:** personalized with the lead's name, the WebDevX services requested, and the appointment time.

Because both branches send the same message, having two identical nodes is a known redundancy — both perform the same send.

#### 7. Beehive Sub (`beehiveSub` — HTTP Request)
A `POST` to the **beehiiv** Subscriptions API (`https://api.beehiiv.com/v2/publications/:publicationId/subscriptions`) intended to subscribe the lead's email to the publication newsletter.

> ⚠ **Not wired into the graph.** Nothing in `connections` points to this node, so it will not run in the current build. It is also not ready to run: it requires a real `:publicationId`, `YOUR_API_KEY`, and the subscriber email/fields to be populated.

#### 8. Time Saved → No Operation (`timeSaved` → `noOp`)
A placeholder terminal branch on the second Gmail output. Carries no logic and simply marks the end of that path.

---

## Airtable table structure

Table **Table 1** in base `demoJOTFORM - Web DEVAgency`.

| Column | Type | Written by workflow |
|---|---|---|
| `Name` | string | Yes |
| `Last Name` | string | Yes |
| `phone` | string | Yes (sanitized) |
| `Correo electrónico` | string | Yes (sanitized) |
| `Service` | string | Yes |
| `Appt` | string | Yes |
| `Notes` | string | No |
| `Assignee` | string | No |
| `Status` | options (`Todo`, `In progress`, `Done`) | No |
| `Attachments` | array | No |
| `Attachment Summary` | string | No |

---

## Setup checklist

### Credentials required

| Service | Type | Used in |
|---|---|---|
| JotForm | API | Jotform Trigger |
| Airtable | Personal Access Token | LeadRecord |
| Gmail | OAuth2 | Appt Confirmed / Appt Confirmed1 |
| beehiiv | API Key (header) | Beehive Sub (when wired up) |

### Placeholders to replace

| Placeholder | Node | What to set |
|---|---|---|
| `YOUR_API_KEY` | Beehive Sub | Your beehiiv API key (also choose an actual `publicationId` in the URL) |
| `:publicationId` | Beehive Sub | Actual beehiiv publication ID |
| `subscriber@example.com` | Beehive Sub | Replace with the lead's email output via `$json...` subscription body |

### Step-by-step activation

1. In n8n create credentials for JotForm, Airtable, Gmail, and (if using it) beehiiv.
2. Import `form-automation.json` into n8n.
3. Confirm the Jotform trigger is pointed at your form and the webhook is registered.
4. Verify the Airtable base/table IDs match your target table.
5. Wire the Beehive Sub node if you want newsletter enrollment (currently disconnected), then replace its placeholders.
6. Activate the workflow.

---

## Notes & caveats

- **Duplicate Gmail nodes:** `Appt Confirmed` and `Appt Confirmed1` are identical. Keeping both adds no behavior — the two IF branches could safely point to a single email node.
- **Redundant end branch:** `Time Saved → No Operation` is a placeholder with no effect and can be removed.
- **Beehive Sub is off-graph:** it exists in the file but no node connects to it, and it needs real credentials plus a publication ID before it can be used. The body/headers currently contain example values.
- **Email recipient source:** the Gmail nodes read `$json.fields['Correo electrónico']` from the Airtable create response. Validate that the Airtable node returns `fields` in its output (or switch to reading the sanitized email directly from the Guardrails output) so the recipient resolves correctly.