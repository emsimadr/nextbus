
# Iteration 01: Board API with realtime-first policy

## Goal

Deliver a working `/v1/board` endpoint that returns the next arrival for all configured stops.
Walk-time aware (`leave_in_minutes`). Safe to poll from Home Assistant and easy for ESP32 to parse.

## In scope

- Implement `GET /v1/board` (all stops)
- Implement `GET /v1/board/{key}` (single stop)
- Implement `GET /health`
- Realtime predictions first, schedule fallback
- Cache with TTL and stale mode (with re-filtering)
- Walk time computation (`leave_in_minutes`)
- Config loading from `config.yaml`
- Up to 2 alternatives per stop
- API key authentication (optional)
- Unit tests for selection logic and config parsing
- Integration tests for board service and API endpoints
- E2E tests with fake MBTA server
- Docker support (Dockerfile, docker-compose.yml)

## Out of scope

- TRMNL plugin
- ESP32 firmware
- Any web UI
- Nearby stop discovery
- User accounts or multi-tenant auth
- Push notifications

## Acceptance criteria

- `/v1/board` returns status `ok` with `source: realtime` when future predictions exist.
- `/v1/board` returns status `ok` with `source: schedule` when no realtime exists but schedule does.
- `/v1/board` returns status `no_service` when neither exists.
- `/v1/board` returns status `stale` when MBTA fails but cached data survives re-filtering.
- `/v1/board` returns status `error` when MBTA fails and no cache is usable.
- `leave_in_minutes` equals `minutes - walk_minutes` for every arrival.
- `alternatives` contains up to 2 additional arrivals.
- `/v1/board/{key}` returns 404 for unknown keys.
- `/health` returns 200 unconditionally.
- All automated tests pass (unit + integration + E2E).
- Service runs via `docker compose up`.
