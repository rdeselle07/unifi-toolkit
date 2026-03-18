# Codebase Structure

**Analysis Date:** 2026-03-18

## Directory Layout

```
unifi-toolkit/
├── run.py                  # Application entry point (env setup, migrations, uvicorn)
├── pyproject.toml          # Python project metadata and dependencies
├── pytest.ini              # Test configuration
├── alembic.ini             # Alembic migration configuration
├── Dockerfile              # Docker build definition
├── docker-compose.yml      # Docker Compose deployment config
├── setup.sh                # First-run setup script
├── upgrade.sh              # Upgrade helper script
├── reset_password.sh       # Auth password reset script
├── app/                    # Main dashboard application
│   ├── __init__.py         # App version (__version__)
│   ├── main.py             # FastAPI app, lifespan, middleware, sub-app mounting
│   ├── routers/            # Dashboard API routers
│   │   ├── auth.py         # Login/logout, session management, AuthMiddleware
│   │   └── config.py       # UniFi config CRUD, connection test, gateway check
│   ├── templates/          # Jinja2 templates
│   │   ├── dashboard.html  # Main dashboard UI
│   │   └── login.html      # Login page (production mode)
│   └── static/             # Dashboard static assets
│       ├── css/
│       │   └── dashboard.css
│       └── images/         # Logos and favicon
├── shared/                 # Shared infrastructure (used by all tools)
│   ├── __init__.py
│   ├── config.py           # ToolkitSettings (pydantic-settings, singleton)
│   ├── database.py         # Async SQLAlchemy engine/session (singleton)
│   ├── unifi_client.py     # UniFi OS API client (1800+ lines, core data layer)
│   ├── unifi_session.py    # Shared singleton UniFi session for schedulers
│   ├── cache.py            # In-memory TTL cache
│   ├── crypto.py           # Fernet encryption for credentials
│   ├── webhooks.py         # Webhook delivery (Slack, Discord, n8n)
│   ├── websocket_manager.py # WebSocket connection manager (singleton)
│   ├── url_validator.py    # URL validation utility
│   └── models/             # Shared SQLAlchemy models
│       ├── __init__.py
│       ├── base.py         # Declarative Base
│       └── unifi_config.py # UniFiConfig model (single-row table)
├── tools/                  # Tool sub-applications
│   ├── wifi_stalker/       # Client device tracking tool
│   │   ├── __init__.py     # Tool version
│   │   ├── main.py         # FastAPI sub-app factory
│   │   ├── scheduler.py    # Background device polling (APScheduler)
│   │   ├── models.py       # Pydantic request/response schemas
│   │   ├── database.py     # SQLAlchemy models (TrackedDevice, ConnectionHistory, etc.)
│   │   ├── routers/
│   │   │   ├── devices.py  # Device CRUD, detail, history, analytics
│   │   │   ├── config.py   # Tool-specific UniFi config endpoints
│   │   │   └── webhooks.py # Webhook CRUD and testing
│   │   ├── templates/
│   │   │   └── index.html  # WiFi Stalker dashboard
│   │   └── static/
│   │       ├── css/styles.css
│   │       └── js/app.js
│   ├── threat_watch/       # IDS/IPS monitoring tool
│   │   ├── __init__.py     # Tool version
│   │   ├── main.py         # FastAPI sub-app factory
│   │   ├── scheduler.py    # Background IPS event polling + purge
│   │   ├── models.py       # Pydantic schemas (events, stats, filters, webhooks, ignore rules)
│   │   ├── database.py     # SQLAlchemy models (ThreatEvent, ThreatWebhookConfig, ThreatIgnoreRule)
│   │   ├── routers/
│   │   │   ├── events.py   # Event listing, detail, stats, timeline
│   │   │   ├── config.py   # Tool-specific config endpoints
│   │   │   ├── webhooks.py # Webhook CRUD and testing
│   │   │   └── ignore_rules.py # Ignore rule CRUD
│   │   ├── templates/
│   │   │   └── index.html  # Threat Watch dashboard
│   │   └── static/
│   │       ├── css/styles.css
│   │       └── js/app.js
│   └── network_pulse/      # Real-time network health dashboard
│       ├── __init__.py     # Tool version
│       ├── main.py         # FastAPI sub-app factory (includes WebSocket endpoint)
│       ├── scheduler.py    # Background network polling + WebSocket broadcast
│       ├── models.py       # Pydantic schemas (gateway, WAN, APs, clients, dashboard data)
│       ├── routers/
│       │   └── stats.py    # Dashboard data and AP detail endpoints
│       ├── templates/
│       │   ├── index.html  # Network Pulse dashboard (Alpine.js)
│       │   └── ap_detail.html # AP detail page
│       └── static/
│           ├── css/styles.css
│           └── js/
│               ├── app.js       # Main dashboard JS (Alpine.js)
│               └── ap_detail.js # AP detail page JS
├── alembic/                # Database migrations
│   └── versions/           # 10 migration files (chronological, prefixed with date)
├── tests/                  # Test suite
├── data/                   # SQLite database storage (runtime, not committed)
├── docs/                   # User-facing documentation
│   ├── INSTALLATION.md
│   ├── QUICKSTART.md
│   ├── SYNOLOGY.md
│   └── UNRAID.md
├── logo/                   # Brand assets
├── unraid/                 # Unraid-specific deployment files
└── .github/
    └── workflows/
        ├── docker-publish.yml  # Docker image build and push
        └── stale-issues.yml    # Auto-close stale issues
```

## Directory Purposes

**`app/`:**
- Purpose: Main dashboard application and shared API endpoints
- Contains: FastAPI app with lifespan, auth system, UniFi config management, system status API
- Key files: `main.py` (app entry, sub-app mounting), `routers/auth.py` (session auth + CSRF), `routers/config.py` (UniFi config CRUD)

**`shared/`:**
- Purpose: Infrastructure code shared across all tools
- Contains: Database layer, UniFi API client, caching, encryption, webhooks, WebSocket management, configuration
- Key files: `unifi_client.py` (largest file, all UniFi API interactions), `unifi_session.py` (shared session singleton), `database.py` (async SQLAlchemy)

**`tools/wifi_stalker/`:**
- Purpose: Track specific WiFi/wired client devices across UniFi network
- Contains: Device tracking, connection history, roaming detection, analytics (dwell time, presence patterns), webhooks
- Key files: `scheduler.py` (background refresh), `database.py` (4 ORM models), `routers/devices.py` (device CRUD + analytics)

**`tools/threat_watch/`:**
- Purpose: Monitor IDS/IPS events from UniFi gateway
- Contains: Threat event collection, statistics, filtering, ignore rules, webhooks
- Key files: `scheduler.py` (event polling + purge), `database.py` (3 ORM models), `routers/events.py` (event API)

**`tools/network_pulse/`:**
- Purpose: Real-time network health monitoring dashboard
- Contains: Gateway stats, WAN health, AP status, client info, throughput charts
- Key files: `scheduler.py` (polling + WebSocket broadcast), `models.py` (DashboardData composite model), `static/js/app.js` (Alpine.js frontend)

**`alembic/`:**
- Purpose: Database schema migrations
- Contains: 10 migration files covering schema additions across all tools
- Key pattern: Migrations are date-prefixed (e.g., `20251202_0146_...`). New columns added in migrations must also be added to `_repair_schema()` in `run.py`

**`data/`:**
- Purpose: Runtime data directory for SQLite database
- Generated: Yes (created at startup)
- Committed: No (`.gitignore`d, volume-mounted in Docker)

## Key File Locations

**Entry Points:**
- `run.py`: Application bootstrap (env vars, migrations, uvicorn launch)
- `app/main.py`: FastAPI application definition and configuration

**Configuration:**
- `shared/config.py`: `ToolkitSettings` class (all env var definitions)
- `alembic.ini`: Alembic migration configuration
- `pyproject.toml`: Python project metadata and dependency list
- `Dockerfile`: Container build definition
- `docker-compose.yml`: Docker deployment configuration

**Core Logic:**
- `shared/unifi_client.py`: All UniFi API communication (device model maps, API calls, data normalization)
- `shared/unifi_session.py`: Shared persistent UniFi connection
- `tools/wifi_stalker/scheduler.py`: Device tracking logic (connect/disconnect/roam detection)
- `tools/threat_watch/scheduler.py`: IPS event collection, parsing, and purge logic
- `tools/network_pulse/scheduler.py`: Network health data assembly and WebSocket broadcast

**Database:**
- `shared/models/unifi_config.py`: UniFi controller config (shared across tools)
- `tools/wifi_stalker/database.py`: `TrackedDevice`, `ConnectionHistory`, `WebhookConfig`, `HourlyPresence`
- `tools/threat_watch/database.py`: `ThreatEvent`, `ThreatWebhookConfig`, `ThreatIgnoreRule`
- `shared/database.py`: Engine/session factory
- `run.py` (`_repair_schema()`): Schema repair safety net for missed migrations

**Frontend:**
- `app/templates/dashboard.html`: Main dashboard
- `tools/wifi_stalker/templates/index.html`: WiFi Stalker UI
- `tools/threat_watch/templates/index.html`: Threat Watch UI
- `tools/network_pulse/templates/index.html`: Network Pulse UI (Alpine.js)
- `tools/network_pulse/templates/ap_detail.html`: AP detail page

**Testing:**
- `tests/`: Test suite directory

## Naming Conventions

**Files:**
- Python modules: `snake_case.py` (e.g., `unifi_client.py`, `websocket_manager.py`)
- Templates: `snake_case.html` (e.g., `ap_detail.html`, `dashboard.html`)
- Static assets: `snake_case.js`, `snake_case.css` (e.g., `ap_detail.js`, `styles.css`)
- Migrations: `YYYYMMDD_HHMM_hash_description.py`

**Directories:**
- Python packages: `snake_case` (e.g., `wifi_stalker`, `threat_watch`, `network_pulse`)
- Asset directories: lowercase singular (e.g., `static`, `templates`, `routers`)

**Database Tables:**
- Tool-prefixed: `stalker_tracked_devices`, `stalker_connection_history`, `stalker_webhook_config`, `stalker_hourly_presence`
- Tool-prefixed: `threats_events`, `threats_webhook_config`, `threats_ignore_rules`
- Shared: `unifi_config`

**Python Classes:**
- SQLAlchemy models: `PascalCase` (e.g., `TrackedDevice`, `ThreatEvent`, `UniFiConfig`)
- Pydantic models: `PascalCase` with suffix (e.g., `DeviceResponse`, `ThreatEventCreate`, `SystemStatus`)
- Utility classes: `PascalCase` (e.g., `WebSocketManager`, `Database`, `UniFiClient`)

## Where to Add New Code

**New Tool:**
1. Create `tools/new_tool/` directory with standard structure:
   - `__init__.py` (version), `main.py` (app factory), `scheduler.py` (if polling needed)
   - `models.py` (Pydantic schemas), `database.py` (SQLAlchemy models if persistent)
   - `routers/` (API endpoints), `templates/` (Jinja2 HTML), `static/` (CSS/JS)
2. Mount in `app/main.py`: `app.mount("/newtool", create_newtool_app())`
3. Start/stop scheduler in `app/main.py` lifespan handler
4. Add Alembic migration for any new tables
5. Add new columns to `_repair_schema()` in `run.py`

**New API Endpoint (existing tool):**
- Add route to appropriate router in `tools/{tool}/routers/`
- Add Pydantic request/response models in `tools/{tool}/models.py`
- Follow existing pattern: `@router.get("/api/endpoint")` with SQLAlchemy session dependency

**New API Endpoint (dashboard level):**
- Add to `app/main.py` directly (for simple endpoints like `/api/debug-info`)
- Or create new router in `app/routers/` and include via `app.include_router()`

**New Database Table:**
- Define SQLAlchemy model in `tools/{tool}/database.py` (or `shared/models/` if shared)
- Inherit from `shared.models.base.Base`
- Create Alembic migration: `alembic revision --autogenerate -m "description"`
- Add all new columns to `_repair_schema()` in `run.py`

**New UniFi API Call:**
- Add method to `shared/unifi_client.py` following existing patterns
- Use `self._api_request()` or direct aiohttp calls with `/proxy/network/api/` prefix
- Add caching in `shared/cache.py` if result is reused frequently

**Shared Utilities:**
- Add to `shared/` as a new module
- Follow singleton pattern with `get_*()` accessor if stateful

**New Frontend Feature:**
- Template: Add/modify in `tools/{tool}/templates/`
- JavaScript: Add/modify in `tools/{tool}/static/js/`
- CSS: Add/modify in `tools/{tool}/static/css/`
- Network Pulse uses Alpine.js; WiFi Stalker and Threat Watch use vanilla JS

## Special Directories

**`data/`:**
- Purpose: SQLite database file (`unifi_toolkit.db`) and Alembic version tracking
- Generated: Yes (created at startup by `run.py`)
- Committed: No (in `.gitignore`, volume-mounted in Docker)

**`alembic/versions/`:**
- Purpose: Database migration scripts
- Generated: Yes (via `alembic revision --autogenerate`)
- Committed: Yes

**`unraid/`:**
- Purpose: Unraid-specific deployment templates
- Generated: No
- Committed: Yes

**`.github/workflows/`:**
- Purpose: CI/CD workflows (Docker image publishing, stale issue management)
- Generated: No
- Committed: Yes

## Version Management

Version is maintained in THREE files that must stay in sync:
- `pyproject.toml` → `version = "X.Y.Z"`
- `app/__init__.py` → `__version__ = "X.Y.Z"`
- `app/main.py` → `version="X.Y.Z"` (FastAPI constructor)

Each tool also maintains its own version in `tools/{tool}/__init__.py`.

## URL Routing Map

| URL Pattern | Handler | Purpose |
|---|---|---|
| `/` | `app/main.py` | Main dashboard |
| `/login`, `/logout` | `app/routers/auth.py` | Authentication |
| `/api/system-status` | `app/main.py` | System status JSON |
| `/api/debug-info` | `app/main.py` | Debug info for issue reporting |
| `/api/update-check` | `app/main.py` | GitHub release check |
| `/api/config/*` | `app/routers/config.py` | UniFi config CRUD |
| `/health` | `app/main.py` | Health check |
| `/ws` | `app/main.py` | WebSocket (main) |
| `/stalker/*` | `tools/wifi_stalker/` | WiFi Stalker sub-app |
| `/threats/*` | `tools/threat_watch/` | Threat Watch sub-app |
| `/pulse/*` | `tools/network_pulse/` | Network Pulse sub-app |
| `/pulse/ws` | `tools/network_pulse/main.py` | WebSocket (Network Pulse) |

---

*Structure analysis: 2026-03-18*
