# Supply Chain Network Optimization & Forecasting Engine

> **Status:** 🚧 Under active development. See [Project Phase Tracker](#project-phase-tracker) below.

An end-to-end decision-support system that forecasts uncertain retailer demand,
converts that uncertainty into low/expected/high scenarios, solves a
cost-minimizing warehouse-to-retailer shipping optimization for each scenario,
and surfaces the results through a REST API and an interactive dashboard.

## Why this project is different

Most portfolio projects do *one* of: a forecasting notebook, a regression
exercise, a BI dashboard, or a toy linear program. This project chains all of
them into a single pipeline where each stage's output is the next stage's
input: **forecast uncertainty becomes optimization constraint uncertainty**,
and the **optimizer's output is what populates the dashboard**. The result is
a business-legible recommendation (a routing plan with a cost range) rather
than an isolated metric.

## Architecture

```
Synthetic/sourced demand data
        ↓
PostgreSQL / SQLite  (retailers, warehouses, demand_history, shipping_costs)
        ↓
Forecasting layer (Prophet)  →  point estimate + confidence interval
        ↓
Scenario conversion  →  low (p10) / expected (p50) / high (p90) demand
        ↓
Optimization layer (PuLP)  →  transportation LP solved once per scenario
        ↓
forecasts / optimization_results tables  (with run-lineage tracking)
        ↓
FastAPI  →  REST endpoints
        ↓
Streamlit + Plotly dashboard
```

Full architecture rationale, module boundaries, and design tradeoffs are in
[`docs/architecture.md`](docs/architecture.md).

## Tech stack

| Layer | Technology |
|---|---|
| Language | Python 3.12 |
| Forecasting | Prophet (SARIMA available as a swappable alternative) |
| Optimization | PuLP (CBC solver) |
| Database | PostgreSQL (production), SQLite (local dev/tests) |
| ORM / migrations | SQLAlchemy, Alembic |
| API | FastAPI + Pydantic |
| Dashboard | Streamlit + Plotly |
| Testing | pytest, pytest-cov |
| Code quality | black, ruff |
| Containerization | Docker, Docker Compose |

## Project structure

```
supply-chain-optimizer/
├── app/            # Orchestration/service layer: wires business logic to persistence
├── forecasting/     # Pure forecasting domain logic — no SQL, no HTTP
├── optimization/     # Pure optimization domain logic — no SQL, no HTTP
├── database/         # SQLAlchemy models, repositories, Alembic migrations
├── api/              # FastAPI routers + Pydantic schemas
├── dashboard/         # Streamlit app, calls the API over HTTP
├── data/              # Synthetic data generation + generated CSVs
├── config/            # Settings, logging, environment template
├── scripts/           # CLI entrypoints (data gen, DB init, pipeline runs)
├── tests/             # pytest suite, mirrors the module layout above
└── docs/              # Architecture, schema, API, deployment docs
```

## Quickstart

### Option A — Docker Compose (recommended)

```bash
cp config/.env.example .env
docker compose up --build
```

- API available at `http://localhost:8000` (docs at `http://localhost:8000/docs`)
- Dashboard available at `http://localhost:8501`

### Option B — Local development

```bash
python -m venv .venv
source .venv/bin/activate       # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp config/.env.example .env      # defaults to a local SQLite file
export PYTHONPATH=.              # required so scripts/ can import project packages
python scripts/init_db.py
python scripts/generate_data.py
python scripts/run_pipeline.py --model prophet   # use --model sarima if CmdStan isn't installed
uvicorn api.main:app --reload
streamlit run dashboard/app.py
```

### Running tests

```bash
pytest
```

## Project Phase Tracker

- [x] Phase 1 — Requirements analysis
- [x] Phase 2 — System design
- [x] Phase 3 — Project scaffolding
- [x] Phase 4 — Implementation (data generation, database, forecasting, optimization, app orchestration, API, dashboard)
- [x] Phase 5 — Integration (`scripts/init_db.py`, `generate_data.py`, `run_pipeline.py` — verified end to end against a real database)
- [x] Phase 6 — Validation (181/181 tests passing; real pipeline run, real API calls, real dashboard rendering against live data; zero broken imports)


## Known limitations

- SARIMA (the swappable alternative to Prophet) can fail to converge on series only just above its minimum history requirement — logged as a warning rather than silently returned. See `docs/architecture.md` for detail.
- No authentication layer (portfolio-scope decision, not an oversight — documented as a future improvement below).
- Optimization is single-period (one target week at a time), not a multi-period plan.

## Future improvements

- Auto-fallback from SARIMA to Prophet on convergence failure, rather than only logging.
- Per-series SARIMA order selection (e.g. `pmdarima.auto_arima`) as an opt-in mode.
- Full two-stage stochastic programming instead of three independent scenario LPs, if the business case justifies the added complexity.
- Authentication/authorization for multi-user deployments.
- CI pipeline running the test suite, `black --check`, and `ruff check` on every push.


## Documentation

- [`docs/architecture.md`](docs/architecture.md) — full architecture, module boundaries, design tradeoffs, known limitations
- [`docs/database_schema.md`](docs/database_schema.md) — schema reference and ER diagram
- [`docs/api_reference.md`](docs/api_reference.md) — every endpoint, request/response shapes, error handling
- [`docs/deployment.md`](docs/deployment.md) — Docker Compose, local dev, environment variables, migrations

## License

MIT License
