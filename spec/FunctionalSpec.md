
# Functional spec: Board service

## Endpoints

### GET /v1/board

Returns all configured stops with their next arrivals.

Response: `BoardResponse` (see DataModel.md)

### GET /v1/board/{key}

Returns a single configured stop by its config key.

Response: `BoardItemResponse` (see DataModel.md)

If the key does not match any configured stop, return HTTP 404.

### GET /health

Returns `{"status": "healthy"}` with HTTP 200. Used for Docker health checks and Home Assistant monitoring.

## Inputs

Each board item is identified by its config:

- route_id (MBTA route)
- stop_id (MBTA stop)
- direction_id (0 or 1)
- walk_minutes (minutes to walk from house to stop)

All stops are pre-configured in `config.yaml`. There are no ad-hoc query parameters.

## Realtime-first policy

Order of preference for each board item:

1. MBTA predictions for the target route, stop, direction
2. MBTA schedules for the same route, stop, direction
3. If neither yields a future arrival, return `status: no_service`

## Selection rules

Given a list of candidate arrivals (from predictions or schedules):

1. Resolve timestamp: use `arrival_time` as primary; fall back to `departure_time` if arrival is null.
2. Discard candidates without any usable timestamp.
3. Discard candidates where the resolved timestamp is in the past (relative to `as_of`).
4. Sort remaining by timestamp ascending.
5. The first is `arrival` (the next bus).
6. The next 2 (if they exist) are `alternatives`.

## Minutes computation

- `minutes` = floor((arrival_time - as_of) in seconds / 60)
- If the result is negative after flooring, the arrival is in the past -- discard it.

## Leave-in-minutes computation

- `leave_in_minutes` = `minutes` - `walk_minutes`
- Can be negative. Negative means the bus is coming but you cannot walk to the stop in time.
- This is the primary "action number" for clients: positive means you still have time to leave.

## Stale mode

If MBTA requests fail for a board item:

- If a cached successful response exists and is younger than `stale_max_age` (from config):
  - Re-filter cached arrivals against current time (discard any that are now past).
  - Recompute `minutes` and `leave_in_minutes` against current time.
  - If at least one arrival survives, return with `status: stale`, `stale_as_of` set to the original cache time.
  - If no arrivals survive (all are now past), return `status: no_service`.
- If no usable cache exists:
  - Return `status: error` with error code `mbta_unreachable`.

## Caching policy

- Cache by key: the board item `key` (which maps to a unique route/stop/direction).
- TTL for fresh cache: `cache_ttl` seconds (default 20).
- Grace period for stale: `stale_max_age` seconds (default 300).

## Edge cases

- Terminal stop: direction still matters. MBTA direction_id is honored.
- If multiple predictions share identical timestamps, pick any (no tiebreaker needed).
- If MBTA returns null for predicted time but includes scheduled time in the predictions feed, treat the candidate as having no prediction timestamp and let it fall through to the schedules call.
- Cold start: cache is empty. First request triggers MBTA fetch. If that fetch fails, return `error`.

## Authentication

If `api_key` is set in config (or `API_KEY` env var), all `/v1/*` requests must include `X-API-Key` header matching the configured value. `/health` is always unauthenticated.

## Acceptance criteria

- When predictions contain at least one future predicted time, return the earliest as `source: realtime`.
- When predictions are empty or missing times, and schedules contain a future time, return the earliest as `source: schedule`.
- When both are empty, return `status: no_service`.
- When MBTA errors, return `stale` if cached data survives re-filtering, else `error`.
- `leave_in_minutes` is correctly computed as `minutes - walk_minutes`.
- `alternatives` contains up to 2 additional arrivals after the next one.
- `/v1/board/{key}` returns 404 for unknown keys.
- `/health` returns 200 unconditionally.
