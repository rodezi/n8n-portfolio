# Instantly Sheets — Email Reply Tracker

## Overview

This is a lightweight automation that keeps the **Google Sheets master dataset** in sync with email activity from **Instantly**. Every night it pulls the latest replies from the Instantly unibox, fetches the full message details for each one, and uses the **email address as the unique key** to update the corresponding row in the sheet — setting its status to `Replied` and recording the timestamp of the last contact.

The sheet is the central record of all email leads. This workflow ensures that any reply activity is reflected there automatically, without manual intervention.

---

## Flow

```
Schedule Trigger (10:00 PM)
        ↓
  Date & Time  (capture current timestamp)
        ↓
  Get many unibox messages  (fetch up to 10 recent replies from Instantly)
        ↓
  Edit Fields 0–4  (parallel — extract message ID from each item by index)
        ↓
  Code in JavaScript  (mark each record as processed)
        ↓
  Edit Fields5  (normalize Lead ID into "Leads" field)
        ↓
  Get unibox message  (fetch full message details for each ID)
        ↓
  Append or update row in sheet  (upsert into Google Sheets — matched by email)
```

---

## Nodes

### `Schedule Trigger`
Runs once daily at **10:00 PM**. Fires the workflow automatically every night to capture the day's replies.

### `Date & Time`
Captures the current datetime at execution time, passed downstream for timestamping.

### `Get many unibox messages`
Fetches the **latest 10 messages** from the Instantly unibox (inbox/replies):
- **Resource:** unibox
- **Operation:** get many
- **Limit:** 10
- **Credential:** Instantly API

Returns an `items[]` array where each element contains a message `id`.

### `Edit Fields` / `Edit Fields1` / `Edit Fields2` / `Edit Fields3` / `Edit Fields4`
Five parallel Set nodes — each extracts one message ID from the response array by index (`items[0].id` through `items[4].id`), storing it as `Lead 1`. This processes up to 5 messages per run in parallel.

### `Code in JavaScript`
Marks each item as processed by adding a tracking flag:
```javascript
for (const item of $input.all()) {
  item.json.myNewField = 1;
}
return $input.all();
```

### `Edit Fields5`
Normalizes the extracted ID into a single field called `Leads`, used as the input for the unibox message fetch:
```
Leads = {{ $json['Lead 1'] }}
```

### `Get unibox message`
Fetches the **full message details** for a single message by its ID:
- **Resource:** unibox
- **Operation:** get
- **Message ID:** `{{ $json.Leads }}`
- **Credential:** Instantly API

Returns fields including `from_address_json` (sender email) and `timestamp_created`.

### `Append or update row in sheet`
**Upserts** the reply data into the master Google Sheets dataset:
- **Operation:** `appendOrUpdate`
- **Sheet:** `dataset_emails.csv`
- **Match key:** `email` — looks up the existing row by the sender's email address
- **Fields updated:**

| Column | Value |
|---|---|
| `email` | `from_address_json[0].address` |
| `Status` | `"Replied"` (hardcoded) |
| `Last Contact` | `timestamp_created` (from Instantly message) |

All other columns (`titulo`, `website`, `telefono`, `ubicacion`, `Notes`) are left untouched.

---

## Key Design Decision — Email as ID

The `email` field is used as the **matching key** for the upsert. This means:
- If a row with that email already exists → it is **updated** (`Status = Replied`, `Last Contact` refreshed)
- If no row exists → a **new row is appended**

This guarantees the sheet always reflects the current reply state of every lead without duplicating records.

---

## External Services & Credentials

| Service | Purpose |
|---|---|
| Instantly API | Fetch unibox messages and message details |
| Google Sheets OAuth2 | Read and update the master lead dataset |

---

## Master Dataset Schema

The sheet `dataset_emails.csv` is the shared lead record used across multiple workflows. Its columns:

| Column | Description |
|---|---|
| `titulo` | Lead title or property type |
| `website` | Source website |
| `telefono` | Phone number |
| `ubicacion` | Location |
| `email` | Email address **(primary key)** |
| `Status` | Current lead status (`Replied`, etc.) |
| `Last Contact` | Timestamp of the most recent interaction |
| `Notes` | Free-form notes |

---

## Notes

- The workflow processes up to **5 message IDs per run** in parallel (Edit Fields 0–4). With a fetch limit of 10, 5 are extracted by index — extending to 10 requires adding Edit Fields 5–9.
- This workflow is **active** and runs on a fixed nightly schedule.
- It is intentionally minimal — it does not parse message content, assign leads, or trigger outreach. Its sole responsibility is keeping the sheet's `Status` and `Last Contact` columns accurate.
