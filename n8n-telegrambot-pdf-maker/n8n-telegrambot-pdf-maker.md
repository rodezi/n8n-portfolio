# Telegram Bot — Real Estate PDF Sheet Generator

A Telegram bot for real estate agents that collects property photos and text descriptions through a conversation, then generates a professional PDF property sheet using AI and delivers it back in the same chat — no external paid APIs required.

---

## Conversation Flow

```
User sends photo  →  Bot: "📷 Photo 1 saved"
User sends photo  →  Bot: "📷 Photo 2 saved"
User sends text   →  Bot: "✅ Information saved"
User: /generar    →  Bot: [typing...] → sends PDF
```

Session state is stored in **n8n Static Data** keyed by `chat_id`. No external database or Redis needed. Each agent has a fully isolated session.

---

## Workflow Architecture (18 nodes)

```
Telegram Trigger → Message Type (Switch)
                        ├── /generate  → Load Session → Enough Data?
                        │                                  ├── [yes] → typing → AI → Parse HTML → HTML→PDF → Send PDF → Clear Session → Final Message
                        │                                  └── [no]  → No Data Warning
                        ├── photo      → Get Photo File → Save Photo in Session → Confirm Photo Received
                        ├── text       → Save Text in Session → Confirm Text Received
                        └── other      → Welcome Message
```

### Node Reference

| # | Node | Type | Description |
|---|------|------|-------------|
| 1 | **Telegram Trigger** | Telegram Trigger | Listens for all incoming messages |
| 2 | **Tipo de mensaje** | Switch | Routes to 4 branches: `/generar`, photo, free text, or fallback |
| 3 | **Mensaje bienvenida** | Telegram | Sends usage instructions when a command other than `/generar` is received |
| 4 | **Obtener archivo foto** | Telegram `getFile` | Retrieves the `file_path` for the highest-resolution version of the photo |
| 5 | **Guardar foto en sesión** | Code | Builds the Telegram CDN URL and pushes it to `staticData.sessions[chatId].photos[]` |
| 6 | **Confirmar foto recibida** | Telegram | Replies with *"📷 Photo N saved"* |
| 7 | **Guardar texto en sesión** | Code | Appends the message text to `staticData.sessions[chatId].texto` |
| 8 | **Confirmar texto recibido** | Telegram | Replies with *"✅ Information saved. Write /generar when ready"* |
| 9 | **Cargar sesión** | Code | Reads the session for the current `chat_id` from Static Data |
| 10 | **Enough Data?** | IF | Checks that the session is not empty before proceeding |
| 11 | **No Data Warning** | Telegram | Warns the user if `/generate` is called with no data in session |
| 12 | **Sending... (typing)** | Telegram `sendChatAction` | Shows *"uploading document"* indicator while processing |
| 13 | **AI — Structure Data** | Anthropic (Claude Opus 4.6) | Converts unstructured property text into a clean 22-field JSON |
| 14 | **Parse Data + Prepare HTML** | Code | Parses AI JSON, builds the full HTML sheet, and exposes it as a binary `htmlFile` |
| 15 | **HTML → PDF** | HTTP Request | Posts the HTML file to Gotenberg (local Docker) and receives the PDF binary |
| 16 | **Enviar PDF por Telegram** | Telegram `sendDocument` | Sends the PDF to the user with a formatted caption |
| 17 | **Limpiar sesión** | Code | Deletes `staticData.sessions[chatId]` so the agent can start a new sheet |
| 18 | **Mensaje final** | Telegram | Confirms the session has been cleared |

---

## AI Extraction — Structured Fields

The **AI — Structure Data** node sends the session text to **Claude Opus 4.6** and extracts:

| Field | Description | Type |
|-------|-------------|------|
| `titulo` | Attractive property title | string |
| `tipo_propiedad` | Casa / Departamento / Terreno / Local / Oficina / Bodega | string |
| `operacion` | Venta / Renta / Venta o Renta | string |
| `precio` | Formatted price with currency (e.g. `$3,500,000 MXN`) | string |
| `precio_m2` | Price per m² if calculable | string |
| `direccion` | Full address | string |
| `colonia` | Neighborhood or development | string |
| `ciudad` | City | string |
| `estado` | State | string |
| `m2_construidos` | Built area | string |
| `m2_terreno` | Land area | string |
| `recamaras` | Bedrooms | number |
| `banos_completos` | Full bathrooms | number |
| `medios_banos` | Half bathrooms | number |
| `niveles` | Floors | number |
| `estacionamientos` | Parking spaces | number |
| `antiguedad` | Age in years or "Nueva construcción" | string |
| `caracteristicas` | List of highlighted features | string[] |
| `descripcion_larga` | 3–4 sentence marketing description | string |
| `descripcion_ubicacion` | Description of the area and nearby points of interest | string |
| `mantenimiento` | Monthly maintenance fee if applicable | string |
| `amueblado` | Furnished flag | boolean |
| `mascotas` | Pets allowed flag | boolean |

> Prompt rule: if a field is not visible or does not apply, return `null` for strings and `0` for numbers. Claude never invents data.

---

## Session Storage — How Static Data Works

All session state lives in `$getWorkflowStaticData('global')` with this shape:

```json
{
  "sessions": {
    "6900434382": {
      "photos": [
        "https://api.telegram.org/file/bot<TOKEN>/<file_path>",
        "https://api.telegram.org/file/bot<TOKEN>/<file_path>"
      ],
      "texto": "Calle Hidalgo 42, Col. Centro...\n3 recámaras, 2 baños..."
    }
  }
}
```

- Keyed by `chat_id.toString()` — each agent is fully isolated
- Photos are stored as direct Telegram CDN URLs (public, no auth headers needed)
- Text is accumulated across multiple messages with `\n` between them
- On `/generar`, the session is loaded, passed to AI, used to build the PDF, then deleted

---

## PDF Generation — Gotenberg (Free, Self-Hosted)

The **HTML → PDF** node posts the HTML file as `multipart/form-data` to Gotenberg's Chromium endpoint:

```
POST http://localhost:3000/forms/chromium/convert/html
Content-Type: multipart/form-data
Field: files = index.html (binary)
```

Gotenberg renders it with a headless Chromium engine — same output quality as a paid API, with no usage limits.

### Setup (single command)

```bash
docker run -d --name gotenberg -p 3000:3000 gotenberg/gotenberg:8
```

> If n8n also runs in Docker, add both containers to the same network and use `http://gotenberg:3000` instead of `http://localhost:3000`.

---

## PDF Sheet Design

The generated PDF is a single A4 page with a professional real estate layout:

| Section | Content |
|---------|---------|
| **Header** | Dark navy + gold branding bar with operation badge (VENTA / RENTA) |
| **Hero** | Property type badge, title, address, price, price/m² |
| **Photos** | Adaptive grid — 1, 2, or 3 columns depending on photo count (max 6) |
| **Two-column block** | Left: specs table / Right: features list + area description |
| **Description** | Full marketing paragraph |
| **Address** | Full address block |
| **Footer** | Agency contact info + generation date |

---

## Required Credentials

| Credential | Node | Type |
|------------|------|------|
| `Telegram Bot` | All Telegram nodes | Telegram API (`TELEGRAM_CRED_ID`) |
| `Anthropic account` | AI — Structure Data | Anthropic API (`L6uWpsm4R1xYB6f9`) |

---

## Variables to Replace Before Activating

| Variable | Node | Description |
|----------|------|-------------|
| `TELEGRAM_CRED_ID` | All Telegram nodes | Your n8n Telegram credential ID |
| `TU_BOT_TOKEN` | Guardar foto en sesión (Code) | Raw Telegram bot token for building CDN URLs |
| Gotenberg URL | HTML → PDF | Change to `http://gotenberg:3000` if using Docker networking |

> **Security note:** `TU_BOT_TOKEN` is hardcoded in the Code node. Replace it with `{{ $env.TELEGRAM_BOT_TOKEN }}` and set the variable in your n8n environment to avoid exposing the token in the workflow JSON.

---

## Use Cases

### Case 1 — Full flow
1. Agent sends 1–6 photos of the property
2. Agent sends one or more text messages with property details (can be unstructured)
3. Agent sends `/generar`
4. Bot shows typing indicator while Claude Opus 4.6 structures the data
5. Gotenberg renders the HTML into a PDF
6. Bot sends the PDF with a summary caption
7. Session is cleared — agent can start a new sheet immediately

### Case 2 — `/generar` with no session data
The bot replies: *"⚠️ No tengo fotos ni información guardada..."* and does not proceed.

### Case 3 — Any other command or unrecognized input
The bot replies with the full usage instructions.

---

## Technical Notes

- Photos are stored as URLs, not binaries — keeping Static Data lightweight
- The HTML uses inline CSS only (no external fonts or CDN) so Gotenberg renders it correctly without network dependencies
- Photo grid columns are determined dynamically: 1 photo → 1 col, 2 photos → 2 col, 3+ photos → 3 col
- `active: false` — the workflow must be manually activated in the n8n panel
- Static Data is not thread-safe for simultaneous writes on the exact same `chat_id`, but this is negligible for individual agent use
