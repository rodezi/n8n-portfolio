# Lead Automation — Real Estate Lead Management

## Overview

This n8n workflow automates the daily retrieval and management of real estate leads from **EasyBroker CRM**. It runs multiple times per day, fetching contacts updated within the last 60 minutes, standardizes the data, appends it to a **Google Sheets** master dataset, and routes leads for CRM injection and **WhatsApp outreach**.

---

## Architecture Summary

```
Manual / Scheduled Trigger
        ↓
  EasyBroker API  (fetch contacts updated in last 60 min)
        ↓
    Split Out  (one item per contact)
        ↓
    Aggregate  (recombine into unified array)
        ↓
  Contacto 1–10  (parallel — extract each contact by index)
        ↓
  Estandarizacion  (standardize & mark records)
        ↓
  Contacto Estandar  (prepare final array)
        ↓
  Append row in sheet  →  Google Sheets master dataset
        ↓
  [CRM injection + WhatsApp outreach]
```

---

## Trigger

**Node:** `When clicking 'Execute workflow'`
- Manual trigger — intended to be executed multiple times per day
- Fetches only contacts updated in the **last 60 minutes** per run, ensuring no duplicate processing across runs

---

## Stage 1 — Lead Retrieval from EasyBroker

### `EasyBroker` (HTTP Request)
Fetches contacts from the EasyBroker CRM API:

- **GET** `https://api.easybroker.com/v1/contacts`
- **Auth:** `X-Authorization` header (API key via credentials)
- **Query param:** `search[updated_after]` = `{{ $now.minus({ minutes: 60 }).toISO() }}`

The dynamic time filter ensures each run only retrieves leads created or modified since the last execution window.

**Response structure:**
```json
{
  "pagination": {
    "limit": 20,
    "page": 1,
    "total": 2,
    "next_page": null
  },
  "content": [
    {
      "id": 20152441,
      "full_name": "...",
      "email": "...",
      "phone": "...",
      "agent": "...",
      "source": null,
      "created_at": "2026-02-19T13:41:12-06:00",
      "updated_at": "2026-02-19T13:41:12-06:00"
    }
  ]
}
```

---

## Stage 2 — Contact Array Processing

### `Split Out`
Splits the `content` array from the API response into individual items — one n8n item per contact — enabling parallel processing downstream.

### `Aggregate`
Recombines the split items back into a unified array so the full contact list is accessible by index in the next stage.

---

## Stage 3 — Individual Contact Extraction (Parallel)

### `Contacto 1` through `Contacto 10` (Set nodes)
Ten parallel Set nodes each extract one contact from the aggregated array by position:

| Node | Source Index |
|---|---|
| Contacto 1 | `content[0]` |
| Contacto 2 | `content[1]` |
| Contacto 3 | `content[2]` |
| ... | ... |
| Contacto 10 | `content[9]` |

All 10 nodes execute in parallel and converge at `Estandarizacion`. This supports up to **10 leads per polling window**.

---

## Stage 4 — Standardization

### `Estandarizacion` (Code node)
Iterates over all incoming contact items and applies a standardization pass — currently adds a processing flag to each record:

```javascript
for (const item of $input.all()) {
  item.json.myNewField = 1;
}
return $input.all();
```

This marks each record as processed and serves as the hook for any additional normalization logic (field renaming, formatting, enrichment).

### `Contacto Estandar` (Set node)
Stores the final standardized `content` array, preparing it for the write step.

---

## Stage 5 — Write to Master Dataset

### `Append row in sheet` (Google Sheets)
Appends each standardized contact as a new row in the master lead tracking spreadsheet.

- **Operation:** Append
- **Credential:** Google Sheets OAuth2
- **Sheet:** `dataset_emails.csv`

**Columns written:**

| Column | Description |
|---|---|
| `titulo` | Contact title / property type |
| `website` | Source website or listing URL |
| `telefono` | Phone number |
| `ubicacion` | Location / address |
| `email` | Email address |
| `Status` | Current lead status |
| `Last Contact` | Timestamp of last interaction |
| `Notes` | Free-form notes |

---

## Stage 6 — CRM Injection & WhatsApp Outreach

After leads are logged in the master sheet, the workflow routes each contact for:

1. **CRM injection** — contact record is created or updated in the internal CRM system
2. **WhatsApp outreach** — an automated initial message is sent to the lead via WhatsApp to begin the sales conversation

---

## Input Data Structure (EasyBroker Contact)

```json
{
  "id": "integer",
  "full_name": "string",
  "email": "string",
  "phone": "string",
  "agent": "string",
  "source": "string | null",
  "created_at": "ISO8601 datetime",
  "updated_at": "ISO8601 datetime"
}
```

---

## External Services & Credentials

| Service | Purpose |
|---|---|
| EasyBroker API | Source of real estate leads |
| Google Sheets OAuth2 | Write leads to master dataset |
| Internal CRM | Inject standardized lead records |
| WhatsApp API | Send automated outreach messages |

---

## Execution Model

The workflow is designed to run **multiple times per day**. Each run:
- Fetches only contacts updated in the **last 60 minutes**
- Processes up to **10 leads per run**
- Appends new rows to the sheet without overwriting existing data (`useAppend: true`)

This rolling-window approach ensures all daily leads are captured across runs without duplication.

---

## Notes

- The workflow currently supports up to **10 contacts per execution**. If more than 10 leads arrive within a 60-minute window, additional `Contacto N` nodes or a loop-based approach would be needed.
- The `Estandarizacion` code node is the central place to add field mapping, renaming, or enrichment logic before data is written to the sheet.
- No error handling is currently implemented — API failures or missing contacts will silently skip those items.
