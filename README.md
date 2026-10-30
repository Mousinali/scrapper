# Sales Lead Scraper

A locally hosted, Python + Streamlit web app that discovers businesses and their
public contact information from live web search and company websites, stores
results in PostgreSQL, and exports them to CSV/Excel. No paid APIs or paid
scraping platforms are required for the core workflow.

It behaves like a virtual researcher: you pick an industry and a location, it
searches the live web, visits permitted public pages, extracts business and
contact details, validates and deduplicates them, and saves results
incrementally so you can watch progress and stop/resume safely.

## Architecture

```
app/
  main.py, pages/        Streamlit dashboard (12 pages)
  db/                     SQLAlchemy models + Alembic migrations
  scraper/                Search adapter, Playwright browser, extraction,
                          validation, dedupe, decision-maker discovery,
                          job orchestration (app/scraper/discovery.py),
                          upload-driven enrichment (app/scraper/enrich.py)
  jobs/control.py         Start/cancel/pause/resume a scraping job
  export/csv_excel.py     CSV/Excel export with formula-injection protection
docker/searxng/           Local SearXNG config (free web search, no paid API)
maps_scraper/             Standalone CustomTkinter + Playwright Google Maps
                          scraper - a separate desktop app, own storage, not
                          part of the Streamlit app's database (see below)
tests/                    pytest suite (pure-logic + in-memory-DB tests)
```

Free web search comes from a **locally hosted SearXNG** instance (Docker),
not a paid search API. Business pages are fetched with a headless Chromium
browser via Playwright — a real browser, not a static HTTP fetch, so it
renders JS-driven pages like a human visitor would.

## Prerequisites

- Python 3.11+ (the extraction/validation code uses modern type-hint syntax)
- PostgreSQL 14+
- Docker (optional, but the default search source — SearXNG — runs in it)
- ~500MB disk for Chromium (installed automatically by Playwright)
- Tk support for Python (needed only for the standalone **Live Map Scraper**
  desktop app). On macOS with Homebrew Python, this isn't included by
  default — see Troubleshooting if `import tkinter` fails with
  `ModuleNotFoundError: No module named '_tkinter'`.

## Setup

### macOS

```bash
./scripts/install_macos.sh
```

This installs Python 3.12 and PostgreSQL via Homebrew if missing, creates a
virtualenv, installs Python dependencies, installs Playwright's Chromium,
creates the `lead_scrap` database, runs migrations, and copies `.env.example`
to `.env` if you don't already have one. If Docker is installed it also
starts a local SearXNG container; if not, it tells you how to install Docker
or configure another SearXNG instance via `SEARXNG_URL` in `.env`.

### Windows 10/11

Run in PowerShell (as your normal user, not elevated unless a step asks):

```powershell
.\scripts\install_windows.ps1
```

This uses `winget` to install Python 3.12 if missing, creates a virtualenv,
installs Python dependencies, and installs Playwright's Chromium. **PostgreSQL
and Docker are not installed automatically on Windows** — the script checks
for them and prints exact next steps if either is missing (installer links
for PostgreSQL, and `winget install Docker.DockerDesktop` for Docker).

### Manual setup (either OS, if the scripts don't fit your environment)

```bash
python3.11 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
playwright install chromium
cp .env.example .env              # then edit DATABASE_URL if needed
createdb lead_scrap               # or: psql -c "CREATE DATABASE lead_scrap;"
alembic upgrade head
```

To run the free local search source (recommended):

```bash
docker run -d --name searxng -p 127.0.0.1:8888:8080 \
  -v "$(pwd)/docker/searxng:/etc/searxng" \
  -e INSTANCE_NAME=lead-scrap-search searxng/searxng:latest
```

`docker/searxng/settings.yml` in this repo already enables the JSON API and
a few extra free engines (Bing, Yahoo, Yep) so discovery keeps working if
DuckDuckGo/Brave/Google CSE get rate-limited under heavy use — this will
happen with free search sources; it's a known limitation, not a bug (see
Limitations below).

## Running the app

```bash
source venv/bin/activate          # Windows: venv\Scripts\activate
streamlit run app/main.py
```

Opens at `http://127.0.0.1:8501`, bound to localhost only by default
(`.streamlit/config.toml`). Start a job from **New Scraping Job**, watch it
run from **Active Jobs**, review leads in **All Leads** / **Lead Details**,
and export from **Export Centre**.

### Live Map Scraper (standalone desktop app)

A separate tool for pulling business leads straight from Google Maps -
deliberately decoupled from the app's database (its own dedupe/daily-cap
state lives in `maps_scraper/data/state.json`, its own CSV/Excel export).

- Sidebar → **Live Map Scraper** → **Launch Maps Scraper**. This spawns the
  CustomTkinter desktop window as its own OS process (Streamlit itself
  can't host native desktop widgets), which opens a **visible** Chrome
  window.
- Enter a query (e.g. "hospitals near me"), set max leads for this run and
  a delay range, click **Start**. It scrolls the results feed and reads
  name/phone/address/website/category straight off each card.
- Capped at **500 unique leads/day** across all runs, tracked by phone (or
  name+address when no phone is listed) in that same state file.
- If Google serves a CAPTCHA, the scraper pauses itself automatically (or
  click **Pause** any time) - solve it by hand in the visible browser, then
  click **Resume** to continue exactly where it left off.
- Export via the **Export CSV** / **Export Excel** buttons in that window.
- Can also be run directly without Streamlit: `python -m maps_scraper.gui`.

If the window opens and closes immediately, the Streamlit launcher page
captures the crash to `maps_scraper/data/launcher.log` and prints it on the
page - check there first (most often a missing dependency or, on macOS,
missing Tk support; see Troubleshooting).

### Upload & Enrich

Takes a CSV/Excel with a **Website** column - the Maps Scraper's own export
works directly - and, unlike the Maps Scraper, writes results into this
app's **main lead database**, so they show up in All Leads/Companies/
Contact Persons/Export Centre like any other job.

For each website it reuses the same pipeline as New Scraping Job: visits
the site, extracts social links/emails/phones, checks on-site About/Team
pages for named contacts, and searches public LinkedIn listings for the
company name. Duplicate detection runs the same as everywhere else in the
app (domain match, then phone+name, then name+location).

Finding LinkedIn/contact-person profiles needs the SearXNG search service
running (see Setup) - without it, social links/emails/phones still get
found, but the LinkedIn step returns nothing.

## Running tests

```bash
source venv/bin/activate
pytest tests/ -v
```

The suite is pure-logic (query generation, extraction heuristics, validation,
dedupe rules, export/formula-injection escaping) plus a few tests against an
in-memory SQLite database — it does not require a live PostgreSQL or SearXNG
instance to run.

## Troubleshooting

**`psycopg2.OperationalError: connection refused`**
PostgreSQL isn't running. macOS: `brew services start postgresql@17`.
Windows: start the "postgresql-x64-\*" service from Services, or via
`pg_ctl start` if you installed it manually.

**Dashboard shows 0 search results / jobs complete with 0 leads**
Check **Settings and Logs** for a `warning` event like "Search engines
currently unavailable: brave (...), duckduckgo (...)". Free search engines
rate-limit or CAPTCHA aggressive querying — this is expected background
behavior for free sources (prod.md explicitly anticipates this), not an app
bug. Wait a while, or add more engines in `docker/searxng/settings.yml` and
`docker restart searxng`.

**`playwright._impl._api_types.Error: Executable doesn't exist`**
Chromium wasn't installed for Playwright. Run `playwright install chromium`
inside the venv.

**Port 8501 (Streamlit) or 8888 (SearXNG) already in use**
Another instance is already running — stop it, or change the port via
`streamlit run app/main.py --server.port 8502` / editing the `docker run -p`
mapping.

**`ModuleNotFoundError: No module named '_tkinter'`** (Live Map Scraper only)
Homebrew's Python on macOS doesn't bundle Tk by default. Fix:
`brew install python-tk@3.12` (match your Python's minor version), then
re-activate the venv - no need to recreate it.

**Live Map Scraper window opens, then closes immediately**
Check `maps_scraper/data/launcher.log`, or whatever the Streamlit page
printed when this happens - it captures the subprocess's stderr. Common
causes: `customtkinter` not installed (`pip install -r requirements.txt`),
or Playwright's Chromium missing (`playwright install chromium`).

**Alembic complains about an existing table / out-of-sync migration state**
Check `alembic current` vs `alembic history`. If this is a throwaway dev DB,
the simplest fix is dropping and recreating it (`dropdb lead_scrap && createdb
lead_scrap && alembic upgrade head`) — never do this against real data.

## Limitations and costs

- **Free search is not guaranteed to be unlimited.** SearXNG's underlying
  engines (DuckDuckGo, Brave, Google CSE, Bing, Yahoo, Yep) rate-limit or
  CAPTCHA-block aggressive use. The app logs a warning event when this
  happens instead of silently returning nothing — check Settings and Logs.
- **Decision-maker discovery is heuristic.** On-site name/title extraction is
  a regex pattern over page text (see the `shortcut:` comment in
  `app/scraper/people.py`) and will miss unusually structured team pages.
  LinkedIn-sourced contacts come only from public search-result titles/
  snippets — the app never logs into LinkedIn or opens a profile page, since
  that would require bypassing authentication the spec forbids.
  `verification_status` reflects this: `source_confirmed` for on-site finds,
  `unverified` for search-result-only finds.
  - **Official-site vs. directory vs. social classification is a maintained
  domain list**, not a general classifier (see `REGISTRY_DIRECTORY_DOMAINS` /
  `SOCIAL_PLATFORM_DOMAINS` / `NON_BUSINESS_DOMAINS` in `app/scraper/extract.py`).
  Known directories (JustDial, Practo, ZaubaCorp, etc.), social platforms, and
  non-business sites (Wikipedia, job boards, etc.) are classified correctly,
  but an unfamiliar aggregator or spam/SEO site can still slip through
  occasionally — free search surfaces a long tail of these for popular
  commercial search terms. Each one found in testing has been added; expect
  new ones on different industries/locations (see `todo.md` for the detailed
  log of what's been fixed this way).
- **No mandatory paid services.** Everything above runs on open-source
  software and free (rate-limited) search engines. An optional AI agent for
  ambiguous page navigation is explicitly out of scope for this build — all
  extraction here is deterministic (Playwright + regex/JSON-LD parsing).
- **500-lead absolute cap, enforced in-process.** The cap and dedupe logic
  use a Python-level lock, which is correct for the single-process concurrency
  this app uses today. It would need a database-level lock (e.g. `SELECT ...
  FOR UPDATE`) if the app were ever split across multiple processes/machines.
- **Live Map Scraper reads Google's own (undocumented) card markup**, not a
  published API - category/address/phone parsing is a text-layout heuristic
  (see the `shortcut:` comment in `maps_scraper/scraper.py`). Verified
  working live against google.com/maps as of 2026-10-26; if Google changes
  that layout, update the parsing there, not the main app.

## Known acceptance criteria status (prod.md §17)

All criteria have been live-verified against real running jobs during
development — see `todo.md` for the detailed, per-phase verification log
(what was tested, what broke, what got fixed) kept throughout the build.
# scrapper
