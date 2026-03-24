# Google Guaranteed Lead → SMS Automation

## Overview

This n8n workflow automates the first-response process for **Google Guaranteed** leads. Every time a new lead notification arrives from Google Local Services Ads, the workflow parses the email, validates the contact data, deduplicates against a Google Sheets log, checks business hours, and fires an immediate SMS to the prospect via **Twilio** — all within seconds. A final email notification is sent to the client confirming the outreach.

---

## Architecture Summary

```
Gmail Trigger (every minute)
    │
    ├─ [Not from Google] → dead-end
    │
    └─ [From Google + subject contains "lead"]
            ↓
     Code — Parse Email
            ↓
     IF — Phone Valid?
      ├─ No  → Gmail — Parsing Alert (notify operator)
      └─ Yes
            ↓
     Sheets — Check Duplicate
            ↓
     IF — Duplicate?
      ├─ Duplicate → dead-end (silent skip)
      └─ New lead
            ↓
     Code — Check Business Hours
            ↓
     IF — Business Hours?
      ├─ Outside hours → dead-end
      └─ Within hours
            ↓
     Twilio — Send SMS
            ↓
     Sheets — Log Lead
            ↓
     Gmail — Notify Client
```

---

## Trigger

**Node:** `Gmail Trigger`
- Polls Gmail **every minute**
- Filters: sender `googlelocalbusiness-noreply@google.com`, subject contains `New lead`
- Provides: email `subject`, `from`, HTML body, and binary attachments

> **Note:** For instant delivery, the Gmail Trigger can be replaced with a Gmail Pub/Sub webhook via Google Cloud.

---

## Stage 1 — Source Validation

### `IF — From Google?`
Double-checks that the email:
- `from` contains `@google.com`
- `subject` contains `lead`

Both conditions must be true. Non-matching emails are silently dropped.

---

## Stage 2 — Email Parsing

### `Code — Parse Email`
Extracts structured lead data from the raw HTML email body using label-based regex patterns:

| Field | Source label |
|---|---|
| `name` | `Customer name` / `Name` |
| `phone` | `Phone` / `phone number` |
| `service` | `Job type` / `Service` / `service requested` |
| `date` | `Date` (falls back to `$now`) |

**Phone normalization:**
- Strips non-numeric characters
- Converts 10-digit numbers to E.164 format (`+1XXXXXXXXXX`)
- Falls back to a full-body regex scan if the labeled field is not found

**Output fields:**
```json
{
  "name": "string",
  "phone": "+1XXXXXXXXXX | null",
  "service": "string",
  "date": "ISO string",
  "raw_subject": "string",
  "parsed_ok": true | false
}
```

### `IF — Phone Valid?`
Checks `parsed_ok`. If `false` (name or phone could not be extracted):
- Routes to **`Gmail — Parsing Alert`** — sends an internal notification to the operator with the raw subject and timestamp so the email can be reviewed manually.

---

## Stage 3 — Deduplication

### `Sheets — Check Duplicate`
Performs a lookup on the `Leads` sheet in Google Sheets using `phone` as the match key.

### `IF — Duplicate?`
If the phone number already exists in the sheet → the lead is a duplicate and the workflow ends silently. Only new, unseen phone numbers proceed.

---

## Stage 4 — Business Hours Check

### `Code — Check Business Hours`
Calculates the current time in **US Eastern (Miami)** timezone, accounting for DST:
- **Weekdays only** (Monday–Friday)
- **Hours:** 8:00 AM – 8:00 PM ET

Appends `send_now: true/false`, `et_hour`, and `day_of_week` to the item.

### `IF — Business Hours?`
- `send_now = true` → proceeds to SMS
- `send_now = false` → dead-end (lead is not contacted outside hours)

---

## Stage 5 — SMS Outreach

### `Twilio — Send SMS`
Sends a personalized SMS to the lead's normalized phone number:

```
Hi {name}, this is [Business Name] — we received your request for {service}.
When's a good time to connect? Reply here or call us at [PHONE].
```

- **From:** configured Twilio number
- **To:** `$json.phone` (E.164 format)

---

## Stage 6 — Logging & Notification

### `Sheets — Log Lead`
Appends a new row to the `Leads` Google Sheet with:

| Column | Value |
|---|---|
| `timestamp` | Execution time |
| `name` | Lead name |
| `phone` | Normalized phone |
| `service` | Service requested |
| `sms_sent` | `true` |
| `sms_time` | SMS send time |
| `status` | `contacted` |

### `Gmail — Notify Client`
Sends a confirmation email to the business owner/client with a summary of the lead that was automatically contacted — name, phone, service, and SMS timestamp.

---

## External Services & Credentials

| Service | Purpose |
|---|---|
| Gmail OAuth2 | Email trigger, parsing alerts, client notifications |
| Google Sheets OAuth2 | Duplicate check + lead logging |
| Twilio API | SMS outreach to the lead |

---

## Error Handling Summary

| Scenario | Handler |
|---|---|
| Email not from Google / subject mismatch | Silent drop (IF false branch) |
| Phone or name could not be parsed | Gmail alert to operator |
| Lead already exists in sheet (duplicate) | Silent skip |
| Lead arrives outside business hours | Silent skip (no SMS sent) |

---

## Setup Checklist

Before activating, replace all placeholder values in the workflow:

1. Gmail OAuth2 credential ID
2. Google Sheets credential ID + Sheet ID
3. Twilio credential ID + Twilio phone number
4. Set `To` address in both Gmail nodes (parsing alert + client notification)
5. Update SMS message template with the real business name and callback phone
6. Verify the Google Guaranteed sender address matches the client's actual notification emails
7. *(Optional)* Replace Gmail Trigger polling with a Gmail Pub/Sub webhook for instant lead response
