# External Integrations

**Analysis Date:** 2026-03-18

## APIs & External Services

**UniFi Controller API (Primary Integration):**
- Purpose: All network data - device status, health, clients, IPS events, gateway info
- Client: Custom async client in `shared/unifi_client.py` (1800+ lines)
- Transport: `aiohttp` with persistent sessions via `shared/unifi_session.py`
- Auth: API key (preferred) or username/password - stored encrypted in SQLite
- API prefix: `/proxy/network/api/` (UniFi OS only, no legacy controller support)
- Connection: Singleton shared client across all schedulers to avoid fail2ban triggers
- SSL: Configurable via `UNIFI_VERIFY_SSL` (default: `false` for self-signed certs)
- Config stored in DB: `shared/models/unifi_config.py` (controller URL, encrypted credentials, site ID)
- Key endpoints used:
  - Health/system info: `get_health()`, `get_system_info()`, `get_gateway_info()`
  - IPS/IDS events: `get_ips_events()`, `get_traffic_flows()` (v2 API)
  - Client data: `get_clients()`, `get_access_points()`
  - IPS settings: `get_ips_settings()`
- Important: Only stable/GA firmware supported. Early Access firmware is explicitly unsupported.
- Important: `unifi.ui.com` cloud access is NOT supported - must use local IP/hostname

**GitHub API:**
- Purpose: Update check - compares running version against latest GitHub release
- Endpoint: `https://api.github.com/repos/Crosstalk-Solutions/unifi-toolkit/releases/latest`
- Client: `aiohttp` one-off request in `app/main.py:check_for_update()`
- Auth: None (public API, unauthenticated)
- Rate: Cached for 1 hour (`shared/cache.py:UPDATE_CHECK_TTL_SECONDS = 3600`)
- Failure: Graceful - badge hidden silently if GitHub is unreachable

## Data Storage

**Database:**
- SQLite via async SQLAlchemy
  - Connection: `DATABASE_URL` env var (default: `sqlite+aiosqlite:///./data/unifi_toolkit.db`)
  - Client: SQLAlchemy 2.0 async (`create_async_engine`, `AsyncSession`)
  - ORM base: `shared/models/base.py:Base`
  - Session factory: Singleton in `shared/database.py:get_database()`
  - Tables:
    - `unifi_config` - Controller connection settings (encrypted credentials)
    - `stalker_tracked_devices` - WiFi Stalker tracked devices
    - `stalker_connection_history` - Device connection event log
    - `stalker_webhook_config` - WiFi Stalker webhook settings
    - `stalker_hourly_presence` - Presence heatmap data
    - `threats_events` - IDS/IPS threat events (30-day retention, auto-purged)
    - `threats_webhook_config` - Threat Watch webhook settings
    - `threats_ignore_rules` - Threat Watch ignore rules
  - Migrations: Alembic (`alembic/` directory, config in `alembic.ini`)
  - Schema repair: `run.py:_repair_schema()` runs on every startup after migrations

**File Storage:**
- Local filesystem only (`./data/` directory)
- SQLite database file is the only persisted data
- Docker volume mount: `./data:/app/data`

**Caching:**
- In-memory dict cache (`shared/cache.py`)
- TTL-based: 30 seconds for gateway/IPS/system data, 1 hour for update checks
- No Redis or external cache service
- Cache keys: `gateway_info`, `ips_settings`, `ap_info`, `system_status`, `update_check`

## Authentication & Identity

**App Auth (Production Mode):**
- Custom session-based auth (`app/routers/auth.py`)
- Enabled only when `DEPLOYMENT_TYPE=production`
- Credentials: `AUTH_USERNAME` + `AUTH_PASSWORD_HASH` (bcrypt) from env vars
- Sessions: In-memory dict (lost on restart, acceptable for single-user)
- Session token: `secrets.token_urlsafe(32)`, stored in `session_token` cookie
- Cookie: `httponly=True`, `secure=True`, `samesite=lax`, 7-day expiry
- Rate limiting: 5 failed attempts per 5 minutes per IP
- CSRF: `X-Requested-With: XMLHttpRequest` header required for state-changing API requests
- WebSocket auth: Session cookie validated before accepting WS connections

**UniFi Controller Auth:**
- API key auth (preferred) or username/password
- Credentials encrypted at rest with Fernet symmetric encryption (`shared/crypto.py`)
- Encryption key: `ENCRYPTION_KEY` env var (Fernet key)
- Configured via web UI after first launch, stored in `unifi_config` DB table

## Monitoring & Observability

**Error Tracking:**
- None (no Sentry, Datadog, etc.)

**Logging:**
- Python `logging` module, configured in `app/main.py`
- Format: `%(asctime)s - %(name)s - %(levelname)s - %(message)s`
- Level: Configurable via `LOG_LEVEL` env var (default: `INFO`)
- Uvicorn access logs enabled

**Health Check:**
- `GET /health` - Returns version info and tool versions
- Docker HEALTHCHECK: `curl -f http://localhost:8000/health` (30s interval)

**Debug Info:**
- `GET /api/debug-info` - Non-sensitive system info for issue reporting
- Returns: app version, tool versions, deployment type, Python version, gateway model/firmware, AP list
- Dashboard footer: "Debug Info" modal with copy-to-clipboard
- "Report Issue" link pre-populates GitHub issue with debug info

## CI/CD & Deployment

**Hosting:**
- Self-hosted Docker containers (Synology NAS, Unraid, TrueNAS, etc.)
- No cloud hosting / no managed platform

**CI Pipeline:**
- GitHub Actions (`.github/workflows/docker-publish.yml`)
- Triggers: Push to `main`, version tags (`v*`), PRs to `main`
- Build: Docker Buildx, multi-arch (`linux/amd64`, `linux/arm64`)
- Cache: GitHub Actions cache (`type=gha`)
- Push: GHCR + Docker Hub (on non-PR events)
- Tags: `latest` (main branch), semver (`1.0.0`, `1.0`), commit SHA

**Stale Issues:**
- GitHub Actions (`.github/workflows/stale-issues.yml`)
- 7-day stale warning, 7-day auto-close

**Reverse Proxy (Production):**
- Caddy 2 (`caddy:2-alpine`) - automatic HTTPS via Let's Encrypt
- Activated via docker-compose `production` profile
- Config: `Caddyfile` (mounted read-only)

## Environment Configuration

**Required env vars:**
- `ENCRYPTION_KEY` - Fernet key for credential encryption (app will not start without it)

**Optional env vars:**
- `DEPLOYMENT_TYPE` - `local` or `production` (default: `local`)
- `DATABASE_URL` - SQLite connection string (default: `sqlite+aiosqlite:///./data/unifi_toolkit.db`)
- `LOG_LEVEL` - Logging level (default: `INFO`)
- `APP_PORT` - Server port (default: `8000`)
- `AUTH_USERNAME` / `AUTH_PASSWORD_HASH` - Production auth credentials
- `DOMAIN` - HTTPS domain for Caddy (production mode)
- `STALKER_REFRESH_INTERVAL` - Device poll interval in seconds (default: `60`)
- `UNIFI_*` vars - Optional bootstrap (usually configured via web UI)

**Secrets location:**
- `.env` file (mounted read-only into container)
- Docker Swarm secrets via `_FILE` suffix env vars (e.g., `ENCRYPTION_KEY_FILE=/run/secrets/key`)

## Webhooks & Callbacks

**Outgoing Webhooks (User-Configured):**

WiFi Stalker (`shared/webhooks.py:deliver_webhook()`):
- Events: `connected`, `disconnected`, `roamed`, `blocked`, `unblocked`
- Config stored in: `stalker_webhook_config` DB table
- Router: `tools/wifi_stalker/routers/webhooks.py`

Threat Watch (`shared/webhooks.py:deliver_threat_webhook()`):
- Events: `threat_detected` (with severity, action, IPs, category)
- Config stored in: `threats_webhook_config` DB table
- Router: `tools/threat_watch/routers/webhooks.py`

**Supported webhook formats:**
- Slack - Attachment-style messages with color, fields, footer
- Discord - Embed-style messages with color, fields, footer, timestamp
- n8n / Generic - Raw JSON payload with structured event data

**Incoming Webhooks:**
- None - the app polls the UniFi controller, does not receive inbound webhooks

## Real-Time Communication

**WebSocket:**
- Endpoint: `GET /ws` (`app/main.py:websocket_endpoint()`)
- Manager: `shared/websocket_manager.py:WebSocketManager`
- Purpose: Real-time device updates pushed to browser
- Auth: Session cookie validated in production mode
- Messages: `device_update`, `status_update`, `pong` (heartbeat)
- Pattern: Server broadcasts to all connected clients; no client-to-server data flow beyond ping

---

*Integration audit: 2026-03-18*
