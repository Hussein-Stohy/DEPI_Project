# Job Market Tracker (NileTech Pulse) — DEPI Graduation Project

An end-to-end data engineering & analytics platform for **Egypt's tech job
market**: it scrapes job postings from five job boards, cleans and standardizes
them, and loads them into a PostgreSQL star schema modeled with dbt. The whole
pipeline is orchestrated with Airflow and served through a FastAPI backend to a
React + Vite frontend — the project is now **complete end-to-end**, from raw
scrape to public web app.

## Architecture

A medallion (Bronze → Silver → Gold) pipeline with a clean separation of
concerns:

| Layer | What | Where |
|-------|------|-------|
| **Producers** (ingestion) | 5 scrapers emitting one uniform contract | `producers/` → `data/raw/` (Bronze) |
| **Processing** (cleaning) | One source-agnostic cleaner + quality gate | `processing/` → `data/silver/` (Silver) |
| **Warehouse** (modeling) | PostgreSQL **star schema** loader (SCD2, idempotent) | `warehouse/` → PostgreSQL (Gold) |
| **dbt** (transformation) | staging → intermediate → marts (+ tests, snapshot) | `dbt/` → schemas `staging`/`intermediate`/`marts` |
| **Orchestration** | Apache Airflow — 3 DAGs (ingestion, processing & warehouse, dbt) | `airflow/` |
| **Backend API** | FastAPI serving analytics to the frontend | `api/` |
| **Frontend** | React + Vite web app | `frontend/` |

### Producers

Every producer emits the **same contract** (`producers/contract.py`) regardless
of how it scrapes, so the processing layer is fully source-agnostic:

| Producer | Method | Browser? | Notes |
|----------|--------|----------|-------|
| **Wuzzuf** | `requests` + BeautifulSoup | no | enriches each posting with its detail-page description (rich skills) |
| **Forasna** | `requests` + BeautifulSoup | no | Arabic IT listings; salary on the card |
| **Jobzella** | Selenium (headless) | yes | reads the Next.js `__NEXT_DATA__` JSON |
| **Bayt** | undetected-chromedriver (**headful**) | yes | behind Cloudflare — needs a display |
| **Indeed** | `python-jobspy` | (library) | may need a proxy on cloud IPs |

The runner skips browser-based producers gracefully when no Chrome/driver/display
is available, so the requests-based ones (and the whole downstream pipeline)
still run anywhere — including Docker.

```bash
make scrape                              # run all available producers
make scrape SOURCES="wuzzuf forasna"     # run a subset
python -m producers.runner --list        # list known producers
JOB_QUERY="data engineer" MAX_JOBS=200 make scrape   # scope the run
```

### Orchestration (Airflow)

The pipeline is scheduled and automated with **Apache Airflow**, split into three
focused DAGs so each stage can be triggered, retried, and monitored independently:

| DAG | Responsibility |
|-----|-----------------|
| **Ingestion DAG** | Runs the 5 producers, collecting raw job postings into the Bronze layer |
| **Processing & Warehouse DAG** | Cleans/standardizes data (Silver) and loads it into the PostgreSQL star schema (Gold) |
| **dbt DAG** | Runs dbt (staging → intermediate → marts) and executes tests/snapshots |

### Backend & Frontend

- **Backend API** — built with **FastAPI**, exposing the dbt marts (in-demand
  roles, salary intelligence, skill demand, company insights, geographic
  distribution, work-mode breakdown, hiring trends) as REST endpoints for the
  frontend.
- **Frontend** — built with **React + Vite**, consuming the backend API to
  present dashboards and search/filter views over Egypt's tech job market.

## Progress (against the project plan)

**Status: ✅ Project complete — all phases done.**

- ✅ **Phase 1** — Environment setup & project structure
- ✅ **Phase 2** — Data ingestion — all 5 producers (Wuzzuf, Forasna, Jobzella, Bayt, Indeed)
- ✅ **Phase 3** — Data cleaning & standardization (unified cleaner, quality gate)
- ✅ **Phase 4** — Data modeling & PostgreSQL star schema
- ✅ **Phase 5** — dbt transformation layer (staging → intermediate → marts, 49 tests, snapshot)
- ✅ **Phase 6** — Airflow orchestration & automation (3 DAGs: ingestion, processing & warehouse, dbt)
- ✅ **Phase 7** — Analytics & insights models (dbt marts)
- ✅ **Phase 8** — Testing, monitoring & documentation
- ✅ **Phase 9** — Backend API (FastAPI)
- ✅ **Phase 10** — Frontend web application (React + Vite)
- ✅ **Phase 11** — Deployment & infrastructure

## Setup (works on any machine)

Nothing is hardcoded — all configuration comes from environment variables /
`.env` (see `.env.example`). There are two supported paths.

### Option A — Docker (recommended, zero local dependencies)

Requires only Docker + Docker Compose. Spins up PostgreSQL, builds the
warehouse, and runs dbt against it:

```bash
cp .env.example .env          # adjust credentials if you like
docker compose up --build     # or: make up
```

`docker compose down -v` (or `make down`) tears it down, database included.
(Browser-based producers don't run inside Docker — there's no Chrome — but the
requests-based producers and the full processing → warehouse → dbt pipeline do.)

### Option B — Local install

Prerequisites: a running **PostgreSQL**, and [`uv`](https://docs.astral.sh/uv/)
(used to provision the Python 3.12 venv — dbt doesn't support 3.13/3.14 yet).
For the browser-based producers you also need **Chromium/Chrome + chromedriver**.

```bash
cp .env.example .env          # set PGHOST/PGPORT/PGUSER/PGPASSWORD/PGDATABASE
make setup                    # creates .venv (app + dbt + optional browser deps)
make bootstrap                # ensures the DB, loads the warehouse, runs dbt
```

`make setup` installs the browser-producer deps too; skip them with
`SKIP_BROWSER=1 make setup`. `make bootstrap` is idempotent (every step is
upsert / `IF NOT EXISTS`), so it's safe to re-run.

### Reproducibility notes

- Pinned dependencies: `requirements.txt` (app), `dbt-requirements.txt` (dbt),
  `requirements-browser.txt` (optional Selenium/jobspy stack).
- A single **Python 3.12** virtualenv (`.venv`) holds the app, dbt, and browser deps.
- The same `PG*` variables drive both the Python loader (`warehouse/config.py`)
  and dbt (`dbt/job_market_tracker/profiles.yml`) — one source of truth.

## Running individual steps

```bash
make scrape          # 1. collect raw data (Bronze)      producers/ -> data/raw/
make pipeline        # 2. clean & standardize (Silver)   processing/ -> data/silver/
make load            # 3. load star schema (Gold)        warehouse/ -> PostgreSQL
make dbt             # 4. transform with dbt (marts)
make test            # warehouse + pipeline test suites (isolated, non-destructive)
make stress          # high-volume stress test (isolated, non-destructive)
```

Run `make` with no target to list all tasks.

### Tests are isolated and safe

`make test` and `make stress` run against a **throwaway database**
(`<PGDATABASE>_test`, created and dropped per run) and a **temp workspace**, so
they never touch your real `data/` lake or warehouse. See `warehouse/isolation.py`.

## Analytics produced (Phase 5 marts, schema `marts`)

`mart_in_demand_roles`, `mart_salary_intelligence`, `mart_skill_demand`,
`mart_company_insights`, `mart_geographic_distribution`,
`mart_work_mode_breakdown`, `mart_hiring_trends` — see `dbt/job_market_tracker/README.md`.

## Real ingestion run (proof of concept)

A genuine end-to-end run on live data is captured under **`proof_of_concept/`**
(`POC_REPORT.md` + mart CSV exports + real raw samples). To reproduce a fresh run:

```bash
make scrape                                # live scrape -> Bronze
make pipeline                              # clean -> Silver
make load                                  # load -> Gold
make dbt                                   # transform -> marts
PYTHONPATH=. .venv/bin/python warehouse/poc_report.py   # write proof_of_concept/
```

## Documentation

- **`INSTRUCTIONS.md`** — full pipeline guide (producers + cleaning), in Arabic
- **`warehouse/README.md`** — Phase 4 star schema, loader, and end-to-end test
- **`dbt/job_market_tracker/README.md`** — Phase 5 dbt models, tests, and marts
- **`proof_of_concept/POC_REPORT.md`** — real end-to-end run results
