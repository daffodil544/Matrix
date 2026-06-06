# Cashflow — Data Sources & Accounting Integrations

> What data the system ingests, where it comes from, and how external APIs are used.

---

## Table of Contents

1. [Accounting System Integrations](#1-accounting-system-integrations)
2. [XLSX Import Format (Universal)](#2-xlsx-import-format-universal)
3. [Sample Data Schema](#3-sample-data-schema)
4. [External APIs](#4-external-apis)
5. [FX & Currency Handling](#5-fx--currency-handling)
6. [Data Quality & Validation Rules](#6-data-quality--validation-rules)
7. [Adding a New Integration](#7-adding-a-new-integration)

---

## 1. Accounting System Integrations

Cashflow ingests data from accounting systems via XLSX export (universal) or direct API connector (where available). All connectors normalise data to the same internal schema before it enters the pipeline.

### Supported systems

| System | Integration method | GL export | Invoice history | Payment history | Notes |
|---|---|---|---|---|---|
| **Exact Online** | XLSX export | ✅ | ✅ | ✅ | Most common in NL construction |
| **Snelstart** | XLSX export | ✅ | ✅ | ✅ | Dutch SME accounting |
| **Twinfield** | XLSX export | ✅ | ✅ | ✅ | Multi-entity support |
| **AFAS** | XLSX export | ✅ | ✅ | ✅ | HR + project cost integration |
| **QuickBooks Online** | XLSX export | ✅ | ✅ | ✅ | International projects |
| **Xero** | XLSX export | ✅ | ✅ | ✅ | International projects |
| **SAP (B1 / S4)** | XLSX export | ✅ | ✅ | ✅ | Larger contractors |
| **Custom / unknown** | XLSX (any format) | ✅ | — | — | AI classifies from description alone |

### Planned direct API connectors

The following connectors are on the roadmap for post-MVP:

- **Exact Online REST API** — real-time GL sync, no manual export needed
- **AFAS REST API** — project costs and HR sync
- **Twinfield SOAP/REST** — multi-entity GL pull

Until direct connectors are live, the XLSX import path covers all systems above.

---

## 2. XLSX Import Format (Universal)

The system is designed to handle **messy, real-world Excel exports** — merged cells, extra metadata rows, inconsistent column naming, and mixed Dutch/English headers are all handled automatically.

### What the parser does

1. Opens the first worksheet by default (configurable via `--sheet-index`)
2. Scans the first 10 rows to detect the header row (looks for account number / description patterns)
3. Extracts these two columns (fuzzy column name matching):

| Internal field | Accepted column name variants |
|---|---|
| `account_number` | `Grootboek`, `Rekening`, `Account`, `GL Code`, `Code`, `Nr`, `Number`, `Rekeningnummer` |
| `account_description` | `Omschrijving`, `Naam`, `Description`, `Account Name`, `Label`, `Grootboekrekening` |

4. Ignores all other columns (balance amounts, periods, cost centres — these are not used for GL classification)
5. Strips formatting, merged cell artefacts, empty rows, and subtotal rows
6. Shows a preview of the first 20 parsed rows before processing begins

### Minimum required columns

Only two columns are strictly required:

```
account_number    account_description
400              Materiaalkost bouw
410              Onderaanneming
610              Brandstofkosten voertuigen
```

### Accepted file formats

`.xlsx` only. Legacy `.xls` files must be converted first:

```bash
python pipeline/utils/convert_xls.py --file old_export.xls
# Outputs: old_export.xlsx
```

---

## 3. Sample Data Schema

The following describes the internal data model that all integrations normalise to. Sample CSV files for local development are in `data/sample/`.

### Invoices (`data/sample/invoices.csv`)

```csv
invoice_id, project_id, customer_id, amount_eur, invoice_date, due_date, status, gl_category
INV-2024-001, PROJ-AMT-01, CUST-001, 148000.00, 2024-11-15, 2024-12-15, paid, Revenue
INV-2024-002, PROJ-SCH-02, CUST-002, 210000.00, 2024-12-01, 2025-01-15, outstanding, Revenue
```

### Payments (`data/sample/payments.csv`)

```csv
payment_id, invoice_id, payment_date, amount_eur
PAY-001, INV-2024-001, 2024-12-22, 148000.00
```

### Projects (`data/sample/projects.csv`)

```csv
project_id, name, region, contractor, start_date, end_date, total_labour_cost_eur, status
PROJ-AMT-01, Amstel Office, Amsterdam-Centrum, Bouwbedrijf de Vries, 2024-09-01, 2025-03-31, 320000, active
PROJ-SCH-02, Schiphol Hub, Haarlemmermeer, Van Wijnen, 2024-11-01, 2025-06-30, 510000, active
PROJ-WPT-03, Westpoort Logistics, Westpoort, Dura Vermeer, 2025-01-15, 2025-08-31, 180000, active
```

### Milestones (`data/sample/milestones.csv`)

```csv
milestone_id, project_id, name, planned_date, invoice_amount_eur
MS-001, PROJ-AMT-01, Casco oplevering, 2025-02-14, 148000.00
MS-002, PROJ-SCH-02, Fundering gereed, 2025-02-28, 210000.00
MS-003, PROJ-WPT-03, Ruwbouw fase 1, 2025-03-07, 62000.00
```

### Customers (`data/sample/customers.csv`)

```csv
customer_id, name, type, avg_payment_lag_days, payment_reliability_score
CUST-001, Gemeente Amsterdam, housing_corporation, 55, 0.92
CUST-002, Schiphol Group, commercial, 35, 0.88
CUST-003, Particulier Renovatie BV, small_repair, 18, 0.74
```

Payment lag defaults by customer type (used when no history is available):

| Customer type | Default lag |
|---|---|
| Small repair | 18 days |
| Commercial | 35 days |
| Housing corporation | 55 days |
| Unknown / new | 30 days |

### Materials (`data/sample/materials.csv`)

```csv
material_id, project_id, milestone_id, description, cost_eur, order_week
MAT-001, PROJ-AMT-01, MS-001, Staalconstructie, 34040.00, W3
MAT-002, PROJ-SCH-02, MS-002, Betonpalen fundering, 48300.00, W5
```

Materials cost is calculated as `invoice_amount × 0.23` when not itemised, and is always ordered 2 weeks before the milestone date. Material costs are **not shifted** when milestones shift due to weather.

### Labour (`data/sample/labour.csv`)

```csv
labour_id, project_id, description, total_cost_eur, start_week, end_week
LAB-001, PROJ-AMT-01, Metselaars en timmerlieden, 320000.00, W1, W26
LAB-002, PROJ-SCH-02, Funderingsploeg, 510000.00, W2, W28
```

Weekly labour = `total_cost / project_duration_weeks`. Rain weeks (>15mm consensus) apply a 0.6 multiplier.

### Subcontractors (`data/sample/subcontractors.csv`)

```csv
subcontractor_id, project_id, name, completion_week, invoice_amount_eur, payment_lag_days
SUB-001, PROJ-AMT-01, Installatiebedrijf Röst, W10, 42000.00, 30
SUB-002, PROJ-WPT-03, Vloerleggers Benelux, W8, 21500.00, 14
```

---

## 4. External APIs

### Weather — Open-Meteo

**URL:** `https://api.open-meteo.com/v1/forecast`

**Auth:** None (free, no key required)

**Endpoint used:**

```
GET /v1/forecast
  ?latitude=52.37
  &longitude=4.90
  &daily=precipitation_sum,temperature_2m_min,wind_speed_10m_max
  &forecast_days=91
  &timezone=Europe/Amsterdam
```

**Fields extracted:**

| Field | Use |
|---|---|
| `precipitation_sum` (mm/day) | Rain threshold → lost day calculation |
| `temperature_2m_min` (°C) | Frost delay flag (< 2°C) |
| `wind_speed_10m_max` (km/h) | Reserved — not yet in delay model |

**Caching:** Results stored in `weather_cache` for the 13-week window. Re-fetched on each forecast run.

**Rate limits:** No hard limits on the free tier for forecast requests.

---

### Weather — OpenWeatherMap

**URL:** `https://api.openweathermap.org/data/3.0/onecall`

**Auth:** API key in `.env` as `OPENWEATHERMAP_API_KEY`

**Endpoint used:**

```
GET /data/3.0/onecall
  ?lat=52.37
  &lon=4.90
  &exclude=minutely,hourly,alerts
  &appid={key}
  &units=metric
```

**Fields extracted:**

| Field | Use |
|---|---|
| `daily.rain` (mm) | Rain consensus calculation |
| `daily.temp.min` (°C) | Frost flag |

**Free tier limits:** 1,000 API calls/day. One forecast run = 1 call. Cache prevents redundant calls within the same run.

**Consensus logic:**

```python
consensus_mm = (open_meteo_mm + openweather_mm) / 2

if abs(open_meteo_mm - openweather_mm) / consensus_mm > 0.30:
    flag = "LOW_WEATHER_CONFIDENCE"

lost_days = (
    5 if consensus_mm > 50 else
    2 if consensus_mm > 30 else
    1 if consensus_mm > 15 else
    0
)
```

---

### OpenAI API

**Model:** `gpt-4o` (GL classification, delay risk, copilot), `text-embedding-3-small` (embeddings)

**Auth:** API key in `.env` as `OPENAI_API_KEY`

**Used for four tasks — none touch financial numbers directly:**

#### GL Classification

```
POST /v1/chat/completions
System: You are a GL account classifier for a Dutch construction company...
User:   Account description: "Brandstofkosten wagenpark"
        Classify into one of: [Revenue, Materials, Payroll, Subcontractors,
        Equipment, Vehicles, Rent, Insurance, Taxes, Interest,
        Professional Services, Utilities, Office Expenses, Maintenance,
        Marketing, Other Operating Expenses, Other Income, Other]
        Return JSON: { category, confidence, reasoning, needs_review }
```

Confidence < 0.75 → `needs_review: true`. System never auto-approves below threshold.

#### Payment Lag Prediction

```
POST /v1/chat/completions
Input:  customer_name, customer_type, invoice_amount, season,
        historical_avg_lag_days, project_type
Output: { predicted_lag_days, confidence, reasoning }
```

Used only when `confidence > 0.80`. Otherwise falls back to `customer.avg_payment_lag_days`.

#### Delay Risk Scoring

```
POST /v1/chat/completions
Input:  project_type, region, contractor_name, weather_week_mm,
        historical_delay_rate_for_contractor
Output: { delay_probability, expected_delay_days, risk_score, reasoning }
```

Output is displayed as an advisory risk score only. Does not automatically shift milestone dates — weather-based shifting is deterministic.

#### Copilot (natural language Q&A)

```
POST /v1/chat/completions
System: You are a financial analyst assistant. Answer questions about the
        following 13-week cash flow forecast. Only reference numbers from
        the provided forecast data. Do not speculate or invent values.
Context: [Full forecast_weeks JSON for the selected company + week]
User:   "Why is Week 8 negative?"
```

The copilot reads `forecast_weeks.audit_json` and explains the derivation chain in plain language. It cannot modify or recalculate values.

#### Embeddings

```
POST /v1/embeddings
Model: text-embedding-3-small
Input: account_description (normalised)
```

Generated when a GL mapping is approved or corrected. Stored in `gl_mappings.embedding` (vector(1536)) for future similarity search via pgvector.

**Approximate token costs per forecast run:**

| Task | Tokens (est.) | Frequency |
|---|---|---|
| GL classification | ~200 per account | Once per upload |
| Payment lag | ~300 per invoice | Per forecast run |
| Delay risk | ~400 per project | Per forecast run |
| Copilot | ~1,500 per question | On demand |
| Embeddings | ~10 per account | On approval |

---

## 5. FX & Currency Handling

The system is EUR-native. All internal values are stored and displayed in euros.

**ECB exchange rates** are fetched weekly from the European Central Bank's public XML feed (no key required):

```
https://www.ecb.europa.eu/stats/eurofxref/eurofxref-daily.xml
```

If the `FX_API_KEY` is set in `.env`, a commercial FX rates provider is used instead (higher update frequency). Without a key, the system uses cached ECB rates updated daily.

**Multi-currency invoices** (e.g. USD subcontractors) are converted to EUR at the rate on the invoice date. The rate used is stored in `invoices.fx_rate_used` for audit purposes.

---

## 6. Data Quality & Validation Rules

The pipeline validates data at ingestion and flags issues rather than silently dropping records.

| Check | Rule | Action on failure |
|---|---|---|
| Missing account description | `account_description` must not be null or blank | Flag for manual review |
| Duplicate GL account | Same `account_number` + `company_id` already exists | Skip if approved; re-classify if unapproved |
| Invoice without project | `project_id` must reference an active project | Flag as `ORPHANED_INVOICE` |
| Milestone without customer | `customer_id` must exist in customers table | Use unknown lag default (30 days) |
| Negative invoice amount | Invoice amount < 0 | Flag as `ANOMALY_NEGATIVE_INVOICE` |
| Missing milestone date | `planned_date` is null | Exclude from schedule, flag |
| Labour cost exceeds project value | `total_labour_cost > sum(milestone_amounts)` | Flag as `ANOMALY_LABOUR_OVERRUN` |
| Weather provider disagreement | Provider delta > 30% | Flag week as `LOW_WEATHER_CONFIDENCE` |

Anomaly flags are displayed in the UI and stored in `forecast_weeks.anomaly_flags` (JSON array).

---

## 7. Adding a New Integration

To add a new accounting system connector:

1. Create `pipeline/connectors/your_system.py`
2. Implement the `BaseConnector` interface:

```python
from pipeline.connectors.base import BaseConnector

class YourSystemConnector(BaseConnector):

    def fetch_gl_accounts(self, company_id: str) -> list[dict]:
        """Return list of {account_number, account_description}"""
        ...

    def fetch_invoices(self, company_id: str, since: date) -> list[dict]:
        """Return list of invoice records matching internal schema"""
        ...

    def fetch_payments(self, company_id: str, since: date) -> list[dict]:
        """Return list of payment records matching internal schema"""
        ...
```

3. Register in `pipeline/connectors/__init__.py`
4. Add connector name to `SUPPORTED_SYSTEMS` in the UI settings panel
5. Write a sample export file to `data/sample/connectors/your_system_sample.xlsx`

For XLSX-only connectors (no API), extend `XLSXBaseConnector` which handles sheet parsing, column detection, and normalisation automatically — you only need to override `column_map()` to map system-specific column names to internal field names.
