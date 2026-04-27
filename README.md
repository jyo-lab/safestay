# CrisisSync — Backend API

**Accelerated Emergency Response & Crisis Coordination in Hospitality**

FastAPI backend powering the CrisisSync platform. Real-time WebSocket updates, AI-powered sensor classification, multi-channel alert dispatch, and full guest/incident management.

---

## Tech stack

| Layer | Technology |
|---|---|
| Framework | **FastAPI** (Python 3.12) |
| ORM | **SQLAlchemy 2.0** async |
| Database | **SQLite** (dev) / **PostgreSQL** (prod) |
| Auth | **JWT** (python-jose) + bcrypt |
| Real-time | **WebSockets** (native FastAPI) |
| SMS alerts | **Twilio** (optional) |
| AI classifier | Rule-based + heuristic engine |
| Deployment | **Vercel** (serverless Python) |

---

## Project structure

```
crisissync/
├── api/
│   └── index.py                 # Vercel serverless entry point
├── app/
│   ├── main.py                  # FastAPI app & middleware
│   ├── core/
│   │   ├── config.py            # Settings (pydantic-settings)
│   │   ├── database.py          # Async SQLAlchemy engine
│   │   └── security.py          # JWT + password hashing
│   ├── models/
│   │   └── user.py              # All ORM models (Hotel, User, Guest, Incident, Sensor…)
│   ├── schemas/
│   │   └── schemas.py           # Pydantic v2 request/response schemas
│   ├── routers/
│   │   ├── auth.py              # POST /api/auth/register|login|logout|me
│   │   ├── incidents.py         # CRUD + /api/incidents/sos
│   │   ├── guests.py            # CRUD /api/guests
│   │   ├── sensors.py           # /api/sensors + /reading (AI classify)
│   │   ├── operations.py        # alerts, tasks, dashboard, analytics, evacuation
│   │   └── ws.py                # WebSocket /ws/{hotel_id}
│   └── services/
│       ├── notifications.py     # Twilio SMS + stub channels
│       ├── ai_classifier.py     # Sensor + incident AI classification
│       └── websocket_manager.py # WS connection pool
├── public/
│   └── index.html               # Frontend UI (open in browser)
├── scripts/
│   └── seed.py                  # Demo data seeder
├── requirements.txt
├── vercel.json                  # Vercel deployment config
├── .env.example
└── README.md
```

---

## Local development in VS Code

### Step 1 — Prerequisites

Make sure you have installed:
- [Python 3.12](https://www.python.org/downloads/)
- [VS Code](https://code.visualstudio.com/)
- VS Code extension: **Python** (ms-python.python)

### Step 2 — Open the project

```bash
# Open the crisissync folder in VS Code
code crisissync
```

Or in VS Code: **File → Open Folder** → select the `crisissync` folder.

### Step 3 — Create a virtual environment

Open the **VS Code terminal** (`Ctrl+`` ` `` or Terminal → New Terminal) and run:

```bash
# Create virtual environment
python -m venv venv

# Activate it
# macOS / Linux:
source venv/bin/activate

# Windows (PowerShell):
venv\Scripts\Activate.ps1

# Windows (Command Prompt):
venv\Scripts\activate.bat
```

VS Code will prompt you to select the new `venv` as the Python interpreter — click **Yes**.

### Step 4 — Install dependencies

```bash
pip install -r requirements.txt
```

### Step 5 — Configure environment

```bash
# Copy the example env file
cp .env.example .env
```

Open `.env` in VS Code and set `SECRET_KEY` to any random string (minimum 32 characters).  
`DATABASE_URL` defaults to SQLite — no extra setup needed for local dev.

Generate a secure key with:
```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

### Step 6 — Seed the database with demo data

```bash
python scripts/seed.py
```

Expected output:
```
✅ Demo data seeded successfully!

Demo credentials:
  Manager  → sarah@grandhotel.com    / manager123
  Security → rahul@grandhotel.com    / security123
  Staff    → ananya@grandhotel.com   / staff123
```

### Step 7 — Run the server

```bash
uvicorn app.main:app --reload --port 8000
```

The API is now live at:

| URL | Description |
|---|---|
| http://localhost:8000 | Root / health check |
| http://localhost:8000/docs | **Swagger interactive docs** |
| http://localhost:8000/redoc | ReDoc documentation |
| http://localhost:8000/health | Health endpoint |
| ws://localhost:8000/ws/{hotel_id} | WebSocket stream |

### Step 8 — Open the frontend

Open `public/index.html` in your browser (double-click or right-click → Open with Live Server).  
Point the API base URL in the HTML to `http://localhost:8000`.

---

## Deploy to Vercel

### Step 1 — Install Vercel CLI

```bash
npm install -g vercel
```

### Step 2 — Login

```bash
vercel login
```

### Step 3 — Link the project

From inside the `crisissync` folder:

```bash
vercel
```

Follow the prompts:
- Set up and deploy? **Y**
- Which scope? Select your account
- Link to existing project? **N** (first time)
- Project name: `crisissync` (or your choice)
- In which directory is your code located? `.` (current directory)

### Step 4 — Set environment variables in Vercel

```bash
# Required
vercel env add SECRET_KEY
# Paste a 32+ char random string

vercel env add DATABASE_URL
# For production use Vercel Postgres / Neon / Supabase URL:
# postgresql+asyncpg://user:password@host/dbname

vercel env add ENVIRONMENT
# production

# Optional — Twilio SMS
vercel env add TWILIO_ACCOUNT_SID
vercel env add TWILIO_AUTH_TOKEN
vercel env add TWILIO_PHONE_NUMBER

# CORS — set to your frontend URL
vercel env add CORS_ORIGINS
# e.g. https://crisissync.vercel.app
```

Or set them in the Vercel dashboard: **Project → Settings → Environment Variables**.

### Step 5 — Deploy to production

```bash
vercel --prod
```

After deploy, your API is live at `https://crisissync-<hash>.vercel.app`.

### Step 6 — Seed demo data (production, one-time)

```bash
# Set local DATABASE_URL to production DB, then run:
DATABASE_URL="postgresql+asyncpg://..." python scripts/seed.py
```

---

## Database options

| Environment | Recommended | Set DATABASE_URL to |
|---|---|---|
| Local dev | SQLite (built-in) | `sqlite+aiosqlite:///./crisissync.db` |
| Vercel prod | **Vercel Postgres** | auto-injected by Vercel |
| Vercel prod | **Neon** (free tier) | `postgresql+asyncpg://...` |
| Vercel prod | **Supabase** | `postgresql+asyncpg://...` |

> **Vercel Postgres** (easiest): In the Vercel dashboard, go to **Storage → Create Database → Postgres**.  
> Vercel automatically injects `DATABASE_URL` into your project.

---

## API overview

### Authentication
```
POST /api/auth/register     Create account + hotel
POST /api/auth/login        Get JWT token
GET  /api/auth/me           Current user profile
POST /api/auth/logout       Logout (client discards token)
```

### Incidents & SOS
```
GET    /api/incidents/          List all incidents
POST   /api/incidents/          Create incident
POST   /api/incidents/sos       Guest SOS trigger (full response chain)
GET    /api/incidents/{id}      Get incident
PATCH  /api/incidents/{id}      Update / resolve incident
```

### Guest tracking
```
GET    /api/guests/             List guests (filter by status, floor)
POST   /api/guests/             Check in guest
GET    /api/guests/{id}         Get guest
PATCH  /api/guests/{id}         Update status / location
```

### Fire & sensor system
```
GET    /api/sensors/                List all sensors
POST   /api/sensors/                Create sensor
POST   /api/sensors/reading         Ingest reading → AI classify → alarm if needed
GET    /api/sensors/{id}/readings   Reading history
```

### Notifications
```
POST   /api/alerts/mass         Send mass notification (SMS/push/PA/in-room)
GET    /api/alerts/             Alert history
```

### Staff tasks
```
GET    /api/tasks/              Task list (staff sees own; manager sees all)
POST   /api/tasks/              Create and assign task
PATCH  /api/tasks/{id}          Update task status
```

### Dashboard & analytics
```
GET    /api/dashboard/stats     Live property metrics
GET    /api/analytics/summary   All-time analytics + AI recommendations
```

### Evacuation
```
GET    /api/evacuation/routes   Evacuation routes
POST   /api/evacuation/activate Activate full / zone evacuation
```

### WebSocket
```
WS     /ws/{hotel_id}           Real-time event stream
```

Events received:
- `incident_created` — new incident opened
- `sos_triggered` — guest SOS with full detail
- `sensor_alarm` — sensor classified as fire/CO/anomaly
- `mass_alert` — notification dispatched
- `evacuation_activated` — evacuation started
- `task_assigned` / `task_updated` — staff task events
- `incident_updated` — status change

---

## Demo credentials (after running seed.py)

| Role | Email | Password |
|---|---|---|
| Hotel Manager | sarah@grandhotel.com | manager123 |
| Security Chief | rahul@grandhotel.com | security123 |
| Front Desk | ananya@grandhotel.com | staff123 |

---

## Environment variables reference

| Variable | Required | Description |
|---|---|---|
| `SECRET_KEY` | ✅ | JWT signing key (min 32 chars) |
| `DATABASE_URL` | ✅ | SQLite or PostgreSQL async URL |
| `ENVIRONMENT` | — | development / production |
| `DEBUG` | — | true / false |
| `CORS_ORIGINS` | — | Comma-separated allowed origins or `*` |
| `TWILIO_ACCOUNT_SID` | — | SMS via Twilio |
| `TWILIO_AUTH_TOKEN` | — | SMS via Twilio |
| `TWILIO_PHONE_NUMBER` | — | SMS sender number |

---

## Troubleshooting

**`ModuleNotFoundError: No module named 'app'`**  
Make sure your virtual environment is activated and you're running commands from the `crisissync` root folder (where `app/` lives).

**`sqlite3.OperationalError: no such table`**  
Run `python scripts/seed.py` to create tables and load demo data.

**Vercel deploy fails with timeout**  
Vercel serverless functions have a 10-second timeout on the free plan. For production workloads, upgrade to Vercel Pro or use a persistent server (Railway / Render / Fly.io).

**WebSocket on Vercel**  
Vercel's serverless functions do not support persistent WebSocket connections. For production real-time features, use a separate WebSocket server or switch to Vercel's Edge Runtime with Pusher / Ably / Supabase Realtime.
