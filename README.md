# Intelligent Workflow Automation System

A production-grade, async workflow automation engine built with **FastAPI** and **Python**.
It ingests lead events via a webhook, evaluates interest level using a rules engine, and
dynamically dispatches asynchronous actions to CRM, SMS, Email, and Newsletter services.

---

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────┐
│                        TRIGGER LAYER                           │
│   Webhook: POST /webhook/lead  ←  CRM / Polling / External     │
└─────────────────────────┬──────────────────────────────────────┘
                          │ LeadEvent (JSON)
                          ▼
┌────────────────────────────────────────────────────────────────┐
│                   ORCHESTRATION ENGINE                         │
│   workflow_engine.py — Rules-based router                      │
│                                                                │
│   interest == "High"   → [SMS Adapter] + [CRM Adapter]         │
│   interest == "Medium" → [Email Adapter] + [CRM Adapter]       │
│   interest == "Low"    → [Newsletter Adapter]                  │
└──────────┬──────────────────────┬──────────────────────────────┘
           │ asyncio.gather()     │ asyncio.gather()
           ▼                      ▼
┌──────────────────┐   ┌─────────────────────────────────────┐
│   SERVICE LAYER  │   │            DATA STORE               │
│  CRM Adapter     │   │  SQLite — workflow execution log     │
│  SMS Adapter     │   │  GET /history → audit trail          │
│  Email Adapter   │   └─────────────────────────────────────┘
│  Newsletter      │
└──────────────────┘
```

---

## Repository Structure

```
intelligent-workflow-automation/
│
├── main.py                          # Entry point — starts FastAPI on :8000
├── requirements.txt
├── workflow_history.db              # Auto-created on first run (SQLite)
│
├── src/
│   ├── api.py                       # FastAPI routes
│   ├── workflow_engine.py           # Core rules engine & orchestration
│   ├── models.py                    # Pydantic schemas
│   ├── database.py                  # Async SQLite persistence
│   └── adapters/
│       ├── crm.py                   # CRM service adapter
│       ├── messaging.py             # SMS + Email adapter
│       └── newsletter.py            # Newsletter queue adapter
│
├── mock_services/
│   ├── mock_crm.py                  # Mock CRM  (port 8001)
│   ├── mock_messaging.py            # Mock SMS/Email  (port 8002)
│   └── mock_newsletter.py           # Mock Newsletter  (port 8003)
│
└── docs/
    └── workflow_diagram.md          # Workflow & architecture description
```

---

## Quickstart

### 1. Install dependencies

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Start mock services (optional — the engine auto-simulates if offline)

Open three separate terminals:

```bash
python -m mock_services.mock_crm
python -m mock_services.mock_messaging
python -m mock_services.mock_newsletter
```

### 3. Start the main workflow API

```bash
python main.py
```

The API is now live at `http://localhost:8000`.  
Interactive docs: `http://localhost:8000/docs`

---

## API Reference

### `POST /webhook/lead`

Trigger a workflow for a new or updated lead.

**Request body:**

```json
{
  "name": "Jane Doe",
  "phone": "+919876543210",
  "email": "jane@example.com",
  "status": "New",
  "interest": "High",
  "source": "CRM-Webhook"
}
```

**`interest` values:** `"High"` | `"Medium"` | `"Low"`  
**`status` values:** `"New"` | `"Status Change"` | `"Closed"`

**Example — High Interest (cURL):**

```bash
curl -X POST http://localhost:8000/webhook/lead \
  -H "Content-Type: application/json" \
  -d '{"name":"Jane","phone":"+91999","email":"j@x.com","status":"New","interest":"High"}'
```

**Example response:**

```json
{
  "lead_name": "Jane",
  "interest": "High",
  "status": "New",
  "actions_taken": ["SMS sent", "CRM updated (Urgent / In Progress)"],
  "sms":  {"success": true, "simulated": true, "message": "SMS sent (simulated)..."},
  "crm":  {"success": true, "simulated": true, "message": "CRM updated (simulated)..."},
  "execution_id": 1
}
```

---

### `GET /history?limit=20`

Returns the last N workflow execution records from the SQLite audit log.

---

### `GET /health`

Returns `{"status": "ok"}`. Use for container health checks.

---

## Workflow Decision Table

| Interest Level | Actions Triggered                              | CRM Status   | Priority |
|---------------|------------------------------------------------|--------------|----------|
| **High**      | SMS via Twilio mock + CRM upsert               | In Progress  | Urgent   |
| **Medium**    | Email drip via SendGrid mock + CRM upsert      | Nurturing    | Medium   |
| **Low**       | Add to monthly newsletter queue                | —            | —        |

---

## Design Decisions

- **Async first** — all I/O (HTTP calls, DB writes) is `async`/`await` with `asyncio.gather` for concurrency.
- **Graceful degradation** — if a mock service is offline, adapters simulate success so the engine never crashes during demos.
- **Rules engine pattern** — adding a new interest level or action requires only a new branch in `workflow_engine.py`.
- **Audit trail** — every execution is persisted to SQLite with full result payloads.
- **Pydantic v2** — strict input validation with enum-guarded fields prevents bad data from entering the pipeline.
