# Architecture

**Analysis Date:** 2026-03-18

## Pattern Overview

**Overall:** Modular monolith with sub-application mounting

**Key Characteristics:**
- Single FastAPI process hosts a main dashboard plus three tool sub-applications mounted at distinct URL prefixes
- Each tool is a self-contained FastAPI app with its own routers, templates, static assets, models, and background scheduler
- Shared infrastructure (database, UniFi API client, cache, WebSocket manager, config) lives in a `shared/` package used by all tools
- Background schedulers poll the UniFi controller on intervals; frontends fetch data via REST APIs and WebSockets
- SQLite database with async SQLAlchemy ORM; single DB file shared across all tools

## Layers

**Entry / Bootstrap (`run.py`):**
- Purpose: Application entry point, pre-startup checks, and migration runner
- Location: `run.py`
- Contains: Environment variable resolution (including `_FILE` secrets), `.env` loading, data directory validation, Alembic migration execution, schema repair, and uvicorn launch
- Depends on: `shared/config.py`, `app/main.py`, `alembic/`
- Used by: Docker CMD / direct invocation

**Main Application (`app/main.py`):**
- Purpose: FastAPI app factory, lifespan management, middleware registration, sub-app mounting, dashboard API endpoints
- Location: `app/main.py`
- Contains: Lifespan startup/shutdown (DB init, scheduler start/stop, shared session cleanup), security headers middleware, auth middleware, dashboard route, `/api/system-status`, `/api/debug-info`, `/api/update-check`, `/health`, WebSocket endpoint
- Depends on: `shared/`, all three tool packages
- Used by: uvicorn (via `run.py`)

**Routers (`app/routers/`):**
- Purpose: Dashboard-level API routes for auth and UniFi config management
- Location: `app/routers/auth.py`, `app/routers/config.py`
- Contains: Session-based login/logout with rate limiting and CSRF protection (`auth.py`); UniFi config CRUD, connection testing, gateway check (`config.py`)
- Depends on: `shared/database.py`, `shared/crypto.py`, `shared/unifi_client.py`, `shared/cache.py`
- Used by: `app/main.py` (router inclusion)

**Tool Sub-Applications (`tools/`):**
- Purpose: Self-contained feature modules mounted as FastAPI sub-apps
- Location: `tools/wifi_stalker/`, `tools/threat_watch/`, `tools/network_pulse/`
- Contains: Each tool has `main.py` (app factory), `scheduler.py` (background polling), `models.py` (Pydantic schemas), `database.py` (SQLAlchemy ORM models), `routers/` (API endpoints), `templates/` (Jinja2 HTML), `static/` (CSS/JS)
- Depends on: `shared/` package for DB, UniFi client, config, cache, WebSocket, webhooks
- Used by: `app/main.py` mounts them at `/stalker`, `/threats`, `/pulse`

**Shared Infrastructure (`shared/`):**
- Purpose: Cross-cutting concerns used by all tools
- Location: `shared/`
- Contains:
  - `unifi_client.py` — UniFi OS API client (aiohttp-based, 1800+ lines), device model mappings, all controller API calls
  - `unifi_session.py` — Singleton shared session to avoid repeated logins across schedulers
  - `database.py` — Async SQLAlchemy engine/session factory (singleton)
  - `config.py` — Pydantic-settings configuration from env vars (singleton)
  - `cache.py` — In-memory TTL cache for gateway info, IPS settings, AP info, update checks
  - `crypto.py` — Fernet symmetric encryption for password/API key storage
  - `webhooks.py` — Webhook delivery (Slack, Discord, n8n formats)
  - `websocket_manager.py` — WebSocket connection tracking and broadcast
  - `url_validator.py` — URL validation utility
  - `models/base.py` — SQLAlchemy declarative base
  - `models/unifi_config.py` — UniFi controller config DB model (single-row table)
- Depends on: External packages (aiohttp, SQLAlchemy, pydantic-settings, cryptography)
- Used by: All tools and `app/`

**Database Models (per tool):**
- Purpose: SQLAlchemy ORM models for persistent storage
- Location: `tools/wifi_stalker/database.py`, `tools/threat_watch/database.py`, `shared/models/unifi_config.py`
- Contains: `TrackedDevice`, `ConnectionHistory`, `WebhookConfig`, `HourlyPresence` (WiFi Stalker); `ThreatEvent`, `ThreatWebhookConfig`, `ThreatIgnoreRule` (Threat Watch); `UniFiConfig` (shared)
- Network Pulse has no database models (stateless, in-memory only via scheduler cache)

**Frontend (`app/templates/`, `tools/*/templates/`, `tools/*/static/`):**
- Purpose: Server-rendered HTML dashboards with client-side JS
- Location: Each tool has its own `templates/` and `static/` directories
- Contains: Jinja2 templates, vanilla JS (WiFi Stalker, main dashboard), Alpine.js (Network Pulse), CSS per tool
- Depends on: REST API endpoints for data fetching, WebSocket for real-time updates

## Data Flow

**Dashboard System Status (polled every 60s by frontend JS):**

1. `dashboard.html` JS calls `GET /api/system-status`
2. `app/main.py` reads `UniFiConfig` from DB, decrypts credentials via `shared/crypto.py`
3. Creates `UniFiClient`, connects, calls `get_system_info()`, `get_health()`, `get_gateway_info()`, `get_ips_settings()`
4. Caches results in `shared/cache.py` (30s TTL) for reuse by other endpoints
5. Returns JSON with system info, health subsystems, gateway details, IPS settings
6. Frontend renders WAN status, device counts, subsystem health cards

**WiFi Stalker Background Refresh (every 60s via APScheduler):**

1. `tools/wifi_stalker/scheduler.py` `refresh_tracked_devices()` fires on interval
2. Gets shared UniFi client via `shared/unifi_session.py` (lazy init, persistent connection)
3. Calls `client.get_clients()` to get all connected clients from UniFi
4. For each `TrackedDevice` in DB: updates connection status, AP, signal, IP, radio band
5. Creates/closes `ConnectionHistory` records on connect/disconnect/roam events
6. Delivers webhooks via `shared/webhooks.py` for status change events
7. Broadcasts updates via `shared/websocket_manager.py` to connected browsers

**Threat Watch Background Refresh (every 60s via APScheduler):**

1. `tools/threat_watch/scheduler.py` fires on interval
2. Gets shared UniFi client, calls `client.get_ips_events()` which internally calls `get_traffic_flows()`
3. v2 traffic-flows API returns events; `_normalize_v2_event()` flattens them to legacy field names
4. Scheduler's `_parse_legacy_ips_event()` extracts fields into `ThreatEvent` DB records
5. Applies ignore rules, delivers threat webhooks for new events
6. Purges events older than 30 days (hourly)

**Network Pulse Background Refresh (every 60s via APScheduler):**

1. `tools/network_pulse/scheduler.py` polls UniFi for health, system info, clients, AP details
2. Assembles `DashboardData` Pydantic model in memory (no DB persistence)
3. Broadcasts full dashboard snapshot via WebSocket to connected browsers
4. Frontend (Alpine.js) reactively updates all dashboard widgets

**State Management:**
- **Server-side:** SQLite DB for persistent data (tracked devices, threat events, webhooks, config). In-memory cache (`shared/cache.py`) for frequently accessed UniFi data with 30s TTL
- **Client-side:** DOM state managed by vanilla JS (dashboard, WiFi Stalker, Threat Watch) or Alpine.js (Network Pulse). No client-side state persistence

## Key Abstractions

**UniFiClient (`shared/unifi_client.py`):**
- Purpose: Encapsulates all communication with the UniFi controller API
- Pattern: Async context-managed client with connect/disconnect lifecycle
- Key methods: `connect()`, `get_system_info()`, `get_health()`, `get_clients()`, `get_ips_events()`, `get_gateway_info()`, `get_ips_settings()`, `get_access_points()`, `get_ap_details()`
- Auto-detects UniFi OS during connection
- All API calls use `/proxy/network/api/` prefix via aiohttp

**Shared Session (`shared/unifi_session.py`):**
- Purpose: Single persistent UniFi connection shared across all tool schedulers
- Pattern: Lazy-initialized singleton with automatic reconnection
- Avoids repeated logins that trigger fail2ban on username/password controllers
- Invalidated on config changes via `invalidate_shared_client()`

**Tool Sub-Application Pattern:**
- Purpose: Each tool is a self-contained FastAPI app created via `create_app()` factory
- Examples: `tools/wifi_stalker/main.py`, `tools/threat_watch/main.py`, `tools/network_pulse/main.py`
- Pattern: Factory function returns configured FastAPI instance with routers, static files, templates, and inline route handlers
- Mounted in main app: `app.mount("/stalker", stalker_app)`, `app.mount("/threats", threat_watch_app)`, `app.mount("/pulse", pulse_app)`

**Background Scheduler Pattern:**
- Purpose: Periodic polling of UniFi API for each tool
- Examples: `tools/wifi_stalker/scheduler.py`, `tools/threat_watch/scheduler.py`, `tools/network_pulse/scheduler.py`
- Pattern: APScheduler `AsyncIOScheduler` with `IntervalTrigger`, started/stopped in app lifespan. Each has `start_scheduler()`, `stop_scheduler()`, `get_last_refresh()` functions
- All schedulers use the shared UniFi session

**Singleton Pattern:**
- Used for: `Database` (`shared/database.py`), `ToolkitSettings` (`shared/config.py`), `WebSocketManager` (`shared/websocket_manager.py`), shared UniFi client (`shared/unifi_session.py`), in-memory cache (`shared/cache.py`)
- Pattern: Module-level `_instance` variable with `get_*()` accessor function

## Entry Points

**Application Entry (`run.py`):**
- Location: `run.py`
- Triggers: `python run.py` or Docker CMD
- Responsibilities: Resolve `_FILE` env vars, load `.env`, validate `ENCRYPTION_KEY`, check data directory, run Alembic migrations + schema repair, launch uvicorn

**FastAPI App (`app/main.py`):**
- Location: `app/main.py`
- Triggers: uvicorn imports `app.main:app`
- Responsibilities: Lifespan (DB init, scheduler start/stop), middleware, route registration, sub-app mounting

**Dashboard (`GET /`):**
- Location: `app/main.py` root route
- Triggers: Browser navigation
- Responsibilities: Renders `dashboard.html` with version info and auth state

**Tool Dashboards:**
- `/stalker/` → `tools/wifi_stalker/main.py` dashboard route
- `/threats/` → `tools/threat_watch/main.py` dashboard route
- `/pulse/` → `tools/network_pulse/main.py` dashboard route
- `/pulse/ap/{ap_mac}` → Network Pulse AP detail page

## Error Handling

**Strategy:** Defensive error handling with graceful degradation. Failures in one tool do not crash others.

**Patterns:**
- UniFi API calls: try/except with logging, return error dict or None. Frontend shows "not connected" state
- Database operations: SQLAlchemy exceptions caught at router level, returned as HTTP error responses
- Migration failures: Schema sync errors auto-recovered by stamping to head + schema repair. Unknown errors logged but app continues
- Shared session: Auto-reconnects on stale session; returns None if config missing or connection fails
- Cache misses: Graceful fallback to fresh API calls when cached data is expired or absent
- WebSocket: Disconnected clients silently removed from active connections list
- Update check: Silently fails if GitHub is unreachable (badge hidden in UI)

## Cross-Cutting Concerns

**Logging:** Python `logging` module throughout. Log level configurable via `LOG_LEVEL` env var (default: INFO). Format: `%(asctime)s - %(name)s - %(levelname)s - %(message)s`

**Validation:** Pydantic models for all API request/response schemas. MAC address normalization in WiFi Stalker. URL validation via `shared/url_validator.py`

**Authentication:** Optional session-based auth, enabled only when `DEPLOYMENT_TYPE=production`. In-memory session store with bcrypt password verification, rate limiting (5 attempts per 5 min), CSRF protection via `X-Requested-With` header for API mutations. WebSocket connections also require valid session in production mode

**Security:** Fernet encryption for stored credentials (`shared/crypto.py`). Security headers middleware (X-Content-Type-Options, X-Frame-Options, X-XSS-Protection, Referrer-Policy). SSL verification configurable per UniFi connection

---

*Architecture analysis: 2026-03-18*
