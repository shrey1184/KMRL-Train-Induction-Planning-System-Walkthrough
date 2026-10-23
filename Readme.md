# KMRL Train Induction Planning System: Backend & Data Ingestion

Backend and data-ingestion layer for a daily **train induction planning system** built for Kochi Metro Rail Limited (KMRL). Each night, planners must decide which trainsets go into revenue service, which stay on standby, and which go to the inspection bay line (IBL). That decision depends on fitness certificates, maintenance job cards, branding contracts, mileage, cleaning slots, and depot layout, and today much of it arrives through spreadsheets and WhatsApp messages.

This system ingests those sources, normalises them into PostgreSQL, scores every trainset, and serves ranked decisions through REST APIs with live WebSocket updates.

> **SIH 2025 National Finalist.** Built by a team for Smart India Hackathon 2025.
> This repository is a consolidated overview of the **backend** and **data ingestion** work. The original, full-history repositories are linked below.

## Original repositories

| Part | Repository |
|------|-----------|
| Backend base (data ingestion from CSVs, train state, rule engine, APIs) | [`shrey1184/kochi-metro-base`](https://github.com/shrey1184/kochi-metro-base) *(private, source available to walk through on request)* |
| Data ingestion (WhatsApp → JSON via Periskope + Groq) | [`shrey1184/metrokochi-dataingestion`](https://github.com/shrey1184/metrokochi-dataingestion) |

> ML models are developed separately and are **not part of this repository**.

## My contribution

My role on the team was to **design the backend architecture, create the synthetic datasets, and build the agent-based ranking system**. I also added the map visualization for ranked trains and restructured the frontend code.

- **Backend architecture:** designed how the system fits together: ingestion layer, PostgreSQL schema, rule/scoring services, REST + WebSocket APIs, and the LLM layer.
- **Synthetic datasets:** created the datasets the system runs on (train master, daily operations, fitness certificates, job cards, branding SLAs, cleaning schedules, failures, stabling layout), since real KMRL data was not available.
- **Agent-based ranked system:** built the multi-agent system that scores every trainset across six constraints and produces a ranked induction list (1st, 2nd, 3rd, ...) with reasoning. This lives in the team's main repository, which is private, so it is only summarised here.
- **Map-based visualization:** added a map view showing multiple trains with their rank, so planners can see at a glance which train is ranked 1st, 2nd, and so on.
- **Frontend structure:** split large frontend components into multiple files for cleaner organisation and easier maintenance.
- **Data ingestion and multilingual support:** built the WhatsApp ingestion pipeline (Periskope webhooks + Groq/Llama 3.1 turning free text into structured JSON) and the multilingual support (English, Hindi, Malayalam), plus the feedback loop that records operator decisions and outcomes so rankings can improve over time.

### Repositories

- **`kochi-metro-base`** (backend base): loads the datasets, builds a per-train state for any date, applies the rule engine to produce IBL / SERVICE / STANDBY decisions with a stated reason and risk score, and serves them through a REST API with scheduled daily ingestion.
- **`metrokochi-dataingestion`** (WhatsApp pipeline): receives messages via Periskope webhooks and structures them with Groq.
- **Main repository** (private, shared team repo): the agent-based ranking system, map visualization, and frontend.

Teammates worked on: `<fill in, e.g. ML models, parts of the frontend>`

---

## Table of contents

1. [System overview](#system-overview)
2. [Tech stack](#tech-stack)
3. [Backend](#backend)
4. [Data ingestion (WhatsApp to JSON)](#data-ingestion-whatsapp-to-json)
5. [Decision rules and risk score](#decision-rules-and-risk-score)
6. [Emergency scenario handling](#emergency-scenario-handling)
7. [API reference](#api-reference)
8. [Setup](#setup)
9. [Testing](#testing)
10. [Status and roadmap](#status-and-roadmap)

---

## System overview

```
 Maximo job cards ─┐
 Fitness certs ────┤
 Mileage / ops ────┼──► Ingestion layer ──► PostgreSQL ──► Scoring / rule engine ──► REST API ──► Dashboard
 Branding SLAs ────┤        (validate,        (Prisma)        (rank 25 trainsets,        │
 Cleaning slots ───┤         normalise)                        6 constraints)            └──► WebSocket alerts
 WhatsApp messages ┘
      │
      └─► Periskope webhook ─► Groq LLM ─► structured JSON (intent, entities, priority)
```

**Six constraints considered per trainset:** fitness certificates, job-card status, branding priority, mileage balancing, cleaning slots, and stabling geometry. Each trainset receives a 0-100 feasibility score, and the fleet is ranked.

## Tech stack

| Layer | Technology |
|-------|-----------|
| Backend runtime | Node.js, Express, TypeScript |
| Database / ORM | PostgreSQL, Prisma |
| Validation | Zod (Joi in early drafts) |
| Real-time | WebSocket (Socket.io) |
| LLM | Groq API (Llama 3.1 70B), used for message structuring and explanations |
| Ingestion service | Python, FastAPI, Pydantic |
| WhatsApp capture | Periskope webhooks (exposed locally with ngrok during development) |
| Prototype rule engine | Python, FastAPI, SQLAlchemy, Pandas, APScheduler |

---

## Backend

### Responsibilities

- **Data ingestion:** consolidate job cards (Maximo), fitness certificates, mileage, branding contracts, cleaning slots, and stabling data
- **Scoring and decisions:** compute a 0-100 feasibility rank per trainset across six constraints
- **APIs:** ingestion, decision, what-if simulation, history/feedback, and dashboard endpoints
- **Real-time:** WebSocket events for decision changes and conflict alerts
- **Audit trail:** every decision, override, and outcome is stored, so history is never rewritten
- **Security:** JWT auth with role-based access (`supervisor`, `admin`, `system`)

### Architecture

```
┌──────────────────────────────────────────────┐
│          API Gateway (Express + middleware)  │
└───────────────┬──────────────────────────────┘
      ┌─────────┼───────────────┐
┌─────▼────┐ ┌──▼──────────┐ ┌──▼─────────┐
│ Ingestion│ │ Decision /  │ │ Dashboard  │
│ service  │ │ agent APIs  │ │ API        │
└─────┬────┘ └──┬──────────┘ └──┬─────────┘
      └─────────┼───────────────┘
        ┌───────▼────────┐
        │ Groq LLM layer │  (explanations, NL query parsing)
        └───────┬────────┘
        ┌───────▼────────┐
        │  PostgreSQL    │  (Prisma ORM, pooled connections)
        └────────────────┘
```

### Database schema (core tables)

| Table | Purpose |
|-------|---------|
| `trainsets` | Fleet master: status, depot, stabling position, cumulative mileage |
| `fitness_certificates` | Per-department certificates (rolling stock, signalling, telecom) with validity status |
| `job_cards` | Maximo work orders: type, priority, status, dates |
| `branding_contracts` | Advertiser SLAs: required vs. current exposure hours, penalty, priority |
| `mileage_tracking` | Daily and cumulative mileage, per-component wear |
| `cleaning_slots` | Scheduled/completed cleaning, bay, manpower |
| `depots`, `stabling_geometry` | Depot bays, positions, shunting distance and time |
| `induction_decisions` | Ranked decisions with score, reasoning, conflicts, override info |
| `decision_history` | Outcomes: status, punctuality impact, feedback score |
| `conflict_alerts` | Detected conflicts with severity and suggested resolution |
| `learning_feedback` | Feature vectors and predicted vs. actual outcomes (export for model training) |

Emergency-handling models (added via Prisma): `EmergencyLog`, `EmergencyPlan`, `CrisisMode`, `RouteAssignment`.

Indexes are placed on all foreign keys and on frequently filtered columns (status, priority, dates, expiry).

### Data quality and validation

- Request payloads are validated with schemas (train number format, enum values for priority/status, `expiryDate > issuedDate`).
- A readiness check flags expired certificates (critical) and open critical job cards (high) before a trainset can be ranked for service.
- Detected problems are written to `conflict_alerts` with a suggested resolution instead of being silently dropped.

---

## Data ingestion (WhatsApp to JSON)

Operations staff report status over WhatsApp ("T005 completed deep cleaning at Bay 3, ready for service"). This module turns those free-text messages into structured records the scoring engine can use.

### Flow

1. A WhatsApp message is sent or received on the connected phone.
2. **Periskope** captures it and POSTs it to the server's webhook.
3. **Groq (Llama 3.1)** extracts intent, entities, category, priority, and summary.
4. The result is saved as JSON, one file per message, and made available through the API.

### Example output

```json
{
  "id": "bd8fbf48-6ce5-4663-8da6-2c21af7cee5f",
  "original_message": "Train #12345 delayed by 30 mins at Borivali",
  "direction": "outgoing",
  "timestamp": "2025-12-08T10:30:00Z",
  "extracted_data": {
    "intent": "report_delay",
    "entities": [
      { "type": "train_number", "value": "12345", "confidence": 0.95 },
      { "type": "duration", "value": "30 mins", "confidence": 0.92 },
      { "type": "location", "value": "Borivali", "confidence": 0.98 }
    ],
    "category": "operations",
    "priority": "high",
    "action_required": true,
    "summary": "Train #12345 experiencing 30 minute delay at Borivali station"
  },
  "processing_time_ms": 823.45
}
```

### Ingestion service endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/messages/ingest` | POST | Manually ingest a message |
| `/api/messages` | GET | List stored messages |
| `/api/messages/{id}` | GET / DELETE | Fetch or delete one message |
| `/api/webhooks/register` | POST | Register the Periskope webhook |
| `/api/webhooks/list` | GET | List registered webhooks |
| `/api/webhook/periskope` | POST | Webhook receiver (called by Periskope) |
| `/docs` | GET | Swagger UI |

### Layout

```
dataingestion/
├── main.py                  # FastAPI app
├── config/settings.py       # Environment-based configuration
├── models/schemas.py        # Pydantic models
├── services/
│   ├── groq_service.py      # LLM extraction
│   ├── storage_service.py   # JSON file storage
│   └── whatsapp_service.py  # Periskope integration
└── data/messages/           # Stored JSON (git-ignored)
```

### Running it locally

```bash
cd dataingestion
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env            # add WHATSAPP_API_KEY and GROQ_API_KEY
python main.py                  # serves on :8001

# in another terminal
ngrok http 8001

# register the webhook with your ngrok URL
curl -X POST "http://localhost:8001/api/webhooks/register?webhook_url=https://<your-ngrok-url>/api/webhook/periskope"
```

**Troubleshooting:** if no messages arrive, confirm ngrok is running, the webhook URL is correct, and the webhook shows as active in the Periskope console. If messages arrive but are not processed, check `GROQ_API_KEY` and the server logs.

> Keep API keys in `.env` and never commit them. Stored messages may contain personal data, so `data/messages/` is git-ignored.

---

## Agent-based ranking (summary)

The ranked induction list comes from a multi-agent system in the team's private main repository. Six specialised agents each analyse one constraint and score every trainset from 0 to 100:

| Agent | Looks at |
|-------|----------|
| Fitness | Certificate validity across rolling stock, signalling, telecom |
| Job cards | Open Maximo work orders by priority |
| Branding | Advertiser exposure hours vs. contract, breach risk |
| Mileage | Deviation from fleet average, to balance wear |
| Cleaning | Cleaning slots, bay and manpower availability |
| Stabling | Shunting distance and position in the depot |

A conflict-resolution step then settles disagreements (hard constraints such as an expired certificate always win; soft conflicts are resolved by comparing costs), the scores are combined with weights, and an LLM writes a plain-language reason for each ranking. The result is shown on the dashboard, including a **map view that labels each train with its rank** (1st, 2nd, ...).

---

## Decision rules and risk score

The rule engine applies rules in strict priority order, so safety constraints always beat business preferences:

| Rank | Rule | Condition | Output |
|------|------|-----------|--------|
| 1 | Technical constraints | Invalid/expired certificate, open blocking job card, major failure the previous day | **IBL** |
| 2 | Cleaning | Deep cleaning scheduled | **STANDBY** |
| 3 | Branding | Active branding contract | **SERVICE** |
| 4 | Mileage balancing | Below fleet median km → SERVICE; at or above median → STANDBY | **SERVICE / STANDBY** |

Every decision carries a `primary_reason`, so a supervisor can see why a trainset was placed where it was.

**Risk score:**

| Factor | Points |
|--------|-------:|
| Invalid certificate | +50 |
| Open blocking job card | +30 |
| Major failure on previous day | +40 |
| Cumulative mileage > 50,000 km / > 30,000 km | +20 / +10 |
| Each shunting move | +5 |

Bands: 0-30 low, 31-60 medium, 61-80 high, 81+ critical.

**Weighted composite scoring** (used by the multi-constraint ranking):

| Constraint | Weight |
|------------|-------:|
| Fitness certificates | 25% |
| Job cards | 20% |
| Branding | 15% |
| Mileage | 15% |
| Stabling | 15% |
| Cleaning | 10% |

Any violated hard constraint forces the score to 0.

**Conflict resolution:** hard constraints (safety, certificates) always win. Soft conflicts, such as maintenance for an overused train against a branding SLA at risk, are resolved by comparing the cost of each option.

---

## Emergency scenario handling

Handles a breakdown during service and cascading failures:

- An IoT-style alert (`POST /api/v1/emergency/breakdown`) logs the event, marks the train `EMERGENCY_WITHDRAWN`, and creates a blocking job card.
- A fast replacement search runs with **relaxed criteria**: certificates valid ≥ 1 day (normally ≥ 3) and up to 3 critical job cards tolerated. Readiness time accounts for crew, shunting, and safety checks.
- **Crisis mode** activates when 3 or more trains fail within 30 minutes: all standby trains are deployed, trains are reassigned from low-demand to critical routes, and service frequency is reduced on non-critical routes.
- Notifications go out over WebSocket (live). SMS, WhatsApp, and email channels are stubbed behind an interface, ready for a gateway.

---

## API reference

### Ingestion

| Endpoint | Purpose |
|----------|---------|
| `POST /api/v1/ingestion/maximo` | Job cards from Maximo |
| `POST /api/v1/ingestion/fitness-certificate` | Create/update a certificate |
| `POST /api/v1/ingestion/mileage` | Daily mileage; returns cumulative total |
| `POST /api/v1/ingestion/whatsapp-log` | Raw WhatsApp text → extracted data |

### Decisions and dashboard

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/dashboard/induction-list?date=` | Ranked induction list with reasoning |
| `POST /api/v1/dashboard/what-if` | Simulate removing a train, adding maintenance, or changing priorities |
| `GET /api/v1/dashboard/conflicts` | Active conflicts grouped by severity |
| `GET /api/v1/dashboard/digital-twin` | Depot geometry for visualisation |
| `POST /api/v1/dashboard/natural-language` | English / Hindi / Malayalam queries |
| `POST /api/v1/agents/decision-submit` | Submit ranked recommendations |
| `GET /api/v1/agents/train-status/:trainId` | Full status of one trainset |

### History and feedback

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/history/decisions` | Past decisions with outcomes |
| `POST /api/v1/history/feedback` | Record the outcome of a decision |

### Emergency

`POST /emergency/breakdown` · `GET /emergency/plan/:id` · `POST /emergency/plan/approve` · `GET /emergency/active` · `GET /emergency/crisis` · `POST /emergency/crisis/reoptimize` · `POST /emergency/route/reassign` · `POST /emergency/frequency/reduce` · `POST /emergency/:id/resolve` (all under `/api/v1`)

### Rule-engine prototype (FastAPI)

| Endpoint | Purpose |
|----------|---------|
| `GET /api/induction/{date}` | Decisions for all trains on a date |
| `GET /api/induction/{date}/status/{status}` | Filter by IBL / SERVICE / STANDBY |
| `GET /api/induction/{date}/high-risk?threshold=70` | Trains above a risk threshold |
| `GET /api/train-states/{date}` | Raw train state before rules |
| `POST /api/ingest`, `GET /api/ingest/status` | Trigger / check CSV ingestion |

```bash
curl http://localhost:8000/api/induction/2025-08-08
```
```json
{
  "date": "2025-08-08",
  "total_trains": 40,
  "summary": { "IBL": 5, "SERVICE": 25, "STANDBY": 10 },
  "decisions": [
    { "train_id": "TM001", "label": "IBL", "primary_reason": "TECHNICAL_CONSTRAINT", "risk_score": 80 }
  ]
}
```

---

## Setup

### Backend (Node.js)

```bash
cd backend
npm install
cp .env.example .env
```

```env
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://user:password@localhost:5432/kmrl_db
GROQ_API_KEY=your_key_here
REDIS_URL=redis://localhost:6379
JWT_SECRET=change_me
```

```bash
npx prisma migrate dev
npx prisma generate
npm run dev
```

### Rule-engine prototype (Python)

```bash
python -m venv .venv && source .venv/bin/activate    # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python init_db.py                                    # create tables, load CSVs from ../dataset
uvicorn app.main:app --reload --port 8000            # docs at /docs
```

The prototype ingests CSVs automatically each day at 8:30 PM (APScheduler) and can be triggered manually via `POST /api/ingest`.

---

## Testing

```bash
python test_train_states.py          # writes train_states_output.json
python test_induction_decisions.py   # writes induction_decisions_output.json
python test_ingestion_complete.py    # exercises every ingestor
```

Planned for the Node backend: unit tests on the scoring rules, integration tests for the ingestion-to-decision flow, and load tests at 100 concurrent requests.

---

## Status and roadmap

| Area | Status |
|------|--------|
| Ingestion APIs and schema | Implemented |
| Rule engine and risk scoring | Implemented |
| WhatsApp → JSON pipeline (Periskope + Groq) | Implemented |
| Emergency handling and crisis mode | Implemented (backend) |
| WebSocket alerts | Implemented |
| SMS / WhatsApp / email notifications | Stubbed, awaiting gateway integration |
| Agent-based ranking and map visualization | Implemented in the private main repo; summarised above |
| ML models | Out of scope for this repo |
| Automated test coverage for scoring | Planned |

## License

Built for Smart India Hackathon 2025. Add the license you want to use here.
