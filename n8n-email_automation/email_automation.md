# Email Automation — Service Order Data Extractor

## Overview

This n8n workflow automates the extraction of service order data from incoming emails. It handles two input formats — **HTML email bodies** (tables) and **Excel (.xlsx) attachments** — parses and standardizes the data, uses an AI agent to structure it into a defined JSON schema, validates the output against the **Internal System**, and sends error notifications if mandatory fields are missing.

---

## Architecture Summary

```
Gmail Trigger
    │
    ├─ [.xlsx attachment found]
    │       ↓
    │   Auth  ──→  Upload Excel  ──→  Parse HTML from response
    │                                           ↓
    │                              Extract & map table columns
    │                                           ↓
    │                              Flatten service data arrays
    │                                           ↓
    │                               AI Agent (GPT-4.1-mini)
    │                                           ↓
    │                              Auth  ──→  Validate JSON
    │                               ├─ Error → LLM Summary → Email notification
    │                               └─ Success → (end, data accepted by Internal System)
    │
    └─ [No .xlsx found] → No-op
```

---

## Trigger

**Node:** `Gmail Trigger2`
- Polls the `servicios` Gmail account every **hour**
- Downloads attachments with prefix `attachment_0`
- Provides: email `subject`, `from`, raw `html` body, and `binary` attachment data

---

## Stage 1 — Attachment Detection & Excel Extraction

### `Extractor xlsx` (Code node)
Inspects the email's binary attachments and filters for `.xlsx` files. If no Excel file is found, returns:
```json
{ "message": "No se encontraron archivos .xlsx" }
```
Otherwise, normalizes the binary key to `attachment` for downstream use.

### `If1`
Routes based on the result of `Extractor xlsx`:
- **No .xlsx found** → dead-end (No-op)
- **Excel found** → continues to Internal System authentication + file upload

---

## Stage 2 — Excel Upload to Internal System

### `Auth` node
Authenticates against the Internal System API:
- **POST** `{URLbase_stg}/login`
- Returns: `access_token`

### `Merge`
Combines the auth token branch and the binary attachment branch by position, so both are available for the upload request.

### `Conector xlsx`
Uploads the Excel file to the Internal System for parsing:
- **POST** `{URLbase_stg}/api/serviceorders/readexcelorder`
- Auth: `Bearer {access_token}`
- Body: `multipart/form-data` with key `template file`
- Returns parsed service order data including an HTML representation of the tables

On error → routes to `Basic LLM Chain1` for error summarization → `Send a message1` email notification.

---

## Stage 3 — HTML Table Parsing

### `Estandar Data HTML` (Set node)
Stores the HTML response from the Internal System into the `Data HTML` field for downstream nodes.

### `If2`
Checks if `Data HTML` is non-empty. If empty, routes to dead-end.

### `Extractor de Tablas2` (Code node)
Extracts all `<table>` elements from the HTML body using regex:

```javascript
const html = $('Gmail Trigger2').first().json.html;
const tableMatches = html.match(/<table[\s\S]*?<\/table>/gi) || [];
const tables = tableMatches.map(tableHtml => {
  const rows = [];
  const rowMatches = tableHtml.match(/<tr[\s\S]*?<\/tr>/gi) || [];
  rowMatches.forEach(rowHtml => {
    const cells = [];
    const cellMatches = rowHtml.match(/<(td|th)[\s\S]*?<\/(td|th)>/gi) || [];
    cellMatches.forEach(cell => {
      const text = cell.replace(/<[^>]+>/g, '').replace(/\s+/g, ' ').trim();
      cells.push(text);
    });
    if (cells.length) rows.push(cells);
  });
  return rows;
});
return [{ json: { tables, tablesFound: tables.length } }];
```

Output: `tables[]` (array of parsed rows/cells), `tablesFound` (count).

### `Estandarizador de Columnas` (Code node)
Normalizes and reindexes the extracted table columns for consistent downstream access.

### `Rows / Columnas` (Set node)
Maps the 31 parsed table columns to named fields:

| Field | Description |
|---|---|
| `Asunto` | Email subject |
| `Contacto` | Sender email (from Gmail metadata) |
| `Headers` | Array of table column headers |
| `Datos de Servicio 1–30` | Individual service data arrays (one per column) |

---

## Stage 4 — Service Data Extraction (Parallel)

Multiple **Set** and **Edit Fields** nodes (e.g., `Servicios 1–9`, `Servicio Array 0-1/0-2`, `Edit Fields 0–51`) each extract one `Datos de Servicio X` field from the column map. These run in parallel branches and converge at the `Wait` node.

### `Wait`
Pauses **3 seconds** to allow all parallel extraction branches to complete before proceeding.

### `Estandarizacion de Arrays` (Code node)
Iterates over all items, finds every field that contains an array, and concatenates them into a single unified `services` array:

```javascript
const output = [];
for (const item of items) {
  const source = item.json;
  let foundArrays = [];
  for (const key in source) {
    if (Array.isArray(source[key])) {
      foundArrays = foundArrays.concat(source[key]);
    }
  }
  output.push({ json: { services: foundArrays, totalFound: foundArrays.length } });
}
return output;
```

### `Array Final` (Set node)
Sets the `services` field to the standardized array output.

### `Array Vacio` (If node)
Checks if `services` is non-empty. If empty → dead-end (No-op).

---

## Stage 5 — AI Extraction

### `AI Agent` (GPT-4.1-mini)
Receives the `services` array and the `Headers` field. The system prompt instructs it to:
- Act as an expert in extracting data from service request emails
- Map the values from the `services` array using the table headers as keys
- Output a single structured JSON object conforming to the **Structured Output Parser** schema

**Supporting nodes:**
- `OpenAI Chat Model1` — provides the `gpt-4.1-mini` model
- `Simple Memory` — session memory buffer (key: `gmailparser`, window: 50 tokens)
- `Structured Output Parser` — enforces the output JSON schema (see below)

If the AI Agent errors, the workflow continues via `continueErrorOutput`.

---

## Stage 6 — Validation Against Internal System

### `Auth` node (re-auth)
Re-authenticates with the Internal System to obtain a fresh `access_token`.

### `Conector Validacion Json`
Sends the AI-structured JSON to the Internal System for validation:
- **POST** `{URLbase_stg}/serviceorders/readjsonorder`
- Auth: `Bearer {access_token}`
- Body: Raw JSON from AI Agent output

On success → data is accepted by the internal system (end of flow).
On error → routes to `Basic LLM Chain` for error summarization → `Send a message` email notification.

---

## Stage 7 — Error Notification

### `Basic LLM Chain` / `Basic LLM Chain1` (GPT-5-mini)
When the Internal System returns a validation error, these chains:
1. Parse the error message from the API response
2. Generate a human-readable summary listing missing mandatory fields

### `Send a message` / `Send a message1` (Gmail nodes)
Send an email to the original requester:
- **To:** `Contacto` field (from email metadata)
- **Subject:** `Campos Obligatorios Faltantes` (Missing Mandatory Fields)
- **Body:** LLM-generated summary of the validation errors

---

## Output JSON Schema (Internal System Payload)

```json
{
  "service_order": {
    "customer_id": "string",
    "so_type_id": "string",
    "so_service_id": "string",
    "so_customer_folio": "string",
    "so_requested_date": "YYYY-MM-DD",
    "so_requested_time": "HH:MM:SS"
  },
  "implantador": {
    "email": "string",
    "firstname": "string",
    "midname": "string",
    "lastname": "string",
    "mother_lastname": "string"
  },
  "origin": {
    "loc_type": "string | null",
    "ext_loc_id": "string | null",
    "location_name": "string | null",
    "addr_street": "string | null",
    "addr_extnum": "string | null",
    "addr_intnum": "string | null",
    "neighborhood": "string | null",
    "city": "string | null",
    "county": "string | null",
    "zip_code": "string | null"
  },
  "destination": {
    "loc_type": "string",
    "ext_loc_id": "string",
    "location_name": "string",
    "addr_street": "string",
    "addr_extnum": "string",
    "addr_intnum": "string | null",
    "neighborhood": "string",
    "city": "string",
    "county": "string",
    "zip_code": "string"
  },
  "equipment": [
    {
      "serialNumber": "string",
      "inv_status": "string",
      "inv_type": "string",
      "inv_subtype": "string",
      "inv_brand": "string",
      "inv_model": "string",
      "id_externo": "string",
      "ext_carga": "string",
      "ext_nombre": "string",
      "comments": "string"
    }
  ],
  "requirements": [
    {
      "requirement": "string",
      "comment": "string"
    }
  ]
}
```

---

## External Services & Credentials

| Service | Credential | Purpose |
|---|---|---|
| Gmail OAuth2 | `Cuenta Solicitud Servicios (servicios)` | Email polling + sending notifications |
| OpenAI API | `IDin OpenAI Account` | AI extraction (gpt-4.1-mini, gpt-5-mini) |
| Internal System API (staging) | — | Auth, Excel parsing, JSON validation |

**Internal System base URL:** configured via n8n environment variable

---

## Error Handling Summary

| Scenario | Handler |
|---|---|
| No `.xlsx` attachment in email | Routes to No-op (silent skip) |
| Internal System Excel upload fails | LLM summarizes error → email to sender |
| HTML body is empty after Excel parse | Routes to No-op |
| `services` array is empty after extraction | Routes to No-op |
| Internal System JSON validation fails (missing fields) | LLM summarizes missing fields → email to requester |
| AI Agent errors | `continueErrorOutput` — flow continues |

---

## Notes

- The workflow has an **older prototype** (Gmail Trigger, Extractor de Tablas, AI Agent1) that is disabled. The active path uses `Gmail Trigger2` and related nodes.
- The `Information Extractor` node is an alternative extraction path using a text-based schema — it is wired in but not the primary AI processing node.
- The `Insert row` (DataTable) node is disabled — it was likely used during testing to capture parsed output in a table.
- AI autofix is enabled on the `Structured Output Parser` to handle minor schema deviations from the LLM.
