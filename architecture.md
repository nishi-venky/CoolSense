# Smart AC Failure Intelligence & Predictive Maintenance — Architecture

**Audience:** backend/frontend/ML/analytics teams + slide deck source material.
**Status:** MVP architecture, implemented and smoke-tested. Values marked
`CONFIGURATION_PENDING` are placeholders awaiting the owning team.

---

## Pre-flight: contradictions, assumptions, missing contracts

Per the brief's own instruction (Section 35), before generating the
architecture these were resolved explicitly:

- **Contradiction resolved:** the prompt both asks the backend to treat ML/
  Azure output schemas as "not yet frozen" *and* gives a specific example JSON
  shape. Resolution: the example shape is implemented as the *current default*
  but every ML/OCR/sensor input passes through an **adapter** (`ml_adapter.py`,
  `ocr_adapter.py`, `sensor_adapter.py`) so only the adapter needs to change
  when the real schema arrives — nothing else in the system depends on it.
- **Contradiction resolved:** the prompt's own ML section describes a
  supervised failure_mode/component/department classifier with a 70/15/15
  split, while the final override says to ship a KMeans **clustering**
  placeholder. Resolution: both are documented — the supervised contract is
  what `ml_adapter.py` / `POST .../ml-result` expects to receive eventually;
  the KMeans script is shipped as the literal fallback the override asked for,
  and is clearly labeled "plan B, not the primary pipeline."
- **Assumption:** `record_id` in the dataset maps to the model's `record`
  input field (the CSV was authored before the interface field was finalized).
- **Assumption:** "technician diagnosis" in the DB/API maps to three separate
  optional fields (`technician_diagnosis_failure_mode/component/department`)
  rather than one free-text field, so it can be structurally compared to the
  AI prediction per Section 30.
- **Missing contract:** final ML JSON schema, final OCR JSON schema, Azure
  deployment name/API version, Gemini response format specifics, Power BI
  connection method (assumed: Web/REST connector against the `/powerbi/*`
  endpoints) — all listed in **OPEN CONTRACTS** at the end of this document.

---

## PART 1 — Executive Architecture

```
                         ┌─────────────────────┐
                         │   React Frontend     │
                         │ (Technician / Mgmt / │
                         │  Quality / Design)    │
                         └──────────┬───────────┘
                                    │ REST / JSON (Bearer JWT)
                                    ▼
                         ┌─────────────────────┐
                         │      FastAPI          │◄──────────────┐
                         │  (single integration   │                │
                         │   layer / modular       │                │ Gemini API
                         │   monolith)              │───────────────┘
                         └──────────┬───────────┘
                    ┌───────────────┼────────────────┬─────────────┐
                    ▼               ▼                ▼             ▼
           ┌───────────────┐ ┌────────────┐  ┌──────────────┐ ┌─────────┐
           │  PostgreSQL    │ │ Azure OpenAI│  │ Azure OpenAI  │ │ Local/  │
           │ (system of     │ │ (chat/       │  │ Embeddings    │ │ object  │
           │  record)       │ │ classification)│  │ (clustering) │ │ storage │
           └───────┬────────┘ └────────────┘  └──────────────┘ └─────────┘
                    │
                    ▼
           ┌───────────────┐
           │   Power BI     │
           │ (Reliability,  │
           │  Failure Mode  │
           │  Mining,        │
           │  Predictive     │
           │  Maintenance)   │
           └───────────────┘
```

**Predictive-maintenance / digital-twin pipeline** (separate flow, converges
back into the same service-ticket workflow):

```
Python Sensor Simulator  →  FastAPI (/sensors/readings)  →  PredictiveService
      (no IoT Hub)             (direct POST, no queue)      (health/anomaly score)
                                                                     │
                                        threshold crossed? ──────── ┤
                                                                     ▼
                                                          PreventiveTicket (PENDING)
                                                                     │
                                                     technician acknowledges/links
                                                                     ▼
                                                       normal ServiceTicket workflow
```

---

## PART 2 — Component Responsibilities

| Component | Responsibility | Input | Output | Owner |
|---|---|---|---|---|
| Frontend (React) | Auth UI, device lookup, ticket entry, image capture, dashboards, chat UI | User actions | REST calls to FastAPI | Frontend team |
| Backend (FastAPI) | Business logic, RBAC, orchestration, validation, error taxonomy | REST requests | JSON envelopes | Backend team (you) |
| Database (PostgreSQL) | System of record, normalized schema | SQL via SQLAlchemy | Rows | Backend team |
| ML (KMeans placeholder / future model) | failure_mode/component/department prediction or clustering | record, date, product_model, serial_range, fix_text, symptom_text | Prediction/cluster JSON | ML team |
| Azure OpenAI | Service-note classification, suggested actions, cluster labeling | Sanitized service note | JSON prediction | Azure team |
| Azure OpenAI Embeddings | Semantic vectors for clustering | Sanitized service note | Embedding vector (internal only) | Azure team |
| OCR (analytics/OCR component) | Extract text from technician note images | Image | OCR JSON | Analytics/OCR team |
| Analytics | Business-question answering, dashboards | DB via analytics endpoints | Aggregated JSON | Analytics team |
| Power BI | Visualization/reporting only, never executes AI | `/powerbi/*` datasets | Dashboards | Manufacturer/BI team |
| Gemini | Technician chatbot | Message + context | Chat reply | External (Gemini) |
| Sensor Simulator | Generates realistic virtual AC telemetry | — | Sensor JSON | Backend/predictive team |
| Predictive Model (MVP heuristic) | Health/anomaly scoring, preventive-ticket triggering | Sensor readings | health_score, anomaly_score, predicted_issue | Backend team (swappable later) |

---

## PART 3 — Database Architecture

### Tables (all in `app/models/`)

**users** *(contains PII — excluded from ML/analytics/Power BI)*
`id PK · username UNIQUE · email · hashed_password · role ENUM · technician_name · phone_number · is_active · created_at`

**devices**
`id PK · device_id UNIQUE INDEX · product_model INDEX · serial_range INDEX · manufacturing_date · status · created_at`

**service_tickets**
`id PK · ticket_id UNIQUE INDEX · device_id FK→devices · technician_id FK→users · date · symptom_text · fix_text · technician_diagnosis_failure_mode/component/department (RAW) · severity · ground_truth_failure_mode/component/department (GROUND TRUTH, nullable until confirmed) · status ENUM · created_at · updated_at`

**service_ticket_images**
`id PK · ticket_id FK→service_tickets · file_path (storage key) · file_name · content_type · file_size_bytes · ocr_text · ocr_raw_json · ocr_submitted_at · created_at`

**ai_analysis_results** *(AI PREDICTION — separate from ground truth)*
`id PK · ticket_id FK→service_tickets · model_name · model_version · prediction_timestamp · predicted_failure_mode/component/department · confidence · suggested_action · cluster_id FK→clusters · raw_result_json (audit only) · normalized_result_json · created_at`

**failure_modes / components / departments** — lookup/reference tables, `id PK, name UNIQUE`.

**clusters**
`id PK · cluster_label · cluster_id_source (DATASET|GENERATED) · description`

**sensor_readings**
`id PK · device_id FK→devices INDEX · timestamp INDEX · temperature · compressor_current · vibration · fan_speed · power_consumption · refrigerant_pressure · humidity`

**device_health**
`id PK · device_id FK→devices INDEX · timestamp INDEX · health_score (0-100) · anomaly_score (0-1) · status ENUM`

**anomalies**
`id PK · device_id FK→devices INDEX · detected_at INDEX · anomaly_score · description · resolved`

**predictive_events**
`id PK · device_id FK→devices INDEX · timestamp INDEX · predicted_issue · confidence · risk_level · actual_outcome (CONFIRMED|FALSE_POSITIVE|PENDING)`

**preventive_tickets**
`id PK · device_id FK→devices · predictive_event_id FK→predictive_events · status ENUM · created_at · acknowledged_by FK→users · acknowledged_at · linked_service_ticket_id FK→service_tickets`

**audit_logs**
`id PK · request_id INDEX · user_id INDEX · timestamp INDEX · action INDEX · resource · result · model_version · ai_service_used · extra JSON`

### Relationships
- `devices 1—N service_tickets`, `service_tickets 1—N service_ticket_images`, `service_tickets 1—N ai_analysis_results` (one ticket can be re-analyzed).
- `devices 1—N sensor_readings/device_health/anomalies/predictive_events`.
- `predictive_events 1—N preventive_tickets`; `preventive_tickets N—1 service_tickets` (link-back).

### Privacy separation
- **PII lives only in `users`** (technician_name, phone_number, email). No other table stores PII.
- Analytics and Power BI endpoints (`/analytics/*`, `/powerbi/*`) never join against PII columns — verified in code (see `powerbi.py` comments).
- ML inputs (`record, date, product_model, serial_range, fix_text, symptom_text`) contain **zero PII**.

### Indexing
Indexes on all FK columns used in joins/filters, plus `device_id`, `ticket_id`, `username`, `timestamp` columns used in the analytics queries and time-series lookups (sensor readings, health, anomalies).

---

## PART 4 — API Catalog

Base path: `/api/v1`. Full request/response JSON in Part 5. All protected
routes require `Authorization: Bearer <access_token>`.

| ID | Method | Path | Purpose | Role(s) | Auth | Sync/Async | DB Tables | External Service |
|---|---|---|---|---|---|---|---|---|
| AUTH-1 | POST | /auth/login | Obtain access+refresh token | Any | No | Sync | users | — |
| AUTH-2 | POST | /auth/refresh | Rotate access token | Any (valid refresh) | No | Sync | users | — |
| AUTH-3 | GET | /auth/me | Current user profile | Any | Yes | Sync | users | — |
| AUTH-4 | POST | /auth/logout | Client-side token discard | Any | Yes | Sync | — | — |
| USR-1 | POST | /users | Create user | ADMIN | Yes | Sync | users | — |
| USR-2 | GET | /users | List users | ADMIN | Yes | Sync | users | — |
| DEV-1 | POST | /devices/lookup | Resolve barcode/identifier → device | Any authenticated | Yes | Sync | devices, service_tickets | — |
| DEV-2 | POST | /devices | Register device | ADMIN | Yes | Sync | devices | — |
| DEV-3 | GET | /devices/{device_id} | Get device | Any authenticated | Yes | Sync | devices | — |
| TKT-1 | POST | /service-tickets | Create service ticket (raw data) | TECHNICIAN | Yes | Sync | service_tickets | — |
| TKT-2 | GET | /service-tickets/{ticket_id} | Get ticket | Any authenticated | Yes | Sync | service_tickets | — |
| TKT-3 | POST | /service-tickets/{ticket_id}/complete | Record ground truth, close ticket | TECHNICIAN, QUALITY | Yes | Sync | service_tickets | — |
| IMG-1 | POST | /service-tickets/{ticket_id}/images | Upload image | TECHNICIAN | Yes | Sync | service_ticket_images | Local storage |
| IMG-2 | GET | /service-tickets/{ticket_id}/images | List images | Any authenticated | Yes | Sync | service_ticket_images | — |
| OCR-1 | POST | /images/{image_id}/ocr | Submit OCR JSON for an image | TECHNICIAN, ADMIN | Yes | Sync | service_ticket_images | OCR team (external submission) |
| AI-1 | POST | /service-tickets/{ticket_id}/analyze | Run Azure OpenAI classification | TECHNICIAN | Yes | Sync | ai_analysis_results, service_tickets | Azure OpenAI |
| AI-2 | POST | /service-tickets/{ticket_id}/ml-result | Push external ML prediction (adapter) | ADMIN | Yes | Sync | ai_analysis_results, service_tickets | ML pipeline (pushed in) |
| AI-3 | GET | /service-tickets/{ticket_id}/ai-results | List AI predictions for ticket | Any authenticated | Yes | Sync | ai_analysis_results | — |
| EMB-1 | POST | /service-tickets/{ticket_id}/embed | Generate embedding (internal) | ADMIN | Yes | Sync | service_tickets | Azure OpenAI Embeddings |
| CMP-1 | GET | /service-tickets/{ticket_id}/comparison | Technician vs AI comparison | Any authenticated | Yes | Sync | service_tickets, ai_analysis_results | — |
| DA-1 | GET | /analytics/devices/{device_id} | Device-specific analytics | TECHNICIAN | Yes | Sync | service_tickets, device_health | — |
| MA-1 | GET | /analytics/manufacturer/overview | Manufacturer-wide reliability overview | OVERALL_MANAGEMENT | Yes | Sync | service_tickets, devices, ai_analysis_results, predictive_* | — |
| MA-2 | GET | /analytics/manufacturer/serial-range-analysis | Serial-range investigation ranking | OVERALL_MANAGEMENT, DESIGN | Yes | Sync | devices, service_tickets | — |
| RA-1 | GET | /analytics/quality/fix-history | Fix/repair analytics | QUALITY | Yes | Sync | service_tickets | — |
| RA-2 | GET | /analytics/design/failure-trends | Failure/component/model trend analytics | DESIGN | Yes | Sync | service_tickets, devices | — |
| SNS-1 | POST | /sensors/readings | Ingest one simulated sensor reading | ADMIN, TECHNICIAN | Yes | Sync | sensor_readings, device_health, anomalies, predictive_events, preventive_tickets | — |
| SNS-2 | GET | /sensors/{device_id}/history | Sensor reading history | TECHNICIAN, OVERALL_MANAGEMENT, DESIGN | Yes | Sync | sensor_readings | — |
| PRD-1 | GET | /predictive/{device_id}/health | Current device health | TECHNICIAN, OVERALL_MANAGEMENT, DESIGN | Yes | Sync | device_health | — |
| PRD-2 | GET | /predictive/{device_id}/anomalies | Anomaly list | OVERALL_MANAGEMENT, TECHNICIAN | Yes | Sync | anomalies | — |
| PRD-3 | GET | /predictive/{device_id}/predictions | Predictive events for device | OVERALL_MANAGEMENT, TECHNICIAN | Yes | Sync | predictive_events | — |
| PRD-4 | GET | /predictive/predictions/outcomes | Prediction-vs-actual summary | OVERALL_MANAGEMENT | Yes | Sync | predictive_events | — |
| PVT-1 | GET | /preventive-tickets | List preventive tickets | OVERALL_MANAGEMENT, TECHNICIAN | Yes | Sync | preventive_tickets | — |
| PVT-2 | POST | /preventive-tickets/{id}/acknowledge | Acknowledge preventive ticket | TECHNICIAN, OVERALL_MANAGEMENT | Yes | Sync | preventive_tickets | — |
| PVT-3 | POST | /preventive-tickets/{id}/link-service-ticket | Link to a real service ticket | TECHNICIAN | Yes | Sync | preventive_tickets | — |
| CHT-1 | POST | /chat/message | Gemini chatbot | TECHNICIAN | Yes | Sync | — | Gemini |
| PBI-1 | GET | /powerbi/dataset/service-tickets | Flattened, PII-free ticket dataset | OVERALL_MANAGEMENT, ADMIN | Yes | Sync | service_tickets, devices | — |
| PBI-2 | GET | /powerbi/dataset/ai-predictions | AI prediction dataset | OVERALL_MANAGEMENT, ADMIN | Yes | Sync | ai_analysis_results | — |
| PBI-3 | GET | /powerbi/dataset/predictive-maintenance | Predictive maintenance dataset | OVERALL_MANAGEMENT, ADMIN | Yes | Sync | device_health, anomalies, preventive_tickets, predictive_events | — |
| SYS-1 | GET | /health | Liveness check | None | No | Sync | — | — |
| AUD-1 | GET | /audit-logs | List audit log entries | ADMIN | Yes | Sync | audit_logs | — |

---

## PART 5 — API Request/Response JSON (representative examples)

**POST /auth/login**
```json
{ "username": "tech1", "password": "Password123!" }
```
→
```json
{ "success": true, "data": { "access_token": "eyJ...", "refresh_token": "eyJ...", "token_type": "bearer" }, "request_id": "..." }
```

**POST /devices/lookup**
```json
{ "identifier": "DEV-90612", "identifier_type": "device_id" }
```
→
```json
{
  "success": true,
  "data": {
    "device": { "device_id": "DEV-90612", "product_model": "AC-Model-X1", "serial_range": "SR-001", "manufacturing_date": null, "status": "ACTIVE" },
    "summary": { "total_service_tickets": 4 }
  },
  "request_id": "..."
}
```

**POST /service-tickets**
```json
{
  "ticket_id": "TCK-2044",
  "device_id": "DEV-90612",
  "date": "2026-08-12T09:30:00Z",
  "symptom_text": "AC not cooling, loud noise from compressor",
  "fix_text": "Replaced compressor",
  "technician_diagnosis_failure_mode": "Mechanical Failure",
  "technician_diagnosis_component": "Compressor",
  "technician_diagnosis_department": "Mechanical",
  "severity": "HIGH"
}
```

**POST /service-tickets/{ticket_id}/analyze** → (Azure OpenAI configured case)
```json
{
  "success": true,
  "data": {
    "model_name": "azure-openai",
    "model_version": "gpt-4o-mini-deployment",
    "prediction_timestamp": "2026-08-12T09:31:02Z",
    "predicted_failure_mode": "Refrigerant Failure",
    "predicted_component": "Refrigeration System",
    "predicted_department": "Refrigeration",
    "confidence": 0.91,
    "suggested_action": "Inspect for refrigerant leakage near the compressor manifold.",
    "low_confidence": false
  },
  "request_id": "..."
}
```

**GET /service-tickets/{ticket_id}/comparison**
```json
{ "success": true, "data": { "failure_mode_match": false, "component_match": true, "department_match": true, "overall_match": false }, "request_id": "..." }
```

**POST /sensors/readings**
```json
{
  "device_id": "DEV-90612", "timestamp": "2026-08-12T10:00:00Z",
  "temperature": 30.1, "compressor_current": 12.4, "vibration": 5.2,
  "fan_speed": 1550, "power_consumption": 2100, "refrigerant_pressure": 205, "humidity": 71
}
```
→
```json
{ "success": true, "data": { "device_id": "DEV-90612", "health_score": 43.93, "anomaly_score": 0.5607, "status": "DEGRADING" }, "request_id": "..." }
```

**Error response (any endpoint)**
```json
{
  "success": false,
  "error": { "code": "AI_SERVICE_UNAVAILABLE", "message": "AI analysis is temporarily unavailable.", "retryable": true, "details": null },
  "request_id": "..."
}
```

---

## PART 6 — RBAC Matrix

`ALLOW-R` = read, `ALLOW-RW` = read/write, `DENY` = forbidden. ADMIN implicitly
passes every `require_roles(...)` check in the codebase (see `auth/deps.py`).

| Endpoint group | Technician | Quality | Design | Management | Admin |
|---|---|---|---|---|---|
| /auth/* | ALLOW-RW | ALLOW-RW | ALLOW-RW | ALLOW-RW | ALLOW-RW |
| /users | DENY | DENY | DENY | DENY | ALLOW-RW |
| /devices/lookup, /devices/{id} | ALLOW-R | ALLOW-R | ALLOW-R | ALLOW-R | ALLOW-R |
| /devices (create) | DENY | DENY | DENY | DENY | ALLOW-RW |
| /service-tickets (create/complete) | ALLOW-RW | ALLOW-RW (complete only) | DENY | DENY | ALLOW-RW |
| /service-tickets/{id} (get) | ALLOW-R | ALLOW-R | ALLOW-R | ALLOW-R | ALLOW-R |
| /service-tickets/{id}/images | ALLOW-RW (own tickets) | ALLOW-R | DENY | DENY | ALLOW-RW |
| /images/{id}/ocr | ALLOW-RW | DENY | DENY | DENY | ALLOW-RW |
| /service-tickets/{id}/analyze | ALLOW-RW | DENY | DENY | DENY | DENY (use ml-result) |
| /service-tickets/{id}/ml-result | DENY | DENY | DENY | DENY | ALLOW-RW |
| /service-tickets/{id}/ai-results | ALLOW-R | ALLOW-R | ALLOW-R | ALLOW-R | ALLOW-R |
| /service-tickets/{id}/embed | DENY | DENY | DENY | DENY | ALLOW-RW |
| /service-tickets/{id}/comparison | ALLOW-R | ALLOW-R | ALLOW-R | ALLOW-R | ALLOW-R |
| /analytics/devices/{id} | ALLOW-R | DENY | DENY | DENY | ALLOW-R |
| /analytics/manufacturer/overview | DENY | DENY | DENY | ALLOW-R | ALLOW-R |
| /analytics/manufacturer/serial-range-analysis | DENY | DENY | ALLOW-R | ALLOW-R | ALLOW-R |
| /analytics/quality/fix-history | DENY | ALLOW-R | DENY | DENY | ALLOW-R |
| /analytics/design/failure-trends | DENY | DENY | ALLOW-R | DENY | ALLOW-R |
| /sensors/readings (ingest) | ALLOW-RW | DENY | DENY | DENY | ALLOW-RW |
| /sensors/{id}/history | ALLOW-R | DENY | ALLOW-R | ALLOW-R | ALLOW-R |
| /predictive/{id}/health | ALLOW-R | DENY | ALLOW-R | ALLOW-R | ALLOW-R |
| /predictive/{id}/anomalies, /predictions | ALLOW-R | DENY | DENY | ALLOW-R | ALLOW-R |
| /predictive/predictions/outcomes | DENY | DENY | DENY | ALLOW-R | ALLOW-R |
| /preventive-tickets (list) | ALLOW-R | DENY | DENY | ALLOW-R | ALLOW-R |
| /preventive-tickets/{id}/acknowledge | ALLOW-RW | DENY | DENY | ALLOW-RW | ALLOW-RW |
| /preventive-tickets/{id}/link-service-ticket | ALLOW-RW | DENY | DENY | DENY | ALLOW-RW |
| /chat/message | ALLOW-RW | DENY | DENY | DENY | DENY |
| /powerbi/dataset/* | DENY | DENY | DENY | ALLOW-R | ALLOW-R |
| /health | ALLOW-R | ALLOW-R | ALLOW-R | ALLOW-R | ALLOW-R |
| /audit-logs | DENY | DENY | DENY | DENY | ALLOW-R |

RBAC is enforced **server-side** via the `require_roles(...)` FastAPI
dependency on every protected route (`app/auth/deps.py`) — never via
frontend button-hiding alone.

---

## PART 7 — ML Integration Contract

**Backend → ML** (what the backend sends, once a request-driven pipeline exists):
```json
{ "record": "REC-1042", "date": "2026-08-12", "product_model": "AC-Model-X1",
  "serial_range": "SR-001", "fix_text": "Replaced compressor",
  "symptom_text": "AC not cooling, loud noise from compressor" }
```
`failure_mode`, `component`, `department` (ground truth) are **never** included.

**ML → Backend** (via `POST /service-tickets/{ticket_id}/ml-result`, `MLRawPayload` schema):
```json
{ "model_name": "kmeans-placeholder", "model_version": "0.1",
  "failure_mode": "Refrigerant Failure", "component": "Compressor",
  "department": "Refrigeration", "confidence": 0.7,
  "suggested_action": "Inspect refrigerant lines", "extra": {} }
```

This is normalized by `app/integrations/ml_adapter.py::normalize_ml_output()`
into the canonical `AIAnalysisResult` shape. **Only this one function needs to
change** when the ML team's real schema lands — no route, model, or frontend
code depends on the raw ML JSON shape.

---

## PART 8 — Azure OpenAI Integration

```
FastAPI route (ai_analysis.py)
      ↓
AzureOpenAIService.classify_service_note()   (app/services/azure_openai_service.py)
      ↓  sanitize() strips injection markers, truncates to 4000 chars
Azure OpenAI Chat Completions API
      ↓
normalized result { failure_mode, component, department, confidence, suggested_action }
      ↓
AIAnalysisResult row (raw_result_json = full API response, for audit only)
      ↓
Frontend / Analytics
```

Environment variables (see `.env.example`):
```
AZURE_OPENAI_ENDPOINT=CONFIGURATION_PENDING
AZURE_OPENAI_API_KEY=CONFIGURATION_PENDING
AZURE_OPENAI_API_VERSION=CONFIGURATION_PENDING
AZURE_OPENAI_DEPLOYMENT=CONFIGURATION_PENDING
```
No secrets are hardcoded; `AzureOpenAIService` is the only caller of Azure
OpenAI in the codebase. Until configured, `analyze_ticket` returns
`503 CONFIGURATION_PENDING` rather than throwing an unhandled error.

---

## PART 9 — Embedding Pipeline

```
sanitized service note (symptom_text)
      ↓
EmbeddingService.embed()  (app/services/embedding_service.py)
      ↓
Azure OpenAI Embeddings API
      ↓
vector (float[]) — internal only, never returned to frontend users
      ↓
Python clustering pipeline (app/ml — currently the KMeans placeholder;
      real pipeline can consume the same vectors later)
      ↓
cluster + cluster_label
      ↓
clusters table (cluster_id_source = GENERATED, distinct from dataset-provided
      cluster_id values, which are cluster_id_source = DATASET)
```

`POST /service-tickets/{id}/embed` currently confirms embedding succeeded but
does not persist the vector (no vector column/index in the MVP schema) — see
OPEN CONTRACTS if a vector store becomes required.

---

## PART 10 — Analytics Contract

**Raw operational APIs** (single-record CRUD): `/devices/*`, `/service-tickets/*`, `/images/*`.

**Analytics APIs** (aggregated, read-only): `/analytics/*`, `/powerbi/*`, `/predictive/predictions/outcomes`.

| Analytics question (Section 16) | Endpoint |
|---|---|
| 1. What is failing the most? | `/analytics/manufacturer/overview` → `failure_mode_distribution` |
| 2. Which AC model has the problem? | `/analytics/manufacturer/overview` → `model_x_ticket_count` |
| 3. Is the problem getting worse? | `/predictive/predictions/outcomes` (trend needs a date-bucketed extension — see OPEN CONTRACTS) |
| 4. Suspicious serial range? | `/analytics/manufacturer/serial-range-analysis` |
| 5. Detecting problems before failures? | `/predictive/predictions/outcomes`, `/preventive-tickets` |
| 6. Departments associated with failures? | `/analytics/manufacturer/overview` → `department_distribution` |
| 7. Components fail most often? | `/analytics/design/failure-trends` → `component_trends` |
| 8. Most common fixes? | `/analytics/quality/fix-history` → `most_common_fixes` |
| 9. Models with disproportionate failures? | `/analytics/manufacturer/overview` → `model_x_ticket_count` |
| 10. Failure modes increasing? | needs time-bucketed extension — OPEN CONTRACTS |
| 11. Serial ranges deserving investigation? | `/analytics/manufacturer/serial-range-analysis` (tickets/device ratio heuristic) |
| 12. Health of a simulated AC? | `/predictive/{device_id}/health` |
| 13. How many anomalies? | `/analytics/manufacturer/overview` → `predictive_maintenance.anomalies_detected` |
| 14. Preventive tickets generated? | `/preventive-tickets`, overview `preventive_tickets_generated` |
| 15. Predicted failures confirmed? | overview `predicted_failures_confirmed` |
| 16. False/pending alerts? | overview `false_positive_alerts` |

**Metric definitions** (denominators, per Section 15 requirement):
- `tickets_per_device` (serial-range analysis) = `count(service_tickets) / count(distinct devices)` for that serial range.
- `health_score` = `100 − (avg deviation-from-baseline ratio across 7 sensor fields) × 100`, range 0–100.
- `anomaly_score` = mean of per-field deviation ratios (0 = within baseline, 1 = maximally deviated), range 0–1.
- No metric is labeled "failure rate" anywhere in the code, since a true rate would need a population-at-risk denominator (e.g. operating hours) not currently modeled — see OPEN CONTRACTS.

---

## PART 11 — Predictive Maintenance

```
Sensor Simulator (Python, external process)
      ↓  POST /sensors/readings  (direct HTTP, no IoT Hub/Event Hub)
FastAPI → sensor_adapter.normalize_sensor_payload()
      ↓
SensorReading row
      ↓
PredictiveService.process_reading()  (heuristic health/anomaly scoring)
      ↓
DeviceHealth row  +  (if anomalous) Anomaly row
      ↓
if health_score ≤ PREDICTIVE_HEALTH_THRESHOLD:
      PredictiveEvent row (predicted_issue, confidence, risk_level)
      ↓
      PreventiveTicket row (status=PENDING)
      ↓
Technician: GET /preventive-tickets → POST .../acknowledge → POST .../link-service-ticket
      ↓
Ordinary ServiceTicket workflow (repair happens, ground truth recorded)
```

Schemas: see `SensorReadingIn`, `DeviceHealthResponse`, `PreventiveTicketResponse`
in `app/schemas/sensor.py`. Endpoints: see Part 4, SNS/PRD/PVT groups.

The MVP scoring model (`app/services/predictive_service.py`) is an explicit,
documented heuristic (deviation from fixed baseline ranges per sensor field) —
**not** a trained model — chosen so the full loop is demonstrable without
requiring labeled failure data upfront. It is designed to be swapped for a
real model without changing the API contract (adapter pattern, same as ML).

---

## PART 12 — Gemini

```
Frontend (chat widget)
      ↓  POST /chat/message  { message, device_id?, ticket_id? }
FastAPI (chatbot.py) — enforces TECHNICIAN role
      ↓
GeminiService.send_message()  (app/services/gemini_service.py)
      ↓
Gemini generateContent API
      ↓
{ reply: "..." }
      ↓
Frontend
```

The Gemini API key (`GEMINI_API_KEY`) never reaches the browser — only the
backend calls Gemini. Request/response schema: `ChatRequest` / `ChatResponse`
in `app/schemas/chatbot.py`.

---

## PART 13 — Error Handling / Taxonomy

Centralized in `app/utils/responses.py`. Every error returns:
```json
{ "success": false, "error": { "code": "...", "message": "...", "retryable": true|false, "details": null }, "request_id": "..." }
```

| Code | HTTP | Retryable | Meaning |
|---|---|---|---|
| VALIDATION_ERROR | 422 | No | Pydantic validation failed |
| AUTH_INVALID_CREDENTIALS | 401 | No | Bad username/password |
| AUTH_TOKEN_EXPIRED / AUTH_TOKEN_INVALID | 401 | Yes/No | JWT expired or malformed |
| FORBIDDEN_ROLE | 403 | No | RBAC denied |
| DEVICE_NOT_FOUND / TICKET_NOT_FOUND / USER_NOT_FOUND / RESOURCE_NOT_FOUND | 404 | No | Missing entity |
| DUPLICATE_RESOURCE | 409 | No | Unique constraint violated |
| AI_SERVICE_UNAVAILABLE | 503 | Yes | Azure OpenAI call failed |
| AI_LOW_CONFIDENCE | 200 | No | Not an error — informational flag on a successful analysis |
| CHATBOT_SERVICE_UNAVAILABLE | 503 | Yes | Gemini call failed |
| DATABASE_ERROR | 500 | Yes | Unexpected DB failure |
| FILE_UPLOAD_ERROR | 400 | No | Bad file type/size |
| INVALID_OCR_PAYLOAD | 422 | No | OCR JSON missing required fields |
| SENSOR_PAYLOAD_ERROR | 422 | No | Malformed sensor reading |
| PREDICTION_ERROR | 500 | Yes | Predictive scoring failure |
| CONFIGURATION_PENDING | 503 | Yes | Azure/Gemini env vars not yet supplied |

No stack traces, API keys, or DB connection strings are ever included in a
response body (verified by inspection of every `raise_error(...)` call site).

---

## PART 14 — Security

- **Passwords:** bcrypt via passlib (`app/core/security.py`) — never stored plaintext.
- **Sessions:** stateless JWT (`python-jose`), `access` (60 min default) and
  `refresh` (7 days default) tokens, distinguished by a `type` claim so a
  refresh token can't be used as an access token.
- **RBAC:** enforced server-side per-route via `require_roles(...)`.
- **Input validation:** Pydantic schemas on every request body.
- **File validation:** content-type allowlist + size cap in `StorageService.validate()`.
- **AI input sanitization:** `AzureOpenAIService._sanitize()` strips known
  prompt-injection markers ("ignore previous", "system:", etc.) and truncates
  input before it reaches Azure OpenAI — service notes are always treated as
  data, never as instructions.
- **Secret management:** all Azure/Gemini/DB credentials via environment
  variables only (`app/core/config.py`); nothing hardcoded, nothing returned
  to the frontend.
- **Internal AI prompts** are not exposed through any API response — only the
  normalized result fields are returned; `raw_result_json` is stored for audit
  but is not exposed on any endpoint used by non-ADMIN roles in this MVP.

---

## PART 15 — Project Structure

```
backend/
├── app/
│   ├── main.py                # FastAPI app, router registration, error handlers
│   ├── core/                  # config, database session, JWT/password security
│   ├── auth/                  # get_current_user, require_roles RBAC dependency
│   ├── models/                # SQLAlchemy ORM models (one file per domain area)
│   ├── schemas/                # Pydantic request/response schemas
│   ├── api/routes/             # one router module per API group (Part 4)
│   ├── services/               # AzureOpenAIService, EmbeddingService, GeminiService,
│   │                            # StorageService, PredictiveService — external/IO boundaries
│   ├── integrations/           # ml_adapter, ocr_adapter, sensor_adapter — evolving-schema
│   │                            # translation layer, see Part 28 of the brief
│   └── utils/                  # response envelope + centralized error taxonomy
├── seed.py                     # demo users/devices for local testing
├── requirements.txt
└── .env.example

ml/
├── kmeans_placeholder_training.py   # plan-B clustering script (see run_instructions.md)
└── requirements.txt

frontend/
├── src/
│   ├── api/client.js            # fetch wrapper, JWT header injection, envelope unwrapping
│   ├── context/AuthContext.jsx  # login/logout/current-user state
│   ├── components/Layout.jsx    # role-aware sidebar nav
│   └── pages/                   # Login, TechnicianDashboard, ManagementDashboard,
│                                  # QualityDashboard, DesignDashboard
└── package.json / vite.config.js
```

`app/integrations/` exists specifically so that when the frontend/ML/OCR/
sensor teams' JSON contracts change, the fix is localized to one small file
per integration rather than rippling through routes, models, and the frontend.

---

## PART 16 — Endpoint Implementation Priority

**CORE** (system doesn't function without these): AUTH-1..3, DEV-1, TKT-1/2/3,
AI-1, CMP-1, DA-1, MA-1, SNS-1, PRD-1, PVT-1/2/3, SYS-1.

**IMPORTANT** (required for full requirement coverage but not blocking a
first demo): USR-1/2, DEV-2/3, IMG-1/2, OCR-1, AI-2/3, RA-1/2, MA-2, SNS-2,
PRD-2/3/4, PBI-1/2/3, CHT-1.

**OPTIONAL** (nice-to-have / audit-oriented): AUD-1, EMB-1.

---

## PART 17 — Contract Dependencies

**Frontend needs:** JWT auth endpoints, device lookup, ticket CRUD, image
upload, AI analysis + comparison, role-scoped analytics, chatbot endpoint.
**Backend provides:** all of the above under `/api/v1`, documented in Part 4,
with a stable response envelope regardless of downstream schema changes.

**ML needs:** the six named input fields (`record, date, product_model,
serial_range, fix_text, symptom_text`) and a place to push results.
**Backend provides:** `POST /service-tickets/{id}/ml-result` +
`ml_adapter.py` as the single point of schema adaptation.

**Azure needs:** endpoint, API key, API version, deployment name (chat +
embeddings).
**Backend provides:** `AzureOpenAIService` / `EmbeddingService` abstractions
that consume exactly those four env vars each and fail gracefully if unset.

**Analytics needs:** flattened, PII-free operational data.
**Backend provides:** `/analytics/*` and `/powerbi/*` read-only datasets.

**Predictive model needs:** a place to receive sensor data and a place to
publish health/anomaly/prediction output.
**Backend provides:** `POST /sensors/readings` (ingest) and
`PredictiveService` (currently a heuristic; swappable).

**OCR needs:** an endpoint to submit recognized text against an image.
**Backend provides:** `POST /images/{image_id}/ocr` + `ocr_adapter.py`.

**Gemini needs:** nothing from the backend beyond the isolated
`GeminiService` call — Gemini is external and unauthenticated by us beyond
the API key.
**Backend provides:** `POST /chat/message` as the sole frontend-facing
surface, so the frontend never talks to Gemini directly.

---

## PART 18 — Final Endpoint Checklist

```
METHOD  PATH                                              ROLE                          PURPOSE
POST    /auth/login                                       Any                           Obtain tokens
POST    /auth/refresh                                     Any                           Rotate access token
GET     /auth/me                                           Any                           Current user
POST    /auth/logout                                       Any                           Logout (client discard)
POST    /users                                              ADMIN                         Create user
GET     /users                                              ADMIN                         List users
POST    /devices/lookup                                    Any                           Resolve device identifier
POST    /devices                                            ADMIN                         Register device
GET     /devices/{device_id}                                Any                           Get device
POST    /service-tickets                                    TECHNICIAN                    Create ticket
GET     /service-tickets/{ticket_id}                        Any                           Get ticket
POST    /service-tickets/{ticket_id}/complete                TECHNICIAN, QUALITY           Record ground truth
POST    /service-tickets/{ticket_id}/images                  TECHNICIAN                    Upload image
GET     /service-tickets/{ticket_id}/images                  Any                           List images
POST    /images/{image_id}/ocr                               TECHNICIAN, ADMIN             Submit OCR text
POST    /service-tickets/{ticket_id}/analyze                 TECHNICIAN                    Run Azure OpenAI analysis
POST    /service-tickets/{ticket_id}/ml-result                ADMIN                         Push ML prediction
GET     /service-tickets/{ticket_id}/ai-results               Any                           List AI predictions
POST    /service-tickets/{ticket_id}/embed                    ADMIN                         Generate embedding
GET     /service-tickets/{ticket_id}/comparison                Any                           Technician vs AI
GET     /analytics/devices/{device_id}                         TECHNICIAN                    Device analytics
GET     /analytics/manufacturer/overview                       OVERALL_MANAGEMENT            Reliability overview
GET     /analytics/manufacturer/serial-range-analysis           OVERALL_MANAGEMENT, DESIGN    Serial-range ranking
GET     /analytics/quality/fix-history                          QUALITY                       Fix/repair analytics
GET     /analytics/design/failure-trends                        DESIGN                        Failure/component trends
POST    /sensors/readings                                       ADMIN, TECHNICIAN              Ingest sensor reading
GET     /sensors/{device_id}/history                             TECHNICIAN, OVERALL_MANAGEMENT, DESIGN   Sensor history
GET     /predictive/{device_id}/health                            TECHNICIAN, OVERALL_MANAGEMENT, DESIGN  Current health
GET     /predictive/{device_id}/anomalies                          OVERALL_MANAGEMENT, TECHNICIAN  Anomaly list
GET     /predictive/{device_id}/predictions                        OVERALL_MANAGEMENT, TECHNICIAN  Prediction list
GET     /predictive/predictions/outcomes                            OVERALL_MANAGEMENT              Prediction-vs-actual
GET     /preventive-tickets                                          OVERALL_MANAGEMENT, TECHNICIAN  List preventive tickets
POST    /preventive-tickets/{id}/acknowledge                          TECHNICIAN, OVERALL_MANAGEMENT  Acknowledge
POST    /preventive-tickets/{id}/link-service-ticket                   TECHNICIAN                      Link to service ticket
POST    /chat/message                                                  TECHNICIAN                      Gemini chatbot
GET     /powerbi/dataset/service-tickets                                OVERALL_MANAGEMENT, ADMIN        PBI ticket dataset
GET     /powerbi/dataset/ai-predictions                                 OVERALL_MANAGEMENT, ADMIN        PBI AI dataset
GET     /powerbi/dataset/predictive-maintenance                          OVERALL_MANAGEMENT, ADMIN        PBI predictive dataset
GET     /health                                                          None                             Liveness check
GET     /audit-logs                                                       ADMIN                            Audit trail
```

---

## Data Flows (Section 32)

1. **Login:** Frontend → `/auth/login` → JWT issued → stored client-side → attached as Bearer header on every subsequent call.
2. **Barcode/device lookup:** Frontend decodes barcode → `/devices/lookup` → device row returned, or `DEVICE_NOT_FOUND`.
3. **Manual service report:** Technician fills form → `/service-tickets` → raw data stored, `status=OPEN`.
4. **Image + OCR:** `/service-tickets/{id}/images` (upload) → OCR team's component processes it out-of-band → `/images/{image_id}/ocr` (submit result) → text attached to image.
5. **AI analysis:** `/service-tickets/{id}/analyze` → sanitize → Azure OpenAI → normalize → `AIAnalysisResult` stored → `status=ANALYZED`.
6. **Low-confidence response:** same as (5), but `confidence < AI_MIN_CONFIDENCE` → `low_confidence: true` returned; ticket is **not** blocked, technician is expected to verify manually.
7. **Technician vs AI:** `/service-tickets/{id}/comparison` compares `technician_diagnosis_*` (if present) to the latest `AIAnalysisResult`.
8. **Device analytics:** `/analytics/devices/{device_id}` — scoped strictly to that device's own history.
9. **Manufacturer analytics:** `/analytics/manufacturer/*` — cross-device aggregates, OVERALL_MANAGEMENT (+DESIGN for serial-range).
10. **Power BI:** scheduled Web/REST pull from `/powerbi/dataset/*` → Power BI model → Reliability Overview / Failure Mode Mining / Predictive Maintenance reports.
11. **Sensor simulation:** Python simulator → `/sensors/readings` directly (no broker) → stored.
12. **Predictive maintenance:** `PredictiveService.process_reading()` computes health/anomaly on every ingested reading.
13. **Preventive ticket creation:** health_score below threshold → `PredictiveEvent` + `PreventiveTicket(status=PENDING)` created automatically, same transaction.
14. **Preventive → technician workflow:** technician calls `/preventive-tickets/{id}/acknowledge`, does the repair as a normal `ServiceTicket`, then `/preventive-tickets/{id}/link-service-ticket` closes the loop.
15. **Gemini chatbot:** Frontend → `/chat/message` → `GeminiService` → Gemini → reply relayed back, never exposing the API key.

---

## OPEN CONTRACTS

Still to be supplied by the respective teams before these are fully live
(all currently marked `CONFIGURATION_PENDING` or documented as an assumption
above — the architecture does not block on any of them):

- **Final ML JSON schema** (frozen field names/types beyond the currently-known `failure_mode/component/department/confidence/suggested_action`).
- **Final confidence threshold** — currently `AI_MIN_CONFIDENCE=0.55`, configurable via env var, not yet validated against real model performance.
- **Azure OpenAI**: endpoint, API key, API version, deployment name.
- **Azure OpenAI Embeddings**: endpoint, API key, API version, deployment name; and whether embedding vectors need to be persisted (would require a vector column/index not in the current schema).
- **OCR JSON schema** from the analytics/OCR team (current adapter accepts `{text, raw_json}` and best-effort extracts `text`/`full_text`/`ocr_text` keys).
- **Predictive model JSON schema**, if/when the MVP heuristic in `PredictiveService` is replaced by a trained model.
- **Power BI connection method** — assumed Web/REST connector against `/powerbi/dataset/*`; confirm if a different mechanism (e.g. a dedicated export job) is preferred.
- **Gemini response schema specifics** (current integration assumes the standard `generateContent` response shape).
- **Final database hosting choice** (local Postgres vs managed service) — connection string is the only thing that needs to change (`DATABASE_URL`).
- **Time-bucketed trend endpoints** (Section 16, questions 3 and 10 — "is it getting worse," "which failure modes are increasing") need an agreed time-bucketing convention (weekly/monthly) before implementation; not built in this MVP pass.
- **Token revocation/blacklisting** — current JWT logout is client-side only; if server-side revocation is required, a token-blacklist table would need to be added.
- **Production CORS origin list** — currently `allow_origins=["*"]` for development convenience; must be restricted before any non-local deployment.
