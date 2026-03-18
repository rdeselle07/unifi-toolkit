# Technology Stack

**Analysis Date:** 2026-03-18

## Languages

**Primary:**
- Python 3.9+ (requires `>=3.9,<3.13`) - All backend logic, API, scheduling, database

**Secondary:**
- JavaScript (vanilla) - Main dashboard frontend (`app/static/`)
- JavaScript (Alpine.js) - Network Pulse frontend (`tools/network_pulse/`)
- HTML/CSS - Jinja2 templates (`app/templates/`, `tools/*/templates/`)

## Runtime

**Environment:**
- Python 3.12 (Docker production image uses `python:3.12-slim`)
- Compatible with 3.9, 3.10, 3.11, 3.12 (not 3.13)

**Package Manager:**
- pip (no poetry, no pipenv)
- Lockfile: Not present - `requirements.txt` uses `>=` minimum version pins
- No `setup.py` or `setup.cfg` - uses `pyproject.toml` for project metadata only (not build)

## Frameworks

**Core:**
- FastAPI `>=0.115.6` - Web framework, REST API, WebSocket support (`app/main.py`)
- Starlette `>=0.47.2` - ASGI foundation (middleware, static files, templating)
- Uvicorn `>=0.32.0` (with `[standard]` extras) - ASGI server, started from `run.py`

**Database:**
- SQLAlchemy `>=2.0.36` - Async ORM with `AsyncSession` (`shared/database.py`)
- aiosqlite `>=0.20.0` - Async SQLite driver
- Alembic `>=1.13.0` - Database migrations (`alembic/` directory)
- greenlet `>=3.0.0` - Required by SQLAlchemy async

**Templating:**
- Jinja2 `>=3.1.6` - Server-side HTML rendering

**Scheduling:**
- APScheduler `>=3.10.4` - Background task scheduling (`AsyncIOScheduler`)
  - Used by: WiFi Stalker, Threat Watch, Network Pulse (each has own scheduler)

**HTTP Client:**
- aiohttp `>=3.10.0` - Async HTTP client for UniFi API communication and webhook delivery

**Testing:**
- Not detected in `requirements.txt` - no test framework dependency declared

**Build/Dev:**
- Black (configured in `pyproject.toml`, line-length 100)
- Ruff (configured in `pyproject.toml`, line-length 100, target py39)
- mypy (configured in `pyproject.toml`, `disallow_untyped_defs = false`)
- Docker multi-stage build (`Dockerfile`)

## Key Dependencies

**Critical:**
- `aiohttp >=3.10.0` - Core UniFi API client and webhook delivery (`shared/unifi_client.py`, `shared/webhooks.py`)
- `sqlalchemy >=2.0.36` - All data persistence (`shared/database.py`, tool models)
- `cryptography >=44.0.1` - Fernet encryption for stored credentials (`shared/crypto.py`)
- `pydantic >=2.10.3` + `pydantic-settings >=2.6.1` - Settings management from env vars (`shared/config.py`)

**Infrastructure:**
- `bcrypt >=4.0.0` - Password hashing for production auth (`app/routers/auth.py`)
- `python-dotenv >=1.0.1` - `.env` file loading (`run.py`)
- `itsdangerous >=2.1.0` - Signing/serialization utilities
- `python-multipart >=0.0.6` - Form data parsing (login form)

**Security Pins (CVE mitigation):**
- `urllib3 >=2.6.0` - GHSA-gm62-xv2j-4w53, GHSA-2xpw-w6gg-jr37
- `filelock >=3.20.1` - GHSA-w853-jp5j-5j7f

## Configuration

**Environment:**
- Configuration via environment variables, loaded by pydantic-settings (`shared/config.py`)
- `.env` file support via python-dotenv (loaded in `run.py` before app start)
- `.env.example` provides template with all available settings
- Docker Swarm `_FILE` env var support for secrets (`run.py:_resolve_file_env_vars()`)
- Supported `_FILE` vars: `ENCRYPTION_KEY`, `AUTH_USERNAME`, `AUTH_PASSWORD_HASH`, `DATABASE_URL`, `UNIFI_PASSWORD`, `UNIFI_API_KEY`

**Key Settings (via `ToolkitSettings` in `shared/config.py`):**
- `ENCRYPTION_KEY` (required) - Fernet key for credential encryption
- `DEPLOYMENT_TYPE` - `local` (no auth) or `production` (auth + HTTPS)
- `DATABASE_URL` - Default: `sqlite+aiosqlite:///./data/unifi_toolkit.db`
- `LOG_LEVEL` - Default: `INFO`
- `APP_PORT` - Default: `8000`
- `AUTH_USERNAME` / `AUTH_PASSWORD_HASH` - Production mode credentials
- `DOMAIN` - For Caddy HTTPS (production mode)
- `STALKER_REFRESH_INTERVAL` - Device poll interval, default 60s

**Build:**
- `Dockerfile` - Multi-stage build (builder + runtime), `python:3.12-slim` base
- `docker-compose.yml` - Service definition with optional Caddy reverse proxy (production profile)
- `alembic.ini` - Migration configuration
- `pyproject.toml` - Project metadata, tool configs (Black, Ruff, mypy)

## Platform Requirements

**Development:**
- Python 3.9-3.12
- pip for dependency installation
- SQLite (included with Python)
- No Node.js/npm required (vanilla JS frontend)

**Production:**
- Docker (primary deployment method)
- Targets: Synology NAS, Unraid, TrueNAS, any Docker host
- Multi-arch builds: `linux/amd64`, `linux/arm64`
- Non-root container user (UID 1000)
- Volume mount for `/app/data` (SQLite database persistence)
- Health check: `curl -f http://localhost:8000/health`
- Optional: Caddy reverse proxy for HTTPS (via docker-compose `production` profile)

**Container Registries:**
- GitHub Container Registry: `ghcr.io/crosstalk-solutions/unifi-toolkit`
- Docker Hub: `crosstalksolutions/unifi-toolkit`

---

*Stack analysis: 2026-03-18*
