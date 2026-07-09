# Monash Unit API

## Overview

Monash Unit API is a lightweight REST service that exposes the Monash University
handbook unit catalogue as a searchable, paginated JSON API. It loads a curated
dataset of **5,000+ units** into memory at startup and answers keyword searches
and code lookups with sub-millisecond latency.

It is intended for developers building student-facing tools (unit explorers,
planners, bookmarking apps) who need a simple, self-hostable source of Monash
unit data instead of scraping the handbook directly.

## Key Features

- Keyword search across unit code and title.
- Pagination with configurable page size.
- Direct unit lookup by unit code.
- Health-check endpoint for container orchestration and uptime probes.
- Single in-memory dataset load for fast, read-only responses.

## Tech Stack

- **Language:** Python 3.11
- **Framework:** Flask
- **Server:** Gunicorn (production), Flask dev server (local)
- **Packaging:** Docker

## Architecture Overview

The service is a single stateless Flask application. On boot it reads the
`units_2025_pretty.json` dataset once into memory; every request is served from
that in-memory list, so there is no database or external dependency at runtime.
Because state is immutable and process-local, the container scales horizontally
behind a load balancer with no coordination.

```
Client ──HTTP──> Flask app ──in-memory lookup──> JSON dataset (loaded at startup)
```

## Setup and Installation

### Prerequisites
- Python 3.11+ (or Docker)

### Local setup
```bash
git clone https://github.com/arsh-bansal/MonashUnitApi.git
cd MonashUnitApi
pip install -r requirements.txt
python app.py
```
The API will be available at `http://localhost:8000`.

### Docker
```bash
docker build -t monash-unit-api .
docker run -p 8000:8000 monash-unit-api
```

## Usage

| Method | Endpoint         | Description                                             |
|--------|------------------|---------------------------------------------------------|
| GET    | `/`              | API metadata and endpoint list                          |
| GET    | `/units`         | List/search units (`?q=`, `?page=`, `?limit=`)          |
| GET    | `/units/<code>`  | Get a single unit by code (e.g. `FIT5225`)              |
| GET    | `/health`        | Health check (`{"status": "ok"}`)                       |

Examples:
```bash
curl "http://localhost:8000/units?q=cloud&page=1&limit=10"
curl "http://localhost:8000/units/FIT5225"
curl "http://localhost:8000/health"
```

## Configuration

| Variable | Default   | Description                     |
|----------|-----------|---------------------------------|
| `HOST`   | `0.0.0.0` | Interface the server binds to   |
| `PORT`   | `8000`    | Port the server listens on      |

## Deployment

The included `Dockerfile` produces a self-contained image. For production, run
the app under Gunicorn (already in `requirements.txt`), for example:
```bash
gunicorn -w 4 -b 0.0.0.0:8000 app:app
```
The stateless design suits any container platform (Docker, Kubernetes, or a PaaS)
with multiple replicas behind a load balancer.

## Limitations and Assumptions

- The dataset is a point-in-time snapshot (`units_2025_pretty.json`); it does not
  update automatically from the live handbook.
- Search is a case-insensitive substring match on code and title only.
- The API is read-only; there are no write, auth, or rate-limit layers.

## Future Improvements

- Scheduled dataset refresh from the handbook source.
- Fuzzy/relevance-ranked search and filtering by faculty.
- Response caching headers and optional rate limiting.
