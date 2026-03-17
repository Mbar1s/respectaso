# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RespectASO is a privacy-first ASO (App Store Optimization) keyword research tool for iOS developers. It's a Django 5.1 full-stack web app using SQLite, Tailwind CSS (CDN), and vanilla JavaScript. It uses only the public iTunes Search API — no credentials or external services required.

## Commands

```bash
# Docker (primary development method)
docker compose up --build        # Build and run
docker compose up -d             # Run in background
docker compose down              # Stop

# Django management (inside container or local venv)
python manage.py runserver       # Dev server
python manage.py migrate         # Apply migrations
python manage.py makemigrations  # Create new migrations
python manage.py collectstatic   # Collect static files
python manage.py shell           # Interactive shell
```

There is no test suite, linter configuration, or CI pipeline.

## Architecture

### Django Project Layout

- **`core/`** — Django project settings, root URL config, WSGI/ASGI entry points, `context_processors.py` (injects VERSION into templates)
- **`aso/`** — Single Django app containing all ASO functionality

### Key Files in `aso/`

- **`services.py`** (~2000 lines) — Core business logic:
  - `ITunesSearchService` — iTunes Search API wrapper (2-second rate limit between calls)
  - `DifficultyCalculator` — 7-factor weighted scoring (rating volume 30%, dominant players 20%, rating quality 10%, market maturity 10%, publisher diversity 10%, app count 10%, content relevance 10%)
  - `PopularityEstimator` — 6-signal composite model with brand keyword detection and finance intent filtering
  - `DownloadEstimator` — 3-stage pipeline: popularity→daily searches, rank→tap-through rate, tap→install conversion
- **`views.py`** (~990 lines) — 19 view functions handling dashboard, search, opportunity finder, CRUD, CSV export, auto-refresh status, trend data
- **`scheduler.py`** — Background thread daemon for daily auto-refresh of all keyword+country pairs with progress tracking, 90-day data retention cleanup
- **`forms.py`** — `AppForm`, `KeywordSearchForm` (max 20 keywords, max 5 countries from 30 supported), `OpportunitySearchForm`
- **`models.py`** — Three models: `App`, `Keyword` (unique on keyword+app), `SearchResult` (stores snapshot with popularity/difficulty scores, competitors JSON, app rank, country)
- **`templatetags/aso_tags.py`** — Custom template filters for number formatting and difficulty label styling

### Request Flow

1. User submits keywords + countries via dashboard form
2. `search_view` parses input, calls `ITunesSearchService.search_apps()` per keyword+country
3. Results feed into `DifficultyCalculator`, `PopularityEstimator`, `DownloadEstimator`
4. Scores and competitor data stored as `SearchResult` linked to `Keyword` → `App`
5. Dashboard displays latest result per keyword+country pair with sorting, filtering, trends

### Templates

Templates live in `aso/templates/aso/`. The dashboard template (`dashboard.html`) is very large (~144KB) with inline JS for sorting, filtering, and interactive features. All templates use Tailwind CSS dark theme (`bg-slate-900`, `bg-[#1e293b]`).

### Deployment

Single Docker image: Python 3.12-slim → Gunicorn (1 worker, 2 threads, port 8080). `entrypoint.sh` auto-generates SECRET_KEY on first run and runs migrations. SQLite database persisted via Docker volume at `/app/data/`.

## Code Style

- Python: PEP 8, Django conventions
- Templates: Tailwind CSS utilities, dark theme design system
- JavaScript: Vanilla JS only (no frameworks), `const`/`let` (no `var`), `async/await`
- Version is defined in `core/settings.py` as `VERSION`
