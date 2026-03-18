# Operation System — Field Service Work Order Automation

## Overview

This n8n workflow automates the full lifecycle of field service work orders via **WhatsApp** and **AI-powered evidence validation**. A field technician initiates a work order by sending its number over WhatsApp; the system validates the order and the user's authorization, then guides the technician through each pending activity — collecting photo or text evidence, validating it via **GPT-4o vision** or a human **agent**, uploading results to **Zoho FSM**, and recording completion status in the **Internal System**. Once all activities are done, a final PDF report is distributed to the operational leader and dispatcher.

---

## Architecture Summary

```
WhatsApp Webhook (order number)
        ↓
  Validate message → Validate order → Validate user authorization
        ↓
  Confirm order start (WhatsApp template + form)
        ↓
  Internal System → Fetch current activity
        ↓
  Pending activity?
  ├─ NO  → Send "Service complete" → Distribute PDF report
  └─ YES → Validate Zoho appointment → Route by activity type
                ↓
          Can proceed?
          ├─ NO  → Collect reason (VF/SNR) → Terminate appointment → Record in Internal System
          └─ YES → Evidence type?
                   ├─ PHOTO + AI validation  → Request photo → GPT-4o validates (up to 3 attempts)
                   ├─ PHOTO + Agent validation → Forward photo to agent → Agent approves/rejects
                   └─ TEXT  → Request text response
                        ↓
                   Upload evidence to Zoho FSM
                        ↓
                   Record completion in Internal System
                        ↓
                   Wait 10s → Loop to next activity
```

---

## Trigger

**Node:** `Trigger Inicio` (Webhook — HTTP POST)
- Receives incoming WhatsApp messages
- Payload: sender phone number, message body (order number), phone number ID
- Entry point for all order executions

---

## Phase 1 — Order Validation & Authorization

### Message Validation
**`Message Body`** (IF node) — filters out button-type messages; only plain text proceeds.

**`Mensaje debe ser Numero`** (Code node) — validates the message body is a numeric value:
```javascript
const abc = !Number.isNaN(Number($('Aggregate1').first().json.body[0].messages[0].text.body));
return [{ json: { isNumber: abc } }];
```
If not numeric → **`Orden Invalida`** (WhatsApp): *"El numero de orden no es valido. Inténtalo de nuevo en 1 minuto."*

### Order Lookup
- Authenticates with the **Internal System** and fetches the order's main participants (dispatcher, operational leader, technician).
- **`Order Not Found`** (IF): if the order does not exist → **`Orden No Existe`** (WhatsApp): *"El numero de orden proporcionado no existe."*

### Authorization Check
**`User Not Authorized`** (IF) — verifies the initiating phone number matches either the registered agent or responsible party (last 10 digits):
```
$json['Telefono Agente'].indexOf($json['Telefono Persona Inicia Orden'].slice(-10)) != -1
OR
$json['Telefono Responsable'].indexOf($json['Telefono Persona Inicia Orden'].slice(-10)) != -1
```
- Unauthorized → **`No Autorizado Para Iniciar Ejecucion`** (WhatsApp)
- Invalid phone format → **`Numero Invalido de Agente o Responsable`** (WhatsApp) — includes the invalid numbers for review

### Order Confirmation
**`Template Inicio de Ejecucion`** — sends a WhatsApp template message to the *other* participant (if agent initiated → notifies responsible, and vice versa).

**`Mensaje de Inicio de Orden`** (WhatsApp `sendAndWait`) — asks the initiator to confirm:
> *"¿Es correcta la orden? Se iniciará la orden: [number]. Responde: Si / No"*
- **Timeout:** 180 minutes
- **No** → `Falso Inicio`: *"Intente de nuevo cuando esté listo."*
- **Yes** → proceeds to execution

---

## Phase 2 — Activity Retrieval

- Re-authenticates with the **Internal System**
- **`SAI Obtener Ordenes Principal`** — fetches current execution info:
  - `GET /serviceorders/{id}/executioninfo?current_act=1`
  - Returns activity description, evidence parameters, evidence format (Ph/Tx), FSM IDs, validation mode flag, technician/agent contact info

**`Datos Estandar`** (Set node) — normalizes all retrieved fields into a single standardized object used throughout the workflow:

| Field | Description |
|---|---|
| `Numero de orden` | Work order ID |
| `Descripcion de Actividad` | Activity description |
| `Validacion - Evidencia` | What the evidence must show |
| `Tipo de Actividad` | Activity type (G = General, etc.) |
| `Formato de evidencia` | `Ph` (photo) or `Tx` (text) |
| `Validacion IA - Agente` | `1` = AI validates, `0` = Agent validates |
| `Teléfono` / `Teléfono Agente` | Technician and agent phone numbers |
| `inv_id`, `activity_id`, `act_evid_id` | Internal System record IDs |
| `FSM_job_sheet_id`, `FSM_field_api_name` | Zoho FSM references |
| `Actividad por realizar` | `true` if pending activities remain |

**`Hay Actividad por realizar?`** (IF):
- **No pending activities** → Phase 6 (service completion)
- **Activities pending** → Phase 3

---

## Phase 3 — Appointment & Activity Routing

### Zoho Appointment Check
**`Validacion Cita Zoho`** — queries the FSM integration layer with the job sheet ID to retrieve appointment and service line item identifiers.

**`Cita Iniciada Zoho`** (IF):
- Not started → **`Iniciar cita Zoho`**: POSTs to FSM layer to start the appointment
- Already started → proceeds

### Activity Type Routing
**`Actividad General y FSM Field ID`** (IF):
- `Tipo de Actividad == "G"` AND `FSM_fileID_text` is non-empty → direct evidence upload path (Phase 5b)
- Otherwise → technician decision prompt

**`Msj Al Tecnico con descripcion de actividad`** (WhatsApp `sendAndWait`):
> *"La actividad es de tipo: [type]. Descripcion: [desc]. Validacion: [params]. ¿Se puede proceder? Si / No"*
- **Timeout:** 180 minutes

**`Se puede trabajar en la actividad?`** (IF):
- **No** → Phase 4 (cannot proceed flow)
- **Yes** → Phase 5 (evidence collection)

---

## Phase 4 — Cannot Proceed Flow

**`Razon de NO proceder`** (WhatsApp `sendAndWait`) — collects:
- **Reason:** `VF` (False Visit) or `SNR` (Service Not Rendered)
- **Motivo:** free-text explanation

Then:
1. **`Terminar Cita Zoho`** — POSTs termination to FSM with reason + notes
2. **Internal System Auth** → **`SAI Registro de actividades`** — records non-completion status
3. **`Status registrado`** (WhatsApp): *"Actividad Registrada. Si existe actividad pendiente se te asignara en un momento."*
4. **`Wait`** (10 sec) → loops back to Phase 2

---

## Phase 5 — Evidence Collection & Validation

### 5a — Photo Evidence with AI Validation (`Validacion IA - Agente == 1`, `Formato == Ph`)

**Evidence request chain (up to 3 attempts):**

| Attempt | Request node | Validation node | On failure |
|---|---|---|---|
| 1 | `Msj descripcion y solicitud de evidencia 1` | `IA - Validacion` | Retry message 1 |
| 2 | `Msj al tecnico evidencia...1` | `IA - Validacion6` | Retry message 3 |
| 3 | `Msj al tecnico evidencia...3` | `IA - Validacion7` | Final retry message 2 |

**GPT-4o system prompt:**
```
Eres un asistente experto analizando imagenes. Tu rol es analizar la siguiente imagen
y validar que efectivamente cumple con los parametros en: [Validacion - Evidencia].
Analiza correctamente los parametros y que la imagen cumpla con esos parametros.
Las imagenes deben ser claras y con buen enfoque, de otra forma no las valides.
Tu output debe ser solo: Valido / No Valido
```

**`Validacion Foto`** (IF): `content == "Valido"`
- **Invalid** → re-request evidence
- **Valid** → download binary image → **`Evidencia Zoho IMG`** (upload to Zoho job sheet)

After successful upload:
1. **`Cerrar Cita Zoho`** — closes appointment in FSM
2. **Internal System Auth** → **`SAI Registro de actividades1`** — records completion (`act_evid_status: F`)
3. **`Validacion exitosa1`** (WhatsApp): *"Validación exitosa. Si existe actividad pendiente se te asignará en un momento."*
4. **`Wait`** (10 sec) → loops back to Phase 2

---

### 5b — Photo Evidence with Agent Validation (`Validacion IA - Agente == 0`, `Formato == Ph`)

1. Technician submits photo via WhatsApp form
2. **`Upload media`** → **`Download media1`** → gets WhatsApp media ID + URL
3. **`Frwrd Evidencia Agente`** — forwards photo directly to agent's WhatsApp number
4. **`Msj a Agente para validar evidencia`** (WhatsApp `sendAndWait`) — asks agent:
   > *"Valida la evidencia del técnico. Actividad: [desc]. Debe contener: [params]. ¿Es correcta? Si / No + comentario opcional"*
   - **Timeout:** 180 minutes

**`actividad valida? (Si/no)`** (IF):
- **No** → **`Msj al tecnico / motivo de no validacion`** — shows agent's rejection comment, requests new evidence → loops back
- **Yes** → download image → **`Evidencia Zoho IMG1`** → **`Cerrar Cita Zoho2`** → **Internal System** records completion → success message → Wait → loop

---

### 5c — Text Evidence (`Formato == Tx`)

1. **`Mensaje con descripcion de actividad y solicitud de evidencia al tecnico4`** (WhatsApp `sendAndWait`) — requests text input
2. **`Documentacion Validacion Tx`** — captures the text response
3. **`Evidencia Zoho Txt`** (HTTP) — POSTs text to Zoho FSM job sheet
4. **`Cerrar Cita Zoho1`** — closes appointment
5. **Internal System Auth** → **`SAI Registro de actividades3`** — records completion
6. **`Validacion exitosa`** (WhatsApp) — success notification
7. **`Wait`** → loops back to Phase 2

---

### 5d — General Activity with FSM File ID (direct path)

When `Tipo de Actividad == "G"` and `FSM_fileID_text` is already populated:
- **Photo**: `Zoho Jobsheet ID` → `Cerrar Cita Zoho3` → Internal System records (`SAI Registro de actividades4`) → Wait
- **Text**: `Evidencia Zoho Txt2` → `Cerrar Cita Zoho4` → Internal System records (`SAI Registro de actividades5`) → Wait

---

## Phase 6 — Service Completion & Reporting

When no more activities remain:

1. **`Servicio Terminado Responsable`** (WhatsApp) → notifies operational leader:
   *"No existen actividades pendientes para la OS [#]. Servicio terminado."*
2. **`Servicio Terminado Agente`** (WhatsApp) → notifies dispatcher with same message
3. **Internal System Auth** → **`SAI Obtener Ordenes Principal1`** — fetches final execution info including a Base64-encoded PDF report
4. **`Obtiene el PDF → Base64 - PDF`** (Code node) — decodes and converts to binary:
```javascript
const cleanBase64 = base64String
  .replace(/^data:application\/pdf;base64,/, "")
  .replace(/\s/g, "");
const buffer = Buffer.from(cleanBase64, 'base64');
const binaryData = await this.helpers.prepareBinaryData(
  buffer, 'reporte_ejecucion_final.pdf', 'application/pdf'
);
return { json: { resultado: "Archivo listo" }, binary: { data: binaryData } };
```
5. **`PDF Reporte Responsable`** (WhatsApp) — sends PDF document to operational leader
6. **`PDF Reporte Agente`** (WhatsApp) — sends PDF document to dispatcher

---

## Internal System Integration

All interactions with the Internal System follow the same auth + action pattern:

### Authentication
- **POST** `/api/login` → returns `access_token`
- Multiple dedicated auth nodes (1–9) are used at each phase to ensure a fresh token

### Key Endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/serviceorders/{id}/executioninfo?current_act=1` | GET | Fetch current activity and order details |
| `/api/serviceorders/{id}/mainparticipants` | GET | Get agent, responsible, and technician info |
| `/api/serviceorders/{id}/updatemonitorevidence` | POST | Record activity completion or non-completion |

### Activity Status Payload (`updatemonitorevidence`)
```json
{
  "inv_id": "...",
  "service_id": "...",
  "activity_id": "...",
  "act_evid_id": "...",
  "act_evid_status": "F",
  "FSM_fileID_text": "...",
  "comment": "..."
}
```
`act_evid_status: "F"` marks the activity as finalized.

---

## Zoho FSM Integration

Evidence and appointment lifecycle are managed through a dedicated FSM integration layer (webhook-based):

| Action | Trigger |
|---|---|
| Start appointment | First activity is about to be executed |
| Upload image evidence | Photo validated and approved |
| Upload text evidence | Text response collected |
| Create jobsheet ID reference | General activity with existing file ID |
| Close appointment | Activity completed successfully |
| Terminate appointment | Activity cannot proceed (VF/SNR) — includes notes |

---

## WhatsApp Interaction Summary

| Message | Recipient | Type | Wait? |
|---|---|---|---|
| Order confirmation | Initiator | Form (Si/No dropdown) | Yes — 3h |
| Template notification | Other participant | Template | No |
| Activity description + proceed? | Technician | Form (Si/No) | Yes — 3h |
| Photo evidence request | Technician | Form (file upload) | Yes — 3h |
| Text evidence request | Technician | Form (text input) | Yes — 3h |
| Cannot proceed — reason | Technician | Form (dropdown + text) | Yes — 3h |
| Forward evidence | Agent | Image (media ID) | No |
| Agent validation request | Agent | Form (Si/No + comment) | Yes — 3h |
| Rejection reason → technician | Technician | Form (file re-upload) | Yes — 3h |
| Activity registered | Technician | Text | No |
| Validation successful | Technician | Text | No |
| Service complete | Responsible + Agent | Text | No |
| Final PDF report | Responsible + Agent | Document | No |

---

## Evidence Validation Decision Tree

```
Evidence type?
├─ PHOTO
│   └─ Validation mode?
│       ├─ AI (GPT-4o)
│       │   ├─ Attempt 1 → Valido? → proceed
│       │   ├─ Attempt 2 → Valido? → proceed
│       │   └─ Attempt 3 → Valido? → proceed / escalate
│       └─ Agent
│           ├─ Forward photo to agent
│           ├─ Agent: Si → proceed
│           └─ Agent: No → show comment → technician re-submits → loop
└─ TEXT
    └─ Collect text → upload → proceed
```

---

## External Services & Credentials

| Service | Purpose |
|---|---|
| WhatsApp Business API | All technician and agent communication |
| OpenAI GPT-4o | AI-powered photo evidence validation |
| Internal System API | Work order data, activity status recording |
| Zoho FSM (via integration layer) | Appointment lifecycle and job sheet evidence |

---

## Error Handling

| Scenario | Behavior |
|---|---|
| Non-numeric order number | WhatsApp error message → stop |
| Order not found in Internal System | WhatsApp error message → stop |
| Unauthorized initiator | WhatsApp error message → stop |
| Invalid agent/responsible phone | WhatsApp notification with details → stop |
| User cancels order start | *"Intente de nuevo cuando esté listo."* → stop |
| AI photo validation fails (attempt 1–2) | Re-request evidence with retry message |
| Agent rejects evidence | Agent's comment forwarded to technician → re-submit loop |
| Cannot proceed with activity | Reason + notes recorded → appointment terminated → loop continues |
| Zoho close fails | `continueErrorOutput` — flow proceeds regardless |
| Internal System order fetch fails | `continueRegularOutput` — flow proceeds |
| WhatsApp sendAndWait timeout | 180 min — no automatic retry |

---

## Notes

- The workflow loops continuously — after each activity completes, it waits 10 seconds and fetches the next pending activity until none remain.
- AI validation uses **GPT-4o** with a dynamic prompt that injects the specific evidence requirements per activity, ensuring context-aware validation.
- A disabled email reporting path exists (Gmail) — the active delivery method is WhatsApp PDF.
- The `Datos Estandar` Set node is the single source of truth for all activity data throughout the flow — all downstream nodes reference it by name.
- The workflow supports up to 9 parallel Internal System authentication nodes to maintain fresh tokens across all branches.
