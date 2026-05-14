# Intelligent Alert Escalation & Resolution System

A comprehensive, full-stack alert management platform designed for enterprise operations. SyncOps ingests driver and compliance alerts, applies a configurable rule engine to escalate or auto-close them, and surfaces everything on a real-time analytics dashboard.

<!-- ---

## 🚀 Live Deployment

| | |
|---|---|
| **Dashboard (Vercel)** | https://syncops-dashboard.vercel.app/ |
| **Backend API (Render)** | https://syncops-api.onrender.com |

> **Note:** The backend is hosted on Render's free tier. The first request after inactivity may take up to **45 seconds** to wake the server. Subsequent requests are fast.

--- -->

## 🔑 Test Credentials

Registration is intentionally disabled — this is an internal B2B tool where accounts are provisioned by an admin. Use the following credentials to log in:

| | |
|---|---|
| **Email** | `test@example.com`/`testadmin@gmail.com`|
| **Password** | `password123`/`admin123`|

---

## ✨ Core Features

### Centralized Alert Ingestion API
A single `POST /api/alerts` endpoint accepts alerts from any upstream source (telematics, feedback systems, compliance services). Each alert carries a `sourceType`, `severity`, `timestamp`, and a flexible `metadata` object, making the schema open to any integration without schema migrations.

### OOP Rule Engine with JSON DSL
Alert processing logic lives in `services/RuleEngine.js` as a strict ES6 class hierarchy — a `BaseRule` parent with `OverspeedRule`, `FeedbackRule`, and `ComplianceRule` subclasses. Business thresholds (escalation counts, time windows, auto-close conditions) are externalized into `rules.json`, so operations teams can tune behavior without a code deployment. A live-reload endpoint (`GET /api/rules/config`, `PUT /api/rules/config`) lets admins update rules at runtime from the dashboard.

### Background Auto-Close Worker
A `node-cron` job runs every 5 minutes and scans for alerts that meet time-based or metadata-based closure criteria. The worker is idempotent — re-runs on an already-closed alert are a no-op — and starts only after the MongoDB connection is established to avoid race conditions.

### React Analytics Dashboard
A single-page React 19 + Tailwind CSS dashboard served from Vercel, featuring:
- **Severity breakdown cards** — alert counts grouped by `high / medium / low`
- **7-day trend chart** — total, escalated, and auto-closed alerts over time (Recharts)
- **Top 5 offending drivers** — ranked by alert count from `metadata.driverId`
- **Recent alert activity table** — all states, collapsible, drill-down on click
- **Auto-closed alerts table** — filterable by last `24h / 48h / 7d`
- **Active rule config panel** — live view of `rules.json` directly from the API
- **Drill-down modal** — full state history timeline, metadata dump, and one-click resolve

---

## 🏗️ Architectural Decisions & Trade-offs

### Time & Space Complexity

**Trends aggregation — O(n) single-pass:**
The 7-day trends chart is powered by a single MongoDB aggregation pipeline that groups and counts alerts by date in one pass over the collection, rather than issuing 7 separate daily queries. This keeps the endpoint at O(n) in the number of documents scanned regardless of the number of days requested.

**Compound index — `{ status: 1, timestamp: -1 }`:**
The most frequent query pattern is filtering by status (e.g. `AUTO-CLOSED`) within a time window. This compound index satisfies both the equality filter and the sort in a single index scan, avoiding a full collection scan on every dashboard load.

**Alert history without an audit log table:**
Rather than maintaining a separate `AlertHistory` collection (which would double write load and require joins), state transitions are recorded as an array of `{ status, at }` entries appended to the alert document itself. History retrieval is O(1) — a single document fetch — and the space overhead is bounded by the number of transitions per alert, which is small and predictable.

### In-Memory Caching
The two most expensive aggregation endpoints — `/api/alerts/summary` (60s TTL) and `/api/alerts/trends` (5 min TTL) — are wrapped with a lightweight JavaScript `Map`-based cache in `services/cache.js`. This avoids re-running heavy aggregation pipelines on every dashboard refresh. The cache is invalidated on every `createAlert` and `resolveAlert` write, so data is never stale after a mutation. A Redis cache would be the natural next step for a multi-process deployment, but for a single-process Node server this keeps the dependency count minimal.

### OOP Rule Engine & Extensibility
`services/RuleEngine.js` uses strict ES6 inheritance so each rule type encapsulates its own evaluation logic and is independently testable. Adding a new alert type requires only: creating a new subclass of `BaseRule`, registering it in the registry object, and adding its thresholds to `rules.json` — no changes to the ingestion controller or the auto-close worker.

### JWT Authentication
Tokens are signed with `jsonwebtoken`, expire after 8 hours (matching a typical work shift), and are verified on every protected route via an `authMiddleware`. Passwords are hashed with `bcrypt` at 12 salt rounds. Email addresses are normalized to lowercase at both registration and login so `User@X.com` and `user@x.com` are treated as the same account.

---

## 🛠️ Tech Stack

**Backend**
| Package | Purpose |
|---|---|
| `express` v5 | HTTP server and routing |
| `mongoose` | MongoDB ODM |
| `jsonwebtoken` + `bcrypt` | Authentication |
| `node-cron` | Background auto-close worker |
| `cors` | Cross-origin requests from Vercel |
| `dotenv` | Environment variable loading |

**Frontend**
| Package | Purpose |
|---|---|
| `react` v19 + `react-dom` | UI |
| `vite` | Build tool and dev server |
| `tailwindcss` v4 | Utility-first styling |
| `recharts` | Trends line chart |

---

## 📁 Project Structure

```
├── server.js                  # express app entry point
├── routes/
│   ├── alertRoutes.js
│   ├── authRoutes.js
│   └── rulesRoutes.js
├── controllers/
│   ├── alertController.js     # crud, summary, trends, resolve
│   ├── authController.js      # register, login
│   └── rulesController.js     # live rules config read/write
├── models/
│   ├── Alert.js               # mongoose schema + compound index
│   └── User.js                # email normalization, role enum
├── services/
│   ├── RuleEngine.js          # base class + overspeed/feedback/compliance subclasses
│   └── cache.js               # lightweight in-memory ttl cache
├── jobs/
│   └── autoCloseWorker.js     # node-cron worker, runs every 5 mins
├── middleware/
│   └── authMiddleware.js      # jwt verification
├── rules.json                 # externalized rule thresholds (DSL)
└── client/                    # vite + react frontend
    ├── src/
    │   ├── Dashboard.jsx      # entire frontend — modular components
    │   └── main.jsx
    ├── vercel.json            # spa rewrite rule for vercel
    └── vite.config.js         # dev proxy → localhost:4000
```

---

## ⚙️ Local Setup

### Prerequisites
- Node.js 18+
- A MongoDB connection string (MongoDB Atlas free tier works)

### 1. Clone and install

```bash
git clone https://github.com/kartikeya042/SyncOps
cd SyncOps

# backend dependencies
npm install

# frontend dependencies
cd client && npm install && cd ..
```

### 2. Configure environment variables

Create a `.env` file in the project root:

```env
PORT=4000
MONGO_URI=mongodb+srv://<user>:<password>@cluster.mongodb.net/alert-escalation
JWT_SECRET=<your-64-char-random-secret>
CORS_ORIGIN=http://localhost:5173
```

Generate a secure `JWT_SECRET`:
```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

### 3. Run the servers

In two separate terminals:

```bash
# terminal 1 — backend (port 4000)
npm start

# terminal 2 — frontend (port 5173)
cd client && npm run dev
```

The dashboard will be available at `http://localhost:5173`. The Vite dev proxy forwards all `/api` calls to `http://localhost:4000`, so no manual CORS configuration is needed locally.

---

## 🌐 Production Environment Variables

**Render (backend):**
```
PORT           = (set automatically by render)
MONGO_URI      = mongodb+srv://...
JWT_SECRET     = <64-char secret>
CORS_ORIGIN    = https://syncops-dashboard.vercel.app
```

**Vercel (frontend):**
```
VITE_API_URL   = https://syncops-api.onrender.com
```

---

## 📡 API Reference

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `POST` | `/api/auth/register` | — | Create a new user |
| `POST` | `/api/auth/login` | — | Login, returns JWT |
| `POST` | `/api/alerts` | ✅ | Ingest a new alert |
| `GET` | `/api/alerts` | ✅ | List alerts (filter by `status`, `severity`, `since`, `limit`) |
| `GET` | `/api/alerts/summary` | ✅ | Severity breakdown + top 5 drivers |
| `GET` | `/api/alerts/trends` | ✅ | 7-day daily totals |
| `GET` | `/api/alerts/:id/history` | ✅ | Full state timeline for one alert |
| `PATCH` | `/api/alerts/:id/resolve` | ✅ | Mark an alert as RESOLVED |
| `GET` | `/api/rules/config` | ✅ | Read current `rules.json` |
| `PUT` | `/api/rules/config` | ✅ | Update `rules.json` at runtime |

### Example — ingest an alert

**cURL**
```bash
curl -X POST https://syncops-api.onrender.com/api/alerts \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-token>" \
  -d '{
    "alertid": "ALT-001",
    "sourceType": "overspeed",
    "severity": "high",
    "timestamp": "2026-02-24T10:00:00.000Z",
    "metadata": { "driverId": "DRV-101", "speed": 95, "limit": 60 }
  }'
```

**Postman**

1. Set method to **POST** and enter the URL:
   ```
   https://syncops-api.onrender.com/api/alerts
   ```

2. Under the **Headers** tab, add:
   | Key | Value |
   |---|---|
   | `Content-Type` | `application/json` |
   | `Authorization` | `Bearer <your-token>` |

3. Under the **Body** tab, select **raw → JSON** and paste:
   ```json
   {
     "alertid": "ALT-001",
     "sourceType": "overspeed",
     "severity": "high",
     "timestamp": "2026-02-24T10:00:00.000Z",
     "metadata": {
       "driverId": "DRV-101",
       "speed": 95,
       "limit": 60
     }
   }
   ```

4. Click **Send**. A `201 Created` response confirms the alert was ingested and the rule engine has evaluated it.

> **Getting your token:** First call `POST /api/auth/login` with your email and password in the body. Copy the `token` string from the response and paste it after `Bearer ` in the Authorization header.
