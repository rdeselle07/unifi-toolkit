# Codebase Concerns

**Analysis Date:** 2026-03-18

## Tech Debt

**Monolithic UniFi Client (1830 lines):**
- Issue: `shared/unifi_client.py` is the largest file in the codebase at 1830 lines. It contains model code mappings (~150 lines of dicts), connection logic, API calls for clients/APs/switches/gateways/IPS/traffic-flows/bandwidth/health, and v2 event normalization all in one file.
- Files: `shared/unifi_client.py`
- Impact: Difficult to navigate, high cognitive load when modifying. Any change to this file risks breaking unrelated functionality.
- Fix approach: Extract model code mappings to `shared/unifi_models.py`. Consider splitting API methods into logical groups (e.g., `unifi_client_devices.py`, `unifi_client_ips.py`, `unifi_client_stats.py`) or use a mixin pattern.

**Duplicate `run_migrations()` Function:**
- Issue: Migration logic is implemented twice -- once in `run.py` (lines 154-197, called at startup) and again in `app/main.py` (lines 45-96, never called). The `app/main.py` version is dead code with a comment "Note: Migrations are now run in run.py BEFORE uvicorn starts" at line 115.
- Files: `run.py` (lines 154-197), `app/main.py` (lines 45-96)
- Impact: Maintenance confusion -- changes to migration logic might be applied to the wrong copy. Dead code adds noise.
- Fix approach: Delete the `run_migrations()` function from `app/main.py`.

**`async for` Session Pattern Used as Context Manager Workaround:**
- Issue: Database sessions are obtained via `async for session in db.get_session()` with a `break` at the end, used in 8 locations across the codebase. This is an unusual pattern -- the async generator yields once and callers break after using it. Standard pattern would be `async with` context manager.
- Files: `app/main.py:370`, `shared/unifi_session.py:58`, `tools/wifi_stalker/scheduler.py:77,503,603`, `tools/threat_watch/scheduler.py:269,371`
- Impact: Confusing to new contributors. The `break` is easy to forget. The FastAPI dependency injection in router endpoints (`Depends(get_db_session)`) already handles this correctly -- only the manual usage sites have this pattern.
- Fix approach: Add an `@asynccontextmanager` wrapper to `Database` class for use outside of FastAPI dependency injection, or convert `get_session()` to return an async context manager.

**Version Maintained in Three Files:**
- Issue: Application version string must be kept in sync across `pyproject.toml`, `app/__init__.py`, and `app/main.py` (FastAPI constructor). No automated sync mechanism exists.
- Files: `pyproject.toml`, `app/__init__.py`, `app/main.py:172`
- Impact: Version drift if one file is missed during a release. This has been a documented concern since early versions.
- Fix approach: Read version from a single source (e.g., `app/__init__.py`) and import it in `app/main.py`. Use `importlib.metadata` or a build-time substitution for `pyproject.toml`.

**Bare `except:` Clauses:**
- Issue: Two bare `except:` clauses exist that catch all exceptions silently, including `SystemExit` and `KeyboardInterrupt`.
- Files: `shared/unifi_client.py:309` (in `_try_unifi_os_login`, catches JSON parse error), `shared/unifi_client.py:1829` (in `__del__`, catches event loop errors)
- Impact: Can mask unexpected errors during debugging. The `__del__` one is less concerning since cleanup failure is generally non-critical, but the login one could hide real issues.
- Fix approach: Replace with `except Exception:` at minimum, or catch specific exceptions (`json.JSONDecodeError`, `RuntimeError`).

## Known Bugs

**`__del__` Method Uses Deprecated `get_event_loop()`:**
- Symptoms: `DeprecationWarning` on Python 3.12+ when `UniFiClient` is garbage collected. May raise `RuntimeError` if no event loop exists.
- Files: `shared/unifi_client.py:1817-1830`
- Trigger: Object destruction during shutdown or garbage collection.
- Workaround: The shared session pattern (`shared/unifi_session.py`) handles cleanup properly, so this rarely triggers in practice.

**Fire-and-Forget `asyncio.create_task()` Without Reference:**
- Symptoms: If `refresh_single_device` raises an exception, it is silently swallowed as an unhandled task exception. The task may also be garbage collected before completion.
- Files: `tools/wifi_stalker/routers/devices.py:82`
- Trigger: Adding a new device to WiFi Stalker tracking.
- Workaround: None needed for correctness -- the next scheduled refresh will pick up the device. But unhandled exceptions are invisible.

## Security Considerations

**In-Memory Session Store:**
- Risk: Sessions are stored in a Python dict (`_sessions` in `app/routers/auth.py:31`). All sessions are lost on application restart, forcing re-login. Not a security risk per se, but the lack of session persistence means no ability to audit or revoke sessions across restarts.
- Files: `app/routers/auth.py:31`
- Current mitigation: Acceptable for single-user deployment. Sessions expire after 7 days. Rate limiting (5 attempts per 5 minutes) is in place.
- Recommendations: Document that this is by design for the target deployment (single-user Docker). If multi-user support is ever added, move to database-backed sessions.

**In-Memory Rate Limiting:**
- Risk: Rate limiting state (`_login_attempts` dict in `app/routers/auth.py:34`) is lost on restart. An attacker could brute-force by restarting the container, though this requires host access which would already be game over.
- Files: `app/routers/auth.py:34`
- Current mitigation: Acceptable given deployment model (local network or behind reverse proxy).
- Recommendations: No action needed for current threat model.

**SSL Verification Disabled by Default:**
- Risk: `verify_ssl` defaults to `False` in both `ToolkitSettings` (`shared/config.py:33`) and `UniFiClient` (`shared/unifi_client.py:180`). All UniFi controller connections skip SSL verification.
- Files: `shared/config.py:33`, `shared/unifi_client.py:201,221-224`
- Current mitigation: UniFi controllers use self-signed certificates by default, so this is the practical default. Users can enable verification via config.
- Recommendations: This is appropriate for the use case. Self-signed certs on UniFi controllers make SSL verification impractical without certificate pinning.

**SSRF Protection on Webhooks:**
- Risk: Webhook URLs are validated against private IP ranges via `shared/url_validator.py`, but unresolvable hostnames are allowed through (line 151: "Can't resolve - allow it"). A DNS rebinding attack could potentially reach internal services.
- Files: `shared/url_validator.py:145-151`
- Current mitigation: SSRF validation blocks known private ranges, cloud metadata IPs, and localhost. The unresolvable hostname pass-through is a deliberate trade-off documented in code comments.
- Recommendations: Consider validating at delivery time as well (resolve hostname just before sending webhook and re-check against blocked ranges).

**Webhook URLs Stored Unencrypted:**
- Risk: Webhook destination URLs (which may contain tokens in query params for Slack/Discord) are stored in plaintext in the SQLite database, unlike credentials which use Fernet encryption.
- Files: `tools/threat_watch/database.py`, `tools/wifi_stalker/database.py`
- Current mitigation: Database file permissions restrict access. The tool runs in a Docker container with a non-root user.
- Recommendations: If webhook URLs contain sensitive tokens, consider encrypting the URL field similar to how passwords are handled in `shared/crypto.py`.

## Performance Bottlenecks

**Redundant API Calls to `/stat/device`:**
- Problem: Multiple methods in `UniFiClient` independently call `/stat/device` -- `get_access_points()`, `get_ap_name_by_mac()` (twice -- once for APs, once for all devices), `get_switch_name_by_mac()`, `get_system_info()`, `has_gateway()`, `get_gateway_info()`, `get_ips_settings()`, `get_ap_details()`. The `/api/system-status` endpoint in `app/main.py` calls several of these in sequence.
- Files: `shared/unifi_client.py` (methods at lines 445, 491, 554, 1115, 1379, 1421, 1695), `app/main.py:355-472`
- Cause: Each method is self-contained and fetches its own device list. No shared device cache within a single request cycle.
- Improvement path: Add a request-scoped device cache to `UniFiClient` (populate once on first `/stat/device` call, reuse for subsequent calls in the same polling cycle). The existing `shared/cache.py` has a 30-second TTL that partially mitigates this for the dashboard, but the schedulers create fresh clients.

**`get_ap_name_by_mac()` Does Two API Calls:**
- Problem: First calls `get_access_points()` (which calls `/stat/device`), then if not found, calls `/stat/device` again directly to check BSSID mappings in `vap_table`.
- Files: `shared/unifi_client.py:491-552`
- Cause: The initial AP lookup only checks device MACs, not radio BSSIDs. The second call is needed for gateway devices with built-in radios.
- Improvement path: Merge into a single `/stat/device` call that checks both device MAC and vap_table BSSIDs.

**`is_client_blocked()` Fetches Entire User List:**
- Problem: Calls `/rest/user` which returns ALL known users (not just active clients) to check if one MAC is blocked.
- Files: `shared/unifi_client.py:652-681`
- Cause: No UniFi API endpoint exists to check a single user's blocked status.
- Improvement path: Cache the blocked status from the client list when available, rather than making an additional API call.

## Fragile Areas

**Schema Repair System (`run.py:200-271`):**
- Files: `run.py:200-271`
- Why fragile: Every new Alembic migration that adds a column MUST also be added to `_repair_schema()`. If forgotten, users who upgrade from an old version may hit missing column errors that are hard to diagnose. The CLAUDE.md documents this requirement, but it depends on developer discipline.
- Safe modification: When adding a new Alembic migration with column additions, always add corresponding entries to `_repair_schema()`. Test by creating a fresh database, running `alembic stamp head` (to simulate the skip scenario), then starting the app.
- Test coverage: No automated tests verify that `_repair_schema()` covers all migration-added columns.

**v2 Traffic Flows Payload Auto-Detection:**
- Files: `shared/unifi_client.py:850-863` (`_v2_uses_new_payload` flag)
- Why fragile: The client auto-detects whether the controller supports the filtered v2 payload by trying it and falling back. The detection result is cached on the client instance (`_v2_uses_new_payload`). If firmware updates change behavior, the cached flag persists until the shared session is invalidated.
- Safe modification: The flag resets when the shared client reconnects (new `UniFiClient` instance). Firmware updates typically require controller restarts, which would cause session reconnection.
- Test coverage: No automated tests for payload format detection or fallback behavior.

**Express AP-Mode Detection Logic (Duplicated 6+ Times):**
- Files: `shared/unifi_client.py` (lines 471-474, 1170-1174, 1189-1192, 1402-1407, 1459-1460, 1473-1476, 1722-1725)
- Why fragile: The logic to detect Express in AP-only mode (`device_mode_override == 'mesh'` and model code in `EXPRESS_MODEL_CODES`) is repeated in 6+ locations. If a new Express variant is released with different detection logic, all instances must be updated.
- Safe modification: Always grep for `device_mode_override` and `EXPRESS_MODEL_CODES` when changing detection logic.
- Test coverage: No automated tests.

## Scaling Limits

**SQLite Database:**
- Current capacity: Suitable for single-site deployments with hundreds of threat events per day and dozens of tracked devices.
- Limit: SQLite handles concurrent reads well but has single-writer limitation. The async SQLAlchemy + aiosqlite setup serializes writes. Could become a bottleneck if multiple tools poll simultaneously with high event volumes.
- Scaling path: The `DATABASE_URL` is configurable, so switching to PostgreSQL is possible without code changes (SQLAlchemy abstraction). However, some raw SQLite queries in `_repair_schema()` would need adaptation.

**In-Memory Cache (No Eviction Policy):**
- Current capacity: Stores gateway info, IPS settings, AP info, system status, and update check results.
- Limit: Cache grows only to a fixed number of keys (5-6 entries), so memory is not a concern. However, there is no cache warming -- first request after TTL expiry always hits the UniFi controller.
- Scaling path: Not a concern for current deployment model.

## Dependencies at Risk

**No Pinned Versions in Requirements:**
- Risk: `requirements.txt` uses `>=` minimum versions with no upper bounds. A `pip install` could pull incompatible major versions of any dependency.
- Impact: Build reproducibility issues. A new major version of FastAPI, SQLAlchemy, or Pydantic could break the application without any code changes.
- Files: `requirements.txt`
- Migration plan: Pin to specific versions or use compatible release specifiers (`~=`) in `requirements.txt`. Use `pip freeze` to generate a lockfile for Docker builds.

**APScheduler 3.x (EOL):**
- Risk: APScheduler 3.x is in maintenance mode. The 4.x rewrite has a completely different API. Currently pinned to `>=3.10.4`.
- Impact: No immediate breakage, but no new features or improvements. The 4.x migration will be a significant effort when needed.
- Files: `requirements.txt`, `tools/threat_watch/scheduler.py`, `tools/wifi_stalker/scheduler.py`, `tools/network_pulse/scheduler.py`
- Migration plan: Not urgent. Monitor APScheduler 4.x stability before migrating.

## Test Coverage Gaps

**No Tests for UniFi Client:**
- What's not tested: The entire `shared/unifi_client.py` (1830 lines) -- connection, authentication, API calls, v2 normalization, model code mappings.
- Files: `shared/unifi_client.py`
- Risk: API response parsing changes could go undetected. The v2 event normalization (`_normalize_v2_event`) is particularly critical and has had multiple bugs fixed (v1.11.1, v1.9.18).
- Priority: High -- this is the core data layer and has been the source of most reported bugs.

**No Tests for Schedulers:**
- What's not tested: Background polling logic in all three schedulers -- WiFi Stalker (`tools/wifi_stalker/scheduler.py`, 652 lines), Threat Watch (`tools/threat_watch/scheduler.py`, 416 lines), Network Pulse (`tools/network_pulse/scheduler.py`, 332 lines).
- Files: `tools/wifi_stalker/scheduler.py`, `tools/threat_watch/scheduler.py`, `tools/network_pulse/scheduler.py`
- Risk: Event parsing, deduplication, webhook triggering, and purge logic are untested. These contain the most complex business logic in the application.
- Priority: High -- scheduler bugs (e.g., v1.9.20 webhook delivery calling wrong function) have caused production issues.

**No Tests for Router Endpoints (Tools):**
- What's not tested: All tool-specific API endpoints -- device CRUD, event listing/filtering, webhook configuration, ignore rules.
- Files: `tools/wifi_stalker/routers/`, `tools/threat_watch/routers/`, `tools/network_pulse/routers/`
- Risk: API contract changes (request/response shapes) could break the frontend without detection.
- Priority: Medium -- the frontend is the primary consumer and manual testing catches most issues.

**Existing Tests Cover Only Shared Utilities:**
- What is tested: `tests/test_auth.py` (263 lines), `tests/test_cache.py` (252 lines), `tests/test_crypto.py` (141 lines), `tests/test_config.py` (121 lines). Total: 777 lines of tests for ~12,200 lines of application code (~6% ratio).
- Files: `tests/`
- Risk: The tested modules (auth, cache, crypto, config) are the most stable parts of the codebase. The untested parts (UniFi client, schedulers, routers) are where bugs actually occur.
- Priority: High -- test effort should shift to the areas with the most churn and bug history.

**No Integration/E2E Tests:**
- What's not tested: Full request flow from HTTP request through to UniFi API mock and database.
- Risk: Interaction bugs between layers (e.g., the v1.9.20 webhook delivery bug where the scheduler called the wrong webhook function) are only caught manually.
- Priority: Medium -- would require UniFi API mocking infrastructure.

---

*Concerns audit: 2026-03-18*
