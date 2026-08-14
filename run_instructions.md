# Run Instructions — Smart AC Failure Intelligence & Predictive Maintenance

This project has three parts:

```
smart-ac-platform/
├── backend/     FastAPI application (this is what everyone integrates against)
├── ml/          Placeholder KMeans clustering script (plan B for the ML team)
└── frontend/    React (Vite) app with role-based dashboards and analytics charts
```

The analytics dashboards are backed by FastAPI and PostgreSQL. Power BI remains
an additional manufacturer reporting/export layer; the React dashboards are the
in-product visualization layer.

---

## 1. Prerequisites

- Python 3.11+ (3.12 used in testing)
- Node.js 18+ and npm
- PostgreSQL 14+ (or just use SQLite for local dev — see note below)

## 2. Backend setup

```bash
cd backend
python3 -m venv venv
source venv/bin/activate        # Windows: venv\\Scripts\\activate
pip install -r requirements.txt
```

### 2.1 Configure environment

```bash
cp .env.example .env
```

Edit `.env`:

- **DATABASE_URL** — point at your Postgres instance, e.g.
  `postgresql+psycopg2://ac_user:ac_password@localhost:5432/smart_ac_db`
- **SECRET_KEY** — set to any long random string.
- **AZURE_OPENAI_*** / **AZURE_OPENAI_EMBEDDING_*** / **GEMINI_API_KEY** — leave
  as `CONFIGURATION_PENDING` until real values are available. The app still boots.

### 2.2 Create the database (if using Postgres)

```sql
CREATE DATABASE smart_ac_db;
CREATE USER ac_user WITH PASSWORD 'ac_password';
GRANT ALL PRIVILEGES ON DATABASE smart_ac_db TO ac_user;
```

### 2.3 Run the API

```bash
uvicorn app.main:app --reload --port 8000
```

Tables are created automatically on startup (`Base.metadata.create_all`).
Interactive API docs: **http://localhost:8000/docs**

### 2.4 Seed demo users/devices and analytics data

In a second terminal (venv still active):

```bash
python seed.py
python seed_analytics_demo.py
```

`seed.py` creates the role accounts and baseline devices. The analytics seed is
separate and repeat-safe: it creates non-PII demo tickets, diagnosis/ground-truth
fields, AI prediction rows, device health history, anomalies, predictive events,
and preventive tickets. Demo analytics records are prefixed `DEMO-AN-`.

| username  | password       | role               |
|-----------|----------------|--------------------|
| tech1     | Password123!   | TECHNICIAN         |
| mgmt1     | Password123!   | OVERALL_MANAGEMENT |
| quality1  | Password123!   | QUALITY            |
| design1   | Password123!   | DESIGN             |
| admin1    | Password123!   | ADMIN              |

Demo devices from `seed.py`: `DEV-90612`, `DEV-90613`, `DEV-90700`.
Analytics devices from `seed_analytics_demo.py`: `DEMO-AN-001` through `DEMO-AN-006`.

### 2.5 Quick sanity check

```bash
curl http://localhost:8000/api/v1/health
```

Then log in and use the returned Bearer token against the analytics endpoints in
`/docs`.

## 3. Frontend setup

```bash
cd frontend
npm install
cp .env.example .env
npm run dev
```

Open **http://localhost:5173**. Sign in with the seeded role account. The
management, quality and design pages now render live charts from FastAPI rather
than static placeholder graphs.

Useful pages:

- `/management` — reliability, failure trends, severity, model/component analysis,
  serial-range risk and predictive maintenance.
- `/quality` — fixes, component repair frequency, repair trend and severity.
- `/design` — failure trend, model comparison, component/failure-mode distribution,
  model × failure-mode matrix and serial-range patterns.
- `/technician` — service console and device-specific analytics.

## 4. ML placeholder (KMeans plan-B script)

This is **not** the primary ML pipeline — it's a fallback so the platform has
something to plug in if the real pipeline isn't ready.

```bash
cd ml
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python kmeans_placeholder_training.py --input /path/to/FINALDATASET.csv --output ./artifacts --k 6
```

## 5. Simulating sensor data (predictive maintenance)

Any process can POST sensor readings directly:

```bash
curl -X POST http://localhost:8000/api/v1/sensors/readings \
  -H "Authorization: Bearer <technician_or_admin_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "device_id": "DEV-90612", "timestamp": "2026-08-12T10:00:00Z",
    "temperature": 30, "compressor_current": 12, "vibration": 5.0,
    "fan_speed": 1500, "power_consumption": 2000,
    "refrigerant_pressure": 200, "humidity": 70
  }'
```

The predictive pipeline can create DeviceHealth, Anomaly, PredictiveEvent and
PreventiveTicket records, which are then visible in the management dashboard.

## 6. Power BI

Power BI reads the same PII-free analytics layer through:

- `GET /api/v1/powerbi/dataset/service-tickets`
- `GET /api/v1/powerbi/dataset/ai-predictions`
- `GET /api/v1/powerbi/dataset/predictive-maintenance`

These endpoints require an `OVERALL_MANAGEMENT` or `ADMIN` bearer token and are
read-only. The React application does not expose the Power BI credentials to the
browser; Power BI is intended as the separate manufacturer reporting layer.

## 7. Smoke testing

Use `/docs` (Swagger UI) to verify, in order:
`POST /auth/login` → **Authorize** with the access token → analytics endpoints.

For the UI, log in as `mgmt1` and open `/management`. After running
`seed_analytics_demo.py`, the failure trend should show a multi-day series and the
model/component/failure-mode/severity charts should contain multiple categories.

## 8. Known limitations

See `architecture.md` → **OPEN CONTRACTS** for external Azure/Gemini/ML/OCR
credentials and other integration contracts. Production CORS and deployment
configuration still need environment-specific values.
