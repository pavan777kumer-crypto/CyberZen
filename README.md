# CyberZen

## Overview

Security operations teams are flooded with alerts — duplicates, near-duplicates,
and low-value noise bury the handful of alerts that actually matter. SentryGrid
is a full-stack demo application that ingests raw security alerts and runs them
through a real machine-learning pipeline to cut that noise down to a small set
of prioritized, explainable incidents.

## Problem Statement

A typical SOC analyst can't manually triage thousands of daily alerts. Most are
duplicates or minor variations of the same underlying activity. SentryGrid
automates triage: it deduplicates, clusters, scores anomalies, predicts
severity, computes an explainable priority score, and correlates related
alerts into incidents — then reports exactly how much alert volume that
removed.

## Features

- JWT authentication with ADMIN / SECURITY_ANALYST roles
- CSV / JSON alert upload, plus manual alert entry, with server-side validation
- A real preprocessing pipeline (missing values, timestamp/IP/severity normalization)
- Duplicate detection (alert type + IP + time window + TF-IDF message similarity)
- TF-IDF + KMeans clustering of alert messages into labeled groups
- Isolation Forest anomaly detection over engineered alert features
- Random Forest severity classifier (LOW/MEDIUM/HIGH/CRITICAL), trained once and persisted to disk
- Explainable 0–100 priority scoring
- Alert correlation into incidents (shared IPs, category, cluster, time window)
- Alert Fatigue Reduction metric: `(total_alerts − final_incidents) / total_alerts × 100`
- Per-alert dynamic AI explanation text generated from the alert's own features
- Dark, data-dense React dashboard with charts backed entirely by live API data
- Windows batch scripts for one-command setup and start

## AI / ML Techniques

| Component | Technique | File |
|---|---|---|
| Deduplication | Rule matching + TF-IDF cosine similarity | `backend/app/ml/deduplication.py` |
| Clustering | `TfidfVectorizer` + `KMeans` | `backend/app/ml/clustering.py` |
| Anomaly detection | `IsolationForest` over engineered features | `backend/app/ml/anomaly_detection.py` |
| Severity prediction | `RandomForestClassifier`, trained + persisted with `joblib` | `backend/app/ml/severity_model.py` |
| Priority scoring | Rule-based, explainable 0–100 score | `backend/app/ml/priority.py` |
| Explanations | Feature-driven natural-language text | `backend/app/ml/explanations.py` |

**Training data methodology:** No public labeled SOC severity dataset ships
with this project, so `backend/app/ml/training_data.py` generates a synthetic
but structurally realistic dataset: severity labels are derived from a
domain-rule score (category risk + asset criticality + sensitive-port usage +
off-hours activity + random jitter), not assigned randomly or hand-picked per
class. The Random Forest has to learn genuine decision boundaries from that
score, and typically reaches ~75% held-out accuracy on the 4-class problem
(vs. ~39% for always guessing the majority class) — you can reproduce this
yourself from the AI Model Insights page's "Retrain model" button.

## Technology Stack

**Backend:** Python 3.10+, FastAPI, Uvicorn, SQLAlchemy, SQLite, Pydantic, python-jose, passlib/bcrypt
**AI/ML:** pandas, NumPy, scikit-learn (TF-IDF, KMeans, Isolation Forest, Random Forest), joblib
**Frontend:** React, Vite, Tailwind CSS, Axios, React Router, Recharts, lucide-react
**Testing:** Pytest, FastAPI TestClient

## Folder Structure

```
ai-cyber-alert-fatigue/
├── backend/
│   ├── app/
│   │   ├── main.py                 FastAPI app + router registration
│   │   ├── core/                   config, security (JWT/bcrypt), auth dependency
│   │   ├── database/                SQLAlchemy engine/session, init_db
│   │   ├── models/                  User, Alert, AlertCluster, Incident, AnalysisRun
│   │   ├── schemas/                 Pydantic request/response models
│   │   ├── api/                     auth, alerts, analysis, incidents, dashboard, ml routes
│   │   ├── ml/                      preprocessing, dedup, clustering, anomaly, severity, priority, explanations
│   │   └── services/                alert ingestion, analysis orchestration, incident correlation
│   ├── tests/                       pytest suite (auth, alerts, ML pipeline, dashboard, incidents)
│   ├── ml_models/                   persisted joblib model (created on first train)
│   ├── requirements.txt
│   └── seed.py                      idempotent demo data + pipeline seeding
├── frontend/
│   └── src/                         pages, components, layouts, services, hooks
├── data/
│   ├── sample_alerts.csv            60-row example upload file
│   └── training_alerts.csv          200-row example upload file
├── setup.bat / start.bat / stop.bat
└── README.md
```

## Windows Setup

Requirements: Python 3.10+ and Node.js 18+ on PATH.

```
setup.bat
```

This creates a virtualenv, installs backend + frontend dependencies, creates
the SQLite database, seeds demo users and 500+ demo alerts, runs the full AI
pipeline once so the dashboard isn't empty, and trains the Random Forest
severity model.

## Start Command

```
start.bat
```

Opens two terminal windows (backend + frontend). To stop, close those windows
or run `stop.bat`.

- **Backend URL:** http://localhost:8000 (interactive API docs at `/docs`)
- **Frontend URL:** http://localhost:5173

## Demo Credentials

| Role | Email | Password |
|---|---|---|
| Admin | `admin@demo.local` | `Admin@123` |
| Security Analyst | `analyst@demo.local` | `Analyst@123` |

## How to Upload Alerts

Go to **Alert Upload** in the sidebar. Drop a `.csv` or `.json` file (see
`data/sample_alerts.csv` for the expected columns), or use the manual entry
form on the right for a single alert. Unsupported file types and rows missing
required fields or containing invalid IP addresses are rejected with a
specific reason per row.

## How to Run AI Processing

Go to **Dashboard** and click **Run AI Analysis**. This runs deduplication →
clustering → anomaly detection → severity prediction → priority scoring →
explanation generation → incident correlation, in that order, over every
alert currently in the database, and records the run in `analysis_runs`.

## How to View the Dashboard

The Dashboard, Alert Explorer, Incidents, and Fatigue Analytics pages all
read live from the backend (`/api/dashboard/*`, `/api/alerts`,
`/api/incidents`) — nothing is hardcoded in the frontend.

## How the Alert Fatigue Calculation Works

```
Alert Reduction % = ((Total Alerts − Final Actionable Incidents) / Total Alerts) × 100
```

"Final actionable incidents" = correlated incidents + any non-duplicate alert
that wasn't grouped into an incident. This number is computed fresh on every
analysis run from the actual alert set — see
`backend/app/services/analysis_service.py`.

## Internship Demo Flow (5 minutes)

1. **Login** as `analyst@demo.local` — dashboard already has seeded results.
2. **Alert Upload** — upload `data/sample_alerts.csv`, point out per-row validation.
3. **Dashboard** — click **Run AI Analysis**, watch the counts update live.
4. **Alert Explorer** — filter by `ANOMALY`, open one alert, read its AI Explanation.
5. **Incidents** — show a correlated incident, change its status to `INVESTIGATING`.
6. **Fatigue Analytics** — show the reduction funnel and the computed %.
7. **AI Model Insights** — show the three models and retrain the Random Forest live.

## Future Enhancements

- Swap the synthetic severity training set for a real labeled SOC dataset
- Streaming ingestion (Kafka/webhook) instead of batch upload
- Per-analyst feedback loop to fine-tune priority weighting over time
- Role-based incident assignment and SLA tracking
- Postgres instead of SQLite for multi-user production deployments

## Testing

```
cd backend
venv\Scripts\activate
pytest -v
```

The suite covers registration/login, JWT-protected routes, alert creation,
CSV upload + validation, the full ML pipeline (preprocessing through Random
Forest), dashboard endpoints, and incident status updates.

**Note on this repository's build/test history:** every ML module
(preprocessing, deduplication, TF-IDF/KMeans clustering, Isolation Forest,
and the Random Forest severity model, including a full train+predict cycle)
was executed directly in the environment this project was built in and
produced real, non-fabricated output — for example the Random Forest reached
75.5% accuracy on a held-out synthetic set in that run. The FastAPI server
and `npm run dev` were **not** executed end-to-end in that build environment,
because it has no network access to `pip install fastapi/sqlalchemy/...` or
`npm install`. All backend Python files pass `py_compile`, and the API/route
wiring follows the same patterns already verified by direct execution. Please
run `pytest -v` after `setup.bat` completes on your machine to confirm the
full stack, including FastAPI request handling, end-to-end.
