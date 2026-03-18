# Testing Patterns

**Analysis Date:** 2026-03-18

## Test Framework

**Runner:**
- pytest >= 8.0.0
- Config: `pytest.ini`

**Assertion Library:**
- Built-in `assert` statements (pytest native)

**Async Support:**
- pytest-asyncio >= 0.23.0
- `asyncio_mode = auto` in `pytest.ini` (no manual `@pytest.mark.asyncio` needed)

**Mocking:**
- pytest-mock >= 3.12.0
- `unittest.mock.patch` / `patch.dict` used directly

**HTTP Testing:**
- httpx >= 0.26.0 with `AsyncClient` and `ASGITransport` for FastAPI integration tests

**Coverage:**
- pytest-cov >= 4.1.0 (installed but coverage not enforced)

**Run Commands:**
```bash
pytest                     # Run all tests (verbose by default via pytest.ini)
pytest -x                  # Stop on first failure
pytest --cov=app --cov=shared --cov=tools --cov-report=html  # With coverage
pytest -k "test_cache"     # Run specific test pattern
pytest -m unit             # Run tests by marker
```

## Test File Organization

**Location:**
- Dedicated `tests/` directory at project root (not co-located)

**Naming:**
- Files: `test_{module}.py` matching the module under test
- Classes: `Test{Feature}` grouping related tests
- Functions: `test_{description}` with descriptive snake_case names

**Structure:**
```
tests/
├── __init__.py
├── conftest.py           # Shared fixtures and test setup
├── test_auth.py          # Tests for app/routers/auth.py
├── test_cache.py         # Tests for shared/cache.py
├── test_config.py        # Tests for shared/config.py
└── test_crypto.py        # Tests for shared/crypto.py
```

**Current Coverage:**
- `shared/cache.py` - Fully tested (get/set, TTL expiration, invalidation, age tracking)
- `shared/crypto.py` - Fully tested (encrypt/decrypt, key generation, edge cases)
- `shared/config.py` - Tested (defaults, env overrides, singleton)
- `app/routers/auth.py` - Tested (auth enabled check, password verify, sessions, rate limiting)
- `shared/unifi_client.py` - **Not tested** (1800+ lines, core data fetching)
- `tools/*/scheduler.py` - **Not tested** (background task schedulers)
- `tools/*/routers/*.py` - **Not tested** (API endpoints)
- `shared/webhooks.py` - **Not tested** (webhook delivery)

## Test Markers

Defined in `pytest.ini` but not yet applied to tests:
```ini
markers =
    unit: Unit tests (fast, no external dependencies)
    integration: Integration tests (may require database/network)
    slow: Slow tests that take significant time
    auth: Authentication-related tests
    cache: Caching system tests
    crypto: Encryption and security tests
    config: Configuration management tests
```

## Test Structure

**Suite Organization:**
```python
"""Tests for caching module."""
import pytest
from shared import cache


class TestGatewayInfoCache:
    """Tests for gateway info caching."""

    def setup_method(self):
        """Clear cache before each test."""
        cache.invalidate_all()

    def test_get_gateway_info_returns_none_when_empty(self):
        """Should return None when cache is empty."""
        result = cache.get_gateway_info()
        assert result is None

    def test_set_gateway_info_stores_data(self):
        """Should store gateway info in cache."""
        data = {"has_gateway": True, "gateway_model": "UDM"}
        cache.set_gateway_info(data)
        result = cache.get_gateway_info()
        assert result == data
```

**Patterns:**
- Group related tests in classes (`TestPasswordEncryption`, `TestSessionManagement`)
- Use `setup_method()` for per-test cleanup (clear state like caches, sessions, rate limit stores)
- Descriptive docstrings on every test: `"""Should return None when cache is empty."""`
- Test both happy path and edge cases in same class

## Test Configuration (conftest.py)

**Location:** `tests/conftest.py`

**Environment Setup:**
```python
# Generate a valid Fernet key for testing
from cryptography.fernet import Fernet
_test_key = Fernet.generate_key().decode()

# Set test environment variables before importing app
os.environ["ENCRYPTION_KEY"] = _test_key
os.environ["DATABASE_URL"] = "sqlite+aiosqlite:///:memory:"
os.environ["DEPLOYMENT_TYPE"] = "local"
os.environ["LOG_LEVEL"] = "ERROR"
```

Environment variables are set **before** app imports to ensure test isolation.

**Key Fixtures:**

```python
@pytest.fixture
async def test_db():
    """Create a fresh test database for each test."""
    engine = create_async_engine("sqlite+aiosqlite:///:memory:", echo=False)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
    async with async_session() as session:
        yield session
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()

@pytest.fixture
async def async_client():
    """Async test client for FastAPI."""
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test"
    ) as client:
        yield client

@pytest.fixture
def sample_unifi_config():
    """Sample UniFi controller configuration."""
    return {
        "controller_url": "https://192.168.1.1:8443",
        "username": "admin",
        "password": "test-password",
        "site_id": "default",
        "verify_ssl": False,
    }

@pytest.fixture
def sample_device_data():
    """Sample Wi-Fi client device data from UniFi API."""
    return {
        "mac": "aa:bb:cc:dd:ee:ff",
        "name": "Test Device",
        "ip": "192.168.1.100",
        "ap_mac": "11:22:33:44:55:66",
        "signal": -45,
        "essid": "HomeNetwork",
    }
```

## Mocking

**Framework:** `unittest.mock` (standard library)

**Patterns:**

Environment variable mocking (most common pattern):
```python
from unittest.mock import patch

def test_auth_disabled_in_local_mode(self):
    with patch.dict(os.environ, {"DEPLOYMENT_TYPE": "local"}):
        assert is_auth_enabled() is False
```

Internal state manipulation (for testing expiration, timing):
```python
def test_gateway_info_expires_after_ttl(self):
    data = {"has_gateway": True}
    cache.set_gateway_info(data)
    # Manually expire the cache entry
    cache._cache["gateway_info"]["timestamp"] = (
        datetime.now(timezone.utc) - timedelta(seconds=cache.CACHE_TTL_SECONDS + 1)
    )
    result = cache.get_gateway_info()
    assert result is None
```

Direct state access for assertions:
```python
def test_create_session_stores_username(self):
    token = create_session("testuser")
    assert token in _sessions
    assert _sessions[token]["username"] == "testuser"
```

**What to Mock:**
- Environment variables (via `patch.dict(os.environ, ...)`)
- Time-dependent state (manually set timestamps on cache/session entries)

**What NOT to Mock:**
- The module under test itself (test real behavior)
- Simple data structures (use real dicts/lists)

## Fixtures and Factories

**Test Data:**
- Inline dictionaries for simple test data
- Shared fixtures in `conftest.py` for reusable data (`sample_unifi_config`, `sample_device_data`)
- No factory libraries or builder patterns used

**Database:**
- In-memory SQLite for test isolation: `sqlite+aiosqlite:///:memory:`
- Fresh database per test via fixture (create_all / drop_all lifecycle)

## Coverage

**Requirements:** None enforced (coverage is commented out in `pytest.ini`)

**Available but disabled:**
```ini
# Uncomment in pytest.ini to enable:
# addopts = --cov=app --cov=shared --cov=tools --cov-report=html --cov-report=term
```

**View Coverage:**
```bash
pytest --cov=app --cov=shared --cov=tools --cov-report=html
# Open htmlcov/index.html
```

## Test Types

**Unit Tests:**
- All current tests are unit tests
- Test individual functions in isolation
- No network calls, no disk I/O
- Use in-memory database or direct state manipulation
- Fast execution (seconds)

**Integration Tests:**
- `async_client` fixture exists for FastAPI integration testing but is not currently used in any tests
- `test_db` fixture exists for database integration tests but is not currently used

**E2E Tests:**
- Not used

## Common Patterns

**Testing Edge Cases:**
```python
def test_encrypt_special_characters(self):
    """Should handle passwords with special characters."""
    password = "p@ssw0rd!#$%^&*(){}[]|\\:;\"'<>,.?/~`"
    encrypted = encrypt_password(password)
    decrypted = decrypt_password(encrypted)
    assert decrypted == password

def test_encrypt_unicode_characters(self):
    """Should handle passwords with unicode characters."""
    password = "password_in_other_languages_and_emoji"
    encrypted = encrypt_password(password)
    decrypted = decrypt_password(encrypted)
    assert decrypted == password
```

**Testing Error Cases:**
```python
def test_decrypt_invalid_data_raises_error(self):
    """Should raise error when decrypting invalid data."""
    with pytest.raises(InvalidToken):
        decrypt_password(b"not-valid-encrypted-data")

def test_verify_invalid_hash_returns_false(self):
    """Should return False for invalid password hash."""
    assert verify_password("test", "not-a-valid-bcrypt-hash") is False
```

**Testing Time-Dependent Behavior:**
```python
def test_cache_age_increases_over_time(self):
    """Cache age should increase as time passes."""
    cache.set_gateway_info({"has_gateway": True})
    age1 = cache.get_cache_age("gateway_info")
    time.sleep(0.1)
    age2 = cache.get_cache_age("gateway_info")
    assert age2 > age1
```

**Testing Aliases/Identity:**
```python
def test_api_key_encryption_uses_same_mechanism(self):
    """API key encryption should be an alias for password encryption."""
    assert encrypt_api_key is encrypt_password
    assert decrypt_api_key is decrypt_password
```

**Testing Singleton Behavior:**
```python
def test_get_settings_returns_same_instance(self):
    """Should return the same instance (singleton pattern)."""
    settings1 = get_settings()
    settings2 = get_settings()
    assert settings1 is settings2
```

## pytest.ini Configuration

```ini
[pytest]
python_files = test_*.py
python_classes = Test*
python_functions = test_*
minversion = 8.0
testpaths = tests
addopts = -v --durations=10 -l -ra
asyncio_mode = auto

filterwarnings =
    ignore::DeprecationWarning:pydantic._internal._config
    ignore::DeprecationWarning:app.routers.auth
```

**Key Settings:**
- `-v`: Verbose output by default
- `--durations=10`: Show 10 slowest tests
- `-l`: Show local variables in tracebacks
- `-ra`: Show summary of all test outcomes except passed
- `asyncio_mode = auto`: Async tests detected automatically

## Adding New Tests

When adding tests for a new module:

1. Create `tests/test_{module_name}.py`
2. Import the functions/classes under test directly
3. Group related tests in `Test{Feature}` classes
4. Add `setup_method()` for state cleanup if testing stateful code
5. Write descriptive docstrings on every test
6. Test happy path, edge cases, and error cases
7. Use existing fixtures from `conftest.py` when applicable
8. No markers needed (markers are defined but not actively used)

---

*Testing analysis: 2026-03-18*
