# RhettBooked

**Describe a BU event in one sentence. Get a plan that's costed, checked against real data, and stress-tested before day one.**

🔗 **Live demo:** https://eventtwin.vercel.app
🏆 Built for **BU Spark! DS+X Hackathon 2026** (CollegeHack) and **Hackachusetts 2026** (Civic Action)
🛠️ Built with **Kiro** using spec-driven development

---

## What it does

An organizer types something like:

> *"300-person overnight AI hackathon at BU, Oct 24–25, students, budget $15k, needs lunch, dinner, midnight snack, breakfast"*

EventTwin returns one organized plan:

| Section | What you get |
|---|---|
| **Summary** | Recommended venue, likely cost and range, readiness score, top risk |
| **Venue** | Room assignments per schedule block from real BU event-space data, utilization, Wi-Fi and power consumption |
| **Getting there** | Nearest MBTA stops with step-free flags, planned disruptions, bus routes, parking |
| **Budget** | Line items that always add up to the total, cost per attendee, over/under budget |
| **Food & water** | Meals matched to open BU dining halls, cost estimate, serving lines, water plan |
| **People & sponsors** | Judges and speakers from real research data, sponsor categories to close the budget gap |
| **Plan options** | Lean / Balanced / Premium comparison |
| **The drill** | A digital twin replays the event in 15-minute steps, finds what breaks, applies fixes and re-runs |
| **Trust & export** | Data labels, privacy report, sensitivity analysis, PDF and CSV export |

### Try the two demo briefs
- **Brief A (Oct 24–25):** the event we built this at. The drill finds power circuits overloading from 10:15 PM, and the fixes raise readiness from 70 to 85.
- **Brief B (Oct 31–Nov 1):** the same event a week later. EventTwin catches the **real planned Green Line closure** on its own and recommends the fare-free Route 57 shuttle.

---

## Why it's not an AI wrapper

The AI is optional. With `LLM_PROVIDER=none`, everything still works. Every number comes from a model:

| Problem | Method |
|---|---|
| Turnout | Beta show-up prior + Monte Carlo (500 draws), giving p10 / p50 / p90 |
| Arrival curve | Learned from **50,106 real Bluebikes trips** near BU (Nov 2024), blended with **245,555 Boston Numina sensor readings** |
| Room assignment | **Constraint optimization** (OR-Tools CP-SAT): layout-aware capacity, air-wall exclusivity, accessibility, cost + walking + unused-capacity objective |
| Wi-Fi / power | Device and load models per 15-minute block, reported as consumption |
| Budget | Monte Carlo over turnout and rate-card ranges |
| Air quality (drill) | CO₂ mass balance: dC/dt = (N·G − Q·(C − C_out)) / V |
| Queues (drill) | Serving and restroom queue models |
| Robustness | One-at-a-time sensitivity analysis (tornado chart) |
| People | TF-IDF similarity × citations × recency over **2,000 real BU research works** (OpenAlex) |
| Sponsors | Topic matching to sponsor categories, with a suggested ask that covers the budget gap |

### Validation
The learned arrival curve was tested with a **temporal holdout** (fit on Nov 1–15, tested on Nov 16–30):

| Day type | Fitted RMSE | Fitted Spearman | Baseline (trapezoid) RMSE | Baseline Spearman |
|---|---|---|---|---|
| Weekend | **0.107** | **0.974** | 0.499 | 0.006 |
| Weekday | 0.125 | 0.832 | 0.644 | 0.470 |

Details and caveats are in [`docs/METHODS.md`](docs/METHODS.md).

---

## Data

Every number in the app carries one of these labels:

- **MEASURED:** from a real dataset
- **ASSUMED:** a documented assumption with a range
- **PLACEHOLDER:** illustrative stand-in data
- **SIMULATED:** produced by a model

| Source | Used for | Label |
|---|---|---|
| BU event spaces (38 spaces, capacities by layout) | Venue optimizer | MEASURED |
| BU dining locations and hours | Food & water | MEASURED |
| BU-area parking | Getting there | MEASURED |
| MBTA stops, alerts and bus routes near BU | Getting there, drill | MEASURED |
| Bluebikes trip data, Nov 2024 | Arrival curve | MEASURED (derived) |
| City of Boston Numina street sensors | Arrival curve | MEASURED (derived) |
| OpenAlex works (BU, 2022+) | Judges and speakers | MEASURED |
| Placeholder rooms for buildings without event-space data | Venue fallback | PLACEHOLDER |
| Rate cards (catering, rentals, staffing) | Budget | ASSUMED |
| Sponsor archetypes (no real company names) | Sponsors | PLACEHOLDER |

Large raw files (the Bluebikes trip file, raw OpenAlex pages) are gitignored. Only compact derived tables are committed.

---

## Privacy & ethics

- **Scrubber:** detects BU IDs, emails, phone numbers and MAC addresses / BSSIDs. The residual scan after scrubbing must be zero.
- **Reversible tokens:** deterministic HMAC-SHA256 tokens, with the token map kept in a **separate SQLite vault** and every detokenization written to an audit log.
- **Aggregates only:** uploaded registration lists produce totals (headcount, dietary and accessibility counts). Any cell with fewer than 5 people is suppressed.
- **People data:** public professional information only (name, institution, research topics, profile link). No contact scraping and no inference of demographics.
- **Known limitation:** name detection currently relies on regex and misses names. Presidio NER is planned.
- **Data caveats shown in the app:** one month of Bluebikes data; Bluebikes riders are not all attendees; Numina is citywide and from 2020–21; crash datasets define crashes differently.

More in [`docs/ETHICS.md`](docs/ETHICS.md).

---

## Architecture

```
Event description
      ↓
Intake (rule-based parser; LLM optional)
      ↓
Privacy layer (scrub → tokenize → aggregate)
      ↓
DuckDB: BU spaces, dining, parking, MBTA, Bluebikes/Numina profiles, OpenAlex
      ↓
Plan models: attendance → venue optimizer → tech → food & water → transport → people → budget
      ↓
Digital-twin drill: occupancy, power, getting home, network, air, queues → verdict → fixes → re-run
      ↓
FastAPI  ──▶  React frontend (light/dark, BU theme)
```

## Tech stack

- **Backend:** Python 3.11, FastAPI, DuckDB, pandas, NumPy, SciPy, OR-Tools, SimPy, scikit-learn, pydantic
- **Frontend:** React, TypeScript, Vite, Tailwind, framer-motion
- **Hosting:** Vercel (frontend), Render (backend, Docker)
- **Built with:** Kiro (steering docs, specs, hooks)

## Repo structure

```
backend/app/
  brief/        intake parser, follow-up questions
  data/         loaders, DuckDB build, Bluebikes/OpenAlex fetchers, placeholder generator
  plan/         attendance, venue optimizer, tech, food & water, transport, people, sponsors, budget, options
  drill/        digital twin, power, network, air, queues, mobility, sensitivity, verdict
  privacy/      scrubber, tokenizer, vault, risk report
  agent/        tools, grounded explainer, RAG
backend/tests/  pytest suite
frontend/src/   React app (cards, budget, drill, sections, demo snapshots)
config/         assumptions.yaml
data/           raw (real BU + V3–V5 data), placeholder, reference
deploy/         demo DuckDB + cache for the hosted backend
docs/           METHODS, ETHICS, KIRO_USAGE, design mocks
.kiro/          steering docs, specs, hooks
```

---

## Running locally

### 1. Environment
Create `.env` in the repo root (it is gitignored):

```
TOKEN_KEY=<python -c "import secrets;print(secrets.token_hex(32))">
VAULT_PATH=secrets/vault.sqlite
LLM_PROVIDER=none
MBTA_API_KEY=            # optional
```

### 2. Backend
```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python -m backend.app.data.build_db
uvicorn backend.app.main:app --reload
```
API docs: http://localhost:8000/docs

### 3. Frontend
```bash
cd frontend
npm install
npm run dev
```
App: http://localhost:5173

### 4. Tests
```bash
pytest backend/tests        # 182 tests
cd frontend && npm test     # 7 tests
```

### Deployment
- **Backend:** Render Blueprint via `render.yaml` (Docker, repo root as build context, `/health` check). Set `TOKEN_KEY`, `LLM_PROVIDER`, `EVENTTWIN_DB=deploy/eventtwin_demo.duckdb` and `ALLOWED_ORIGINS`.
- **Frontend:** Vercel, root directory `frontend`, with `VITE_API_URL` pointing to the Render service.
- If the API is unreachable, the frontend loads saved demo snapshots and shows a "Demo snapshot" badge.

---

## How we used Kiro

- **Steering docs** (`.kiro/steering/`): product rules, tech stack, repo structure and the BU design system, applied to every task.
- **Specs** (`.kiro/specs/`): `eventtwin-core` (14 requirements, correctness properties, 23 prioritized tasks) and `design-v2` (the redesign).
- **Hooks:** tests on save, a privacy guard for personal data, a label guard for unlabelled numbers, and a usage log.
- **Testing:** 189 tests, including property-style checks (CO₂ steady state, air-wall exclusivity, explainer grounding, budget rows summing to the total).

See [`docs/KIRO_USAGE.md`](docs/KIRO_USAGE.md) for the task-by-task log.

---

## Status

### ✅ Done
- **v1:** the full planning pipeline, digital-twin drill, privacy layer, people/sponsors, plan options, Trust & export, deployment
- **Design v2:** BU-themed light/dark redesign with smooth motion, accessible status pills and a responsive layout

### 🚧 In progress
- **Research agent:** a Gemini-powered agent using an **MCP server** of read-only tools (web search with grounding, OpenStreetMap, OpenAlex, plan context) to find **real caterers and people** with guardrails:
  - every result must cite a source
  - caterers are cross-checked against OpenStreetMap
  - public professional information only
  - prompt-injection defence and tool-call limits
- **Cost model v2:** BU resources (rooms, Wi-Fi, power, water) count as $0 and are reported as **consumption ranges**; the budget counts only spend outside the university, with an optional internal-chargebacks toggle
- **Three room arrangement plans:** Single Hub, Anchor + Breakouts and Surge-Ready, with a customizable budget that recalculates instantly from the Monte Carlo samples
- **Drill as an optional "Stress-test this plan" button**
- Presidio NER for name detection, and MiniLM embeddings as an option alongside TF-IDF

---

## Limitations

- Event-space data covers 38 BU spaces; other buildings use labelled placeholder rooms.
- Overnight access and AV availability are assumed and flagged as "Confirm with BU Reservations".
- The arrival curve comes from one month of mobility data, not from attendance records.
- The drill is a planning model, not a guarantee.

## Data credits & licences

MBTA (public API terms) · Bluebikes system data (Bluebikes data licence) · City of Boston Open Data (Numina sensors) · OpenAlex (CC0) · BU event-space and dining information from bu.edu pages · Hackachusetts 2026 campus data portal. Check each source's terms before reusing the data.

## Team

*[Add team names and roles]*
