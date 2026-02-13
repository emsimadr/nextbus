
# Data model

## Core types (language agnostic)

### BoardItemConfig

Source: `config.yaml` stops list.

- key: string (unique identifier, e.g. `"route_1_inbound"`)
- label: string (human-readable display name, e.g. `"Route 1 - Harvard Sq"`)
- route_id: string (MBTA route ID)
- stop_id: string (MBTA stop ID)
- direction_id: int (0 or 1)
- walk_minutes: int (minutes to walk from house to stop, default 0)

### Arrival

A single upcoming bus arrival.

- time: ISO 8601 timestamp (when the bus arrives at the stop)
- minutes: int (minutes until bus arrives at stop, always non-negative)
- leave_in_minutes: int (minutes until you must leave the house = `minutes - walk_minutes`; can be negative if too late to catch on foot)
- source: enum { realtime, schedule }
- trip_id: optional string (MBTA trip ID, kept for debugging/tracing)

### BoardItemResponse

One configured stop with its next arrival.

- key: string
- label: string
- route_id: string
- stop_id: string
- direction_id: int
- walk_minutes: int
- status: enum { ok, no_service, stale, error }
- arrival: optional Arrival (the next bus)
- alternatives: list of Arrival (up to 2 additional upcoming arrivals, always present, possibly empty)
- stale_as_of: optional timestamp string (only present when status is stale)
- error: optional ErrorDetail (only present when status is error)

### ErrorDetail

- code: string (one of: `mbta_unreachable`, `mbta_rate_limited`, `unknown`)
- message: string (human-readable description)

### BoardResponse

Top-level response for `/v1/board`.

- as_of: ISO 8601 timestamp (when this response was generated)
- items: list of BoardItemResponse

## Invariants

- If status is `ok`, arrival must be present.
- If status is `no_service`, arrival is null, alternatives is empty.
- If status is `stale`, cached arrivals are re-filtered against current time. If all cached arrivals are now past, status becomes `no_service` instead of `stale`.
- If status is `error`, error object must be present with a defined code.
- `minutes` is always non-negative (past arrivals are filtered out before response).
- `leave_in_minutes` can be negative (bus is arriving but you can't walk there in time).
- `alternatives` is always a list (empty, never null).
- On cold start with no cache and MBTA failure, status is `error` with code `mbta_unreachable`.
