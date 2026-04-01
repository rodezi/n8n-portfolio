# TelegramBot — Invoice Parser

Telegram bot that receives photos or PDFs of Mexican invoices (CFDI), extracts fiscal data using AI, and automatically records it in Google Sheets.

**Bot:** t.me/realtek_facturabot

---

## Workflow

```
Send Photo → Wait → Has attachment? ──► Analyze Image → Extractor → Sheets — Register → Confirmation OK
                                    └──► No attachment warning
```

### Nodes

| # | Node | Type | Description |
|---|------|------|-------------|
| 1 | **Send Photo** | Telegram Trigger | Listens for incoming messages and automatically downloads attachments |
| 2 | **Wait** | Wait | Brief pause before processing to stabilize file download |
| 3 | **Has attachment?** | IF | Checks if the message contains a `document` or `photo`. If not, routes to the warning |
| 4 | **No attachment warning** | Telegram | Replies to the user: *"Please send an image or PDF of the invoice"* |
| 5 | **Analyze Image** | Anthropic (Claude Sonnet 4.6) | Visually analyzes the invoice and returns a JSON with all CFDI fields |
| 6 | **Extractor** | Information Extractor | Parses the JSON returned by Claude using OpenRouter (GPT-5.1) as the structured language model |
| 7 | **Sheets — Register** | Google Sheets | `appendOrUpdate` into the *Facturas recibidas* sheet of the *Facturas Template* spreadsheet |
| 8 | **Telegram — Confirmation OK** | Telegram | Replies to the user: *"✅ Invoice successfully registered"* |

---

## Extracted Fields (CFDI)

The **Analyze Image** node instructs Claude to extract the following fields from the fiscal document:

| Field | Description | Type |
|-------|-------------|------|
| `rfc_emisor` | Issuer RFC | string |
| `razon_social_emisor` | Issuer name or business name | string |
| `regimen_fiscal_emisor` | Fiscal regime code (e.g. `601`) | string |
| `folio_fiscal_uuid` | CFDI UUID (36 characters) | string |
| `serie_folio` | Series and folio number (e.g. `A-1042`) | string |
| `fecha_emision` | Issue date in `YYYY-MM-DD` format | string |
| `uso_cfdi` | CFDI use code and description (e.g. `G03 - General expenses`) | string |
| `forma_pago` | Payment form code and description (e.g. `03 - Transfer`) | string |
| `metodo_pago` | `PUE` or `PPD` | string |
| `moneda` | Invoice currency (usually `MXN`) | string |
| `descripcion_concepto` | Description of the product or service | string |
| `clave_sat` | SAT product/service code | string |
| `cantidad` | Number of units | number |
| `unidad` | Unit of measure (e.g. `PZA`, `SER`, `E48`) | string |
| `subtotal` | Subtotal before taxes | number |
| `iva_16` | 16% VAT amount | number |
| `isr_retenido` | Withheld ISR (0 if not applicable) | number |
| `iva_retenido` | Withheld VAT (0 if not applicable) | number |
| `total` | Total amount including taxes | number |

> Prompt rule: if a field is not visible or does not apply, return `null` for strings and `0` for numbers. Never invent data.

---

## Google Sheets — Sheet Structure

**Spreadsheet:** Facturas Template
**Sheet:** Facturas recibidas

| Column | Source | Description |
|--------|--------|-------------|
| `id_registro` | Manual | Unique record identifier |
| `fecha_procesamiento` | `$today` (n8n) | Date the invoice was processed |
| `archivo_original` | — | Original filename (field pending mapping) |
| `enviado_por` | Telegram `first_name + last_name` | Name of the user who sent the invoice |
| `rfc_emisor` | AI | Issuer RFC |
| `razon_social_emisor` | AI | Issuer business name |
| `regimen_fiscal_emisor` | AI | Fiscal regime |
| `folio_fiscal_uuid` | AI | CFDI UUID |
| `serie_folio` | AI | Series and folio |
| `fecha_emision` | AI | CFDI issue date |
| `uso_cfdi` | AI | CFDI use |
| `forma_pago` | AI | Payment form |
| `metodo_pago` | — | Payment method (field pending mapping) |
| `moneda` | AI | Currency |
| `descripcion_concepto` | AI | Concept description |
| `clave_sat` | AI | SAT code |
| `cantidad` | AI | Quantity |
| `unidad` | AI | Unit of measure |
| `subtotal` | AI | Subtotal |
| `iva_16` | AI | 16% VAT |
| `isr_retenido` | AI | Withheld ISR |
| `iva_retenido` | AI | Withheld VAT |
| `total` | AI | Total |
| `categoria_gasto` | Manual | Expense category (filled manually) |
| `estatus` | Manual | Invoice status (e.g. pending, approved) |
| `notas_validacion` | Manual | Accounting validation notes |
| `pagada` | Manual | Whether the invoice has been paid |
| `fecha_pago` | AI / Manual | Payment date (`fecha_emision` as initial value) |

---

## Required Credentials

| Credential | Used by | Type |
|------------|---------|------|
| `Telegram Bot Facturacion` | Send Photo, No attachment warning, Confirmation OK | Telegram API |
| `Anthropic account` | Analyze Image | Anthropic API |
| `OpenRouter account` | OpenRouter Chat Model (Extractor) | OpenRouter API |
| `Google Sheets account` | Sheets — Register | Google Sheets OAuth2 |

---

## Use Cases

### Case 1 — Valid image/PDF received
1. User sends a photo or PDF of an invoice to the bot
2. The bot downloads the file and waits
3. Claude Sonnet 4.6 visually analyzes the document
4. The Extractor structures the data into typed fields
5. The data is recorded in Google Sheets
6. The bot replies: `✅ Invoice successfully registered`

### Case 2 — Message without attachment
1. User sends a text message or other content without an attachment
2. The bot replies: `⚠️ Please send an image or PDF of the invoice.`

---

## Technical Notes

- The **Analyze Image** node uses `inputType: binary`, processing the file downloaded directly by the Telegram trigger.
- The `metodo_pago` field in Sheets is mapped with an empty value (`"="`) — pending connection to the Extractor output.
- `matchingColumns` for the Sheets upsert is set to `folio_fiscal_uuid` as the unique CFDI key, preventing duplicate records.
- The workflow runs in `binaryMode: separate` for efficient image file handling.
- `active: false` — the workflow must be manually activated in the n8n panel.
