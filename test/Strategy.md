
# Testing strategy

## Testing pyramid

```
         /  Smoke  \          optional, real MBTA, manual
        /    E2E    \         real HTTP, real server, fake MBTA
       / Integration \        components wired, mocked boundaries
      /     Unit      \       pure functions, no I/O
```

Every development cycle must pass all levels except smoke before merging.

## Level 1: Unit tests (fast, no I/O)

Pure function tests. No HTTP, no cache, no time dependency unless frozen.

### Selection logic (test_selection.py)

- Selects earliest future arrival from a list of candidates.
- Discards arrivals without a timestamp.
- Discards arrivals in the past relative to `as_of`.
- Returns up to 2 alternatives after the next arrival.
- Resolves arrival_time vs departure_time (arrival preferred, departure as fallback).
- Returns empty when all candidates are past or missing timestamps.

### Minutes computation (test_selection.py)

- `minutes` = floor of time difference in seconds / 60.
- `leave_in_minutes` = minutes - walk_minutes.
- `leave_in_minutes` can be negative (bus too close to catch on foot).
- `minutes` is never negative (past arrivals are filtered before computation).

### Config parsing (test_config.py)

- Loads valid config.yaml with all fields.
- Applies defaults (walk_minutes=0, cache_ttl=20, stale_max_age=300).
- Rejects config with missing required fields (stops, route_id, stop_id, key).
- Rejects config with duplicate keys.
- Environment variables override config file values (MBTA_API_KEY, API_KEY).

## Level 2: Integration tests (mocked boundaries)

Components wired together, but external HTTP (MBTA) is mocked in-process.

### Cache behavior (test_cache.py)

- Returns cached data within TTL (cache hit).
- Triggers refresh after TTL expires (cache miss).
- Serves stale data when upstream fails and cache is within stale_max_age.
- Re-filters stale arrivals against current time (drops past arrivals).
- Returns no_service when stale data has no surviving future arrivals.
- Returns error on cold start with upstream failure (no cache exists).
- Uses frozen time (freezegun or similar) for deterministic TTL tests.

### Board service (test_board.py)

- Orchestrates MBTA client, cache, and selection correctly.
- Falls back from predictions to schedules when predictions yield no future arrivals.
- Returns no_service when both predictions and schedules are empty.
- Handles MBTA errors gracefully (stale or error status).

### API endpoints (test_api.py)

- Uses FastAPI TestClient with mocked board service.
- `/v1/board` returns valid BoardResponse JSON schema.
- `/v1/board/{key}` returns valid BoardItemResponse for a known key.
- `/v1/board/{key}` returns 404 for unknown key.
- `/health` returns 200 with `{"status": "healthy"}`.
- X-API-Key enforcement when api_key is configured.
- `/health` is accessible without API key.

## Level 3: End-to-end tests (real HTTP, fake MBTA)

The real service runs as an HTTP server. A fake MBTA server returns fixture data.
Tests make actual HTTP requests over the network. Nothing is mocked in-process.

### Setup (conftest.py fixtures)

- **fake_mbta_server**: a lightweight HTTP server (pytest-httpserver) that serves
  fixture MBTA JSON responses. Configurable per test to return predictions,
  schedules, errors, or empty results.
- **bus_tracker_server**: starts the actual bus-tracker FastAPI app via uvicorn on a
  random port, configured to point at the fake MBTA server. Yields the base URL.
  Torn down after each test module.
- **config_file**: writes a temporary config.yaml pointing at the fake MBTA server URL.

### Test scenarios (test_e2e.py)

Each test uses real httpx calls against the running bus-tracker server.

1. **Happy path: realtime predictions available**
   - Fake MBTA returns predictions with future arrival times.
   - `GET /v1/board` returns status ok, arrival with source realtime, correct minutes
     and leave_in_minutes, up to 2 alternatives.

2. **Fallback to schedule**
   - Fake MBTA returns empty predictions, but schedules with future times.
   - Response has status ok, arrival with source schedule.

3. **No service**
   - Fake MBTA returns empty predictions and empty schedules.
   - Response has status no_service, arrival is null, alternatives is empty.

4. **MBTA down, stale cache**
   - First request succeeds (primes cache). Then fake MBTA starts returning 500.
   - Second request returns status stale with recomputed minutes.
   - Verifies stale_as_of is present and older than as_of.

5. **MBTA down, stale cache expired (all arrivals past)**
   - Prime cache, wait (or use time manipulation) until cached arrivals are past.
   - Fake MBTA returns 500.
   - Response is no_service (not stale with past data).

6. **Cold start failure**
   - Fake MBTA returns 500 from the start. No cache exists.
   - Response is status error with code mbta_unreachable.

7. **Board key lookup**
   - `GET /v1/board/{key}` returns the correct single item.
   - `GET /v1/board/nonexistent` returns 404.

8. **Health check**
   - `GET /health` returns 200 regardless of MBTA state.

9. **API key enforcement**
   - Configure api_key in config.
   - Requests without X-API-Key header get 401.
   - Requests with correct X-API-Key succeed.
   - `/health` works without API key.

10. **Walk time computation end-to-end**
    - Configure walk_minutes=5. Fake MBTA returns a bus arriving in 8 minutes.
    - Verify arrival.minutes=8, arrival.leave_in_minutes=3.

## Level 4: Smoke tests (optional, real MBTA)

Not run in CI. Requires a real MBTA API key and network access.
Run manually to verify the live integration works.

### Smoke scenarios (test_smoke.py)

- Start the service with real config pointing at real MBTA API.
- `GET /v1/board` returns a valid response with real bus data.
- Response times are plausible (arrival times are in the near future, not 1970).
- Second request within cache TTL is a cache hit (fast response).

Mark with `@pytest.mark.smoke` so they are skipped by default:

```bash
# Normal test run (all levels except smoke)
pytest

# Include smoke tests (requires MBTA_API_KEY)
pytest -m smoke
```

## Fixtures

- `test/fixtures/mbta/predictions_normal.json` -- predictions with 3 future arrivals.
- `test/fixtures/mbta/predictions_empty.json` -- empty predictions data array.
- `test/fixtures/mbta/predictions_null_times.json` -- predictions with null arrival_time, valid departure_time.
- `test/fixtures/mbta/schedules_normal.json` -- schedules with future arrivals.
- `test/fixtures/mbta/schedules_empty.json` -- empty schedules data array.
- `test/fixtures/mbta/error_500.json` -- MBTA server error response.

## Running tests

```bash
# All automated tests (unit + integration + e2e)
pytest

# Just unit tests (fastest)
pytest test/test_selection.py test/test_config.py

# Just e2e tests
pytest -m e2e

# Include manual smoke tests
pytest -m smoke

# With coverage
pytest --cov=src --cov-report=term-missing
```

## Principles

- Every development cycle ends with a full `pytest` run (levels 1-3 all green).
- E2E tests prove the system works over real HTTP. They are not optional.
- No live MBTA calls in automated tests. Fake MBTA server with fixtures only.
- Selection logic is pure functions -- tested without HTTP or caching.
- Cache tests use controllable clocks (freezegun).
- E2E tests start real servers on random ports -- no port conflicts.
- Smoke tests are the only tests that touch real MBTA and they run on demand.
