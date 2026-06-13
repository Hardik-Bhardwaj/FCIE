# FCIE — Fundability & Conversion Intelligence Engine

**AI operating intelligence layer for student financing and admissions platforms.**

A full-stack marketing + prototype platform that demonstrates how an ML-powered engine can lift student loan conversion rates from **6.2% → 9.2%** (+₹25Cr/year in funded loans).

> Built for edtech and fintech founders running student financing portals in India.

**Live demo:** [[https://fcie.replit.app](https://fcie.replit.app)](https://journey-insight--hardikb2501.replit.app/)

---

## What's in this demo

### Marketing site (`artifacts/fcie-website`)
A single-page React site with Apple-aesthetic design (white, Inter font, bold numbers). Sections:

| Section | What it shows |
|---|---|
| Hero | Core value prop — 6.2% → 9.2% conversion, +₹25Cr/year |
| Results Grid | 4 key metrics from the model (AUC, lift, funded/month, revenue) |
| Six Modules | FCIE's six intelligence modules (Fundability Score, Reason Engine, etc.) |
| Interactive Funnel | Hover to see root causes at each drop-off stage |
| Risk Analyzer | **Live JS scoring engine** — enter student parameters, get a real probability score |
| Counselor Queue | **Live animated queue** — shows prioritized students with inactivity timers |
| Journey Timeline | 30-day split animation: with FCIE vs without FCIE |
| Platform Comparison | Build vs FCIE vs Do Nothing table |
| ROI Table | Full unit economics — ₹25Cr/year calculation |
| Document Intelligence | Document completeness scoring examples |
| Partnership Roadmap | Week-by-week integration timeline |
| Pilot Request Form | **Real form → stored in PostgreSQL** |
| ChatBot | Floating assistant with 13-topic knowledge base (no API key needed) |

### API Server (`artifacts/api-server`)
Express 5 backend serving:
- `GET /api/healthz` — health check
- `POST /api/pilot` — saves pilot form submissions to the database

### Database (`lib/db`)
PostgreSQL via Drizzle ORM. One table:
- `pilot_requests` — stores founder name, company, email, role, platform scale, message, timestamp

---

## The ML model behind the numbers

All numbers come from an XGBoost model trained on ~3,000 historical student loan applications with 35 features:

**Feature categories:**
- Academic: GRE/GMAT, university tier, GPA
- Financial: family income, co-applicant, existing liabilities
- Documentation: completeness score (0–1), document types uploaded
- Lender fit: match score between student profile and lender criteria
- Engagement: days since last activity, RFM signals, session count
- Counselor: response time, touchpoint count, escalation rate

**Model outputs:**
- Fundability probability (0–1)
- Top 3 SHAP-based reason codes (why this score)
- Recommended action (WhatsApp nudge, counselor callback, lender reroute, etc.)

**Achieved results:**
- AUC: 0.80–0.87
- Conversion lift: 6.2% → 9.2% (+48% relative)
- +55 funded students/month
- +₹2.1Cr/month additional loan disbursements

The `RiskAnalyzer` component on the website reimplements the model's scoring logic in JavaScript using the same feature weights and thresholds, so it works without any Python runtime.

---

## Tech stack

| Layer | Tech |
|---|---|
| Frontend | React 18, Vite, Tailwind CSS v4, Framer Motion |
| UI components | shadcn/ui (Radix UI primitives) |
| Routing | Wouter |
| Backend | Node.js 24, Express 5, TypeScript 5.9 |
| Database | PostgreSQL, Drizzle ORM, drizzle-zod |
| Validation | Zod v4 |
| Package manager | pnpm workspaces |
| Build | esbuild (API), Vite (frontend) |
| Fonts | Inter (Google Fonts), Space Mono |

---

## Project structure

```
fcie-project/
├── artifacts/
│   ├── api-server/              # Express 5 API
│   │   └── src/
│   │       ├── app.ts           # Express setup, CORS, logging
│   │       ├── index.ts         # Entry point
│   │       └── routes/
│   │           ├── health.ts    # GET /api/healthz
│   │           └── pilot.ts     # POST /api/pilot
│   │
│   └── fcie-website/            # React + Vite site
│       └── src/
│           ├── App.tsx          # Root — router + ChatBot
│           ├── pages/home.tsx   # All page sections assembled here
│           └── components/
│               ├── ChatBot.tsx          # Floating assistant widget
│               ├── CounselorQueue.tsx   # Live animated priority queue
│               ├── InteractiveFunnel.tsx # Hover funnel with root causes
│               ├── JourneyTimeline.tsx  # 30-day side-by-side animation
│               ├── PilotForm.tsx        # Lead capture → DB
│               ├── PlatformComparison.tsx # Decision table
│               └── RiskAnalyzer.tsx     # Live JS scoring engine
│
├── lib/
│   ├── db/                      # PostgreSQL + Drizzle ORM
│   │   └── src/schema/
│   │       └── pilot-requests.ts # pilot_requests table
│   ├── api-spec/                # OpenAPI contract
│   ├── api-zod/                 # Generated Zod schemas
│   └── api-client-react/        # Generated React Query hooks
│
└── scripts/
    └── post-merge.sh            # Runs after deploys (DB push)
```

---

## Running locally

### Prerequisites
- Node.js 20+
- pnpm (`npm install -g pnpm`)
- PostgreSQL database

### Setup

```bash
# 1. Clone and install
git clone https://github.com/your-org/fcie-project.git
cd fcie-project
pnpm install

# 2. Environment variables
cp .env.example .env
# Edit .env and set DATABASE_URL and SESSION_SECRET

# 3. Push database schema
pnpm --filter @workspace/db run push

# 4. Start API server (terminal 1)
PORT=5000 pnpm --filter @workspace/api-server run dev

# 5. Start website (terminal 2)
PORT=3000 BASE_PATH=/ pnpm --filter @workspace/fcie-website run dev

# 6. Open http://localhost:3000
```

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `SESSION_SECRET` | Yes | Random string for session signing |

---

## Connecting live student data

The current site uses a **static scoring engine** (JS reimplementation of XGBoost logic). To connect live student data from your portal:

### Option A — Webhook (recommended for production)
Fire an event from your Python backend whenever a student action occurs:

```python
import requests

requests.post("https://your-fcie-app.com/api/events", json={
    "studentId": "GR10041",
    "event": "document_uploaded",
    "docCompleteness": 0.72,
    "daysInactive": 0,
    "funnelStage": "docs_submitted",
    "gre_score": 315,
    "income_band": "medium"
})
```

### Option B — Python microservice with real model
Export your trained model from Colab:

```python
import joblib
joblib.dump(model, 'fcie_model.pkl')
```

Then run a Flask scoring service alongside the API:

```python
from flask import Flask, request, jsonify
import joblib, numpy as np

app = Flask(__name__)
model = joblib.load('fcie_model.pkl')

@app.route('/score', methods=['POST'])
def score():
    features = request.json['features']
    prob = model.predict_proba([features])[0][1]
    return jsonify({"fundability_score": round(prob, 3)})
```

### Option C — Batch re-scoring (simplest to start)
Run a nightly cron job that re-scores all active students and pushes scores back to FCIE.

---

## Key business metrics (from model evaluation)

| Metric | Without FCIE | With FCIE | Delta |
|---|---|---|---|
| Conversion rate | 6.2% | 9.2% | +48% |
| Funded students/month | ~185 | ~240 | +55 |
| Revenue/month | ₹7.4Cr | ₹9.5Cr | +₹2.1Cr |
| Revenue/year | — | — | +₹25Cr |
| Model AUC | — | 0.80–0.87 | — |

---

## License

MIT
