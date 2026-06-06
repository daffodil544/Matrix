# Cashflow — Run Instructions & Architecture Overview

🚀 **[Open the Project](https://gl-guardian.lovable.app)**

> AI-assisted 13-week cash flow forecasting for construction companies.
> Deterministic financials. Auditable numbers. Zero black-box calculations.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Environment Setup](#2-environment-setup)
3. [Data Ingestion](#3-data-ingestion)
4. [Running the Forecast](#4-running-the-forecast)
5. [Architecture Overview](#5-architecture-overview)
6. [Pipeline Stages (F0–F5)](#6-pipeline-stages-f0f5)
7. [Key Design Principles](#7-key-design-principles)
8. [Folder Structure](#8-folder-structure)

---

## 1. Prerequisites

| Requirement | Version | Purpose |
|---|---|---|
| Node.js | 18+ | Lovable / frontend runtime |
| Python | 3.11+ | Data pipeline scripts |
| Supabase account | — | Database + vector embeddings |
| OpenAI API key | GPT-4o | GL classification, copilot, risk scoring |
| Open-Meteo | Free tier | Weather forecasts (no key required) |
| OpenWeatherMap | Free tier | Weather consensus (API key required) |

---

## 2. Environment Setup

### Clone & install

```bash
git clone https://github.com/your-org/cashflow.git
cd cashflow
npm install          # frontend + pptxgenjs
pip install -r requirements.txt   # pipeline dependencies
```

### Environment variables

Copy `.env.example` to `.env` and fill in:

```bash
# Supabase
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_KEY=your-service-role-key   # required for embeddings writes

# OpenAI
OPENAI_API_KEY=sk-...

# Weather
OPENWEATHERMAP_API_KEY=your-owm-key

# Optional: FX rates
FX_API_KEY=your-fxrates-key         # leave blank to use cached ECB rates
```

### Supabase: enable pgvector

Run once in the Supabase SQL editor:

```sql
create extension if not exists vector;

-- GL mappings table with embedding column
alter table gl_mappings add column if not exists embedding vector(1536);

-- Semantic similarity index
create index on gl_mappings
using ivfflat (embedding vector_cosine_ops)
with (lists = 100);
```

---

## 3. Data Ingestion

There are two ingest paths: **GL mapping** (accounting exports) and **project data** (milestones, costs, customers).

### 3a. Upload a GL accounting export (XLSX)

Drag and drop an `.xlsx` file in the UI, or run via CLI:

```bash
python pipeline/ingest_gl.py \
  --file data/exports/accounting_export_2024.xlsx \
  --company-id acme-construction
```

What this does:

1. Reads the first worksheet; extracts `account_number` and `account_description` columns
2. Normalises descriptions (lowercase, strip punctuation, resolve Dutch synonyms)
3. Checks `gl_mappings` for exact matches — uses approved mapping if found
4. Runs vector similarity search against stored embeddings
5. Falls back to OpenAI classification if no strong match (confidence < 0.75)
6. Writes results to `gl_mappings` with `approved = false` pending human review
7. Displays classification review table in the UI

### 3b. Ingest project & milestone data

Projects can be loaded from a structured CSV or connected directly from an accounting system (see Data Sources doc):

```bash
python pipeline/ingest_projects.py \
  --file data/projects/active_projects.csv \
  --company-id acme-construction
```

Expected CSV columns:

```
project_id, project_name, region, contractor, start_date, end_date,
total_labour_cost, milestone_name, milestone_date, milestone_invoice_amount,
customer_id, customer_type
```

### 3c. Approve GL mappings (triggers learning loop)

After reviewing classifications in the UI:

- **Approve** a mapping → sets `approved = true`, generates embedding, stores in `classification_feedback`
- **Edit category** → saves correction as high-priority override, generates embedding
- Future uploads will match against approved mappings first, before calling OpenAI

---

## 4. Running the Forecast

### Via the UI

1. Navigate to the **Forecast Dashboard**
2. Select company and forecast start date
3. Click **Generate Forecast** — the pipeline runs automatically
4. Results appear as a 13-week table with per-week drill-down

### Via CLI (for testing or automation)

```bash
python pipeline/run_forecast.py \
  --company-id acme-construction \
  --start-date 2025-06-09 \
  --weeks 13
```

Output is written to `forecast_weeks` in Supabase and printed as a summary table:

```
Week   Cash In      Cash Out     Net Cash     Balance      Confidence
W1     €87,499      €45,200      €42,299      €42,299      91%
W2     €0           €38,100      -€38,100     €4,199       84%
W3     €148,000     €51,400      €96,600      €100,799     78%
...
```

### Forecast step order

Each run executes these steps in sequence:

```
1. Load GL-mapped cost data from Supabase
2. Detect recurring invoice patterns (monthly / quarterly / annual)
3. Schedule milestone invoices from project data
4. Apply seasonal index for weeks beyond milestone visibility
5. Calculate material, labour, and subcontractor costs
6. Fetch weather from Open-Meteo + OpenWeatherMap; compute consensus
7. Shift affected milestones by weather-derived lost days
8. Apply per-customer payment lag (historical or AI-predicted if confidence > 80%)
9. Run AI delay risk scoring per project
10. Aggregate weekly cash in / cash out / net cash / running balance
11. Score forecast confidence (0–100%) per week
12. Run anomaly detection across all weeks
13. Write results + audit JSON to forecast_weeks table
```

---

## 5. Architecture Overview

```
┌────────────────────────────────────────────────────────────────[...]
│                         FRONTEND (Lovable)                       │
│                                                                  │
│  ┌─────────────┐  ┌──────────────────┐  ┌────────────────────┐  │
│  │  GL Upload  │  │ Forecast Dashboard│  │   AI Copilot Panel │  │
│  │  & Review   │  │  13-week table   │  │  Natural language  │  │
│  │  Table      │  │  Audit drill-down│  │  Q&A on forecasts  │  │
│  └──────┬──────┘  └────────┬─────────┘  └─────────┬──────────┘  │
└─────────┼─────────────────┼───────────────────────┼────────────[...]
          │                 │                        │
          ▼                 ▼                        ▼
┌────────────────────────────────────────────────────────────────[...]
│                     SUPABASE (Backend)                           │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │  gl_mappings │  │forecast_weeks│  │  classification_      │  │
│  │  + embeddings│  │  + audit_json│  │  feedback + embeddings│  │
│  │  (pgvector)  │  │              │  │  (learning loop)      │  │
│  └──────────────┘  └──────────────┘  └───────────────────────┘  │
│                                                                  │
│  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ projects │  │ milestones│  │ customers│  │ weather_cache │  │
│  │ invoices │  │ materials │  │ payments │  │               │  │
│  │ labour   │  │subcontract│  │          │  │               │  │
│  └──────────┘  └───────────┘  └──────────┘  └───────────────┘  │
└────────────────────────────────────────────────────────────────[...]
          │                                        │
          ▼                                        ▼
┌────────────────────────┐          ┌──────────────────────────┐
│     OpenAI API         │          │     External APIs         │
│                        │          │                           │
│  • GL classification   │          │  • Open-Meteo (weather)   │
│  • Payment lag predict │          │  • OpenWeatherMap         │
│  • Delay risk scoring  │          │  • ECB FX rates (optional)│
│  • Copilot Q&A         │          │                           │
└────────────────────────┘          └──────────────────────────┘
```

### Component responsibilities

**Frontend (Lovable / React)**
Handles file upload, classification review, forecast display, drill-down audit views, and the natural-language copilot interface. Reads from and writes to Supabase via the JS client.

**Supabase (Postgres + pgvector)**
Single source of truth for all financial data, GL mappings, embeddings, forecast results, and audit JSON. Row-level security ensures per-company data isolation. The `forecast_weeks.audit_json` co[...]

**OpenAI API**
Used in four distinct, isolated roles — none of which can directly modify a financial number:
- GL account classification (returns category + confidence)
- Payment lag prediction (used only when confidence > 80%)
- Project delay risk scoring (returns probability + expected days)
- Copilot natural-language explanations (reads from forecast data, does not recalculate)

**Weather APIs**
Two independent providers fetched per forecast run. Consensus = average of both. If provider disagreement exceeds 30%, the week is flagged `LOW_WEATHER_CONFIDENCE`. Results are cached in `weather[...]

---

## 6. Pipeline Stages (F0–F5)

The forecast pipeline is structured as six sequential stages. Each stage writes its output to Supabase before the next stage begins, enabling restarts at any point.

| Stage | Name | Input | Output |
|---|---|---|---|
| F0 | Data Load | Supabase tables | Validated in-memory dataset |
| F1 | Revenue Forecast | Invoices, milestones, seasonal index | Weekly revenue schedule |
| F2 | Cost Forecast | Projects, materials, labour, subcontractors | Weekly cost schedule |
| F3 | Weather & Shifting | Open-Meteo + OWM APIs, milestones | Shifted milestone dates, labour adjustments |
| F4 | Cash Timing | Customer payment history, AI lag prediction | Per-invoice cash arrival week |
| F5 | Aggregation & Scoring | F1–F4 outputs | 13-week table, confidence scores, anomaly flags, audit JSON |

---

## 7. Key Design Principles

**Determinism first.** Every financial number is the output of an explicit, traceable formula. AI outputs are inputs to decision logic — they never touch a cash value directly.

**Confidence gating.** AI estimates replace defaults only when the model's own confidence exceeds the threshold (80% for payment lag, 75% for GL classification). Below threshold, the system falls[...]

**Full audit trail.** `forecast_weeks.audit_json` stores the complete derivation chain: source invoice → milestone → weather adjustment → payment lag → cost allocation → final value. Us[...]

**Learning loop.** Every human correction to a GL mapping generates an embedding and is stored as a high-priority override. The system gets more accurate with use, without retraining.

**Weather consensus.** Using two independent weather providers and averaging their output reduces single-provider error. Disagreement flags are surfaced in the UI so users can decide how much to [...]

---

## 8. Folder Structure

```
cashflow/
├── pipeline/
│   ├── ingest_gl.py          # GL XLSX ingestion + classification
│   ├── ingest_projects.py    # Project / milestone / customer import
│   ├── run_forecast.py       # Main forecast orchestrator (F0–F5)
│   ├── stages/
│   │   ├── f1_revenue.py
│   │   ├── f2_costs.py
│   │   ├── f3_weather.py
│   │   ├── f4_cash_timing.py
│   │   └── f5_aggregation.py
│   └── ai/
│       ├── gl_classifier.py  # OpenAI GL classification
│       ├── lag_predictor.py  # Payment lag prediction
│       ├── delay_scorer.py   # Project delay risk scoring
│       └── copilot.py        # Natural language Q&A
├── supabase/
│   ├── migrations/           # SQL migration files
│   └── seed/                 # Sample data for local dev
├── data/
│   ├── exports/              # Drop XLSX accounting exports here
│   ├── projects/             # Drop project CSVs here
│   └── sample/               # Example files for onboarding
├── src/                      # Lovable / React frontend
├── .env.example
├── requirements.txt
├── package.json
└── README.md                 # This file
```

---

## Quick Start (TL;DR)

```bash
# 1. Clone and install
git clone https://github.com/your-org/cashflow.git && cd cashflow
npm install && pip install -r requirements.txt

# 2. Add credentials to .env

# 3. Enable pgvector in Supabase SQL editor (see §2)

# 4. Drop an accounting export into data/exports/ and run:
python pipeline/ingest_gl.py --file data/exports/your_export.xlsx --company-id your-co

# 5. Approve GL mappings in the UI

# 6. Run the forecast
python pipeline/run_forecast.py --company-id your-co --start-date 2025-06-09 --weeks 13
```
