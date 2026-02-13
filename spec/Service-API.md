
# Service API contract (v1)

Base path: `/v1`

## Authentication

Optional shared API key:

- Header: `X-API-Key: <token>`
- If `api_key` is not set in config, auth is disabled.
- `/health` is always unauthenticated.

## Endpoints

### GET /v1/board

Returns all configured stops with their next arrivals.

Response 200:

```json
{
  "as_of": "2026-02-13T12:34:56-05:00",
  "items": [
    {
      "key": "route_1_inbound",
      "label": "Route 1 - Harvard Sq",
      "route_id": "1",
      "stop_id": "place-harsq",
      "direction_id": 0,
      "walk_minutes": 4,
      "status": "ok",
      "arrival": {
        "time": "2026-02-13T12:41:00-05:00",
        "minutes": 6,
        "leave_in_minutes": 2,
        "source": "realtime",
        "trip_id": "trip-001"
      },
      "alternatives": [
        {
          "time": "2026-02-13T12:53:00-05:00",
          "minutes": 18,
          "leave_in_minutes": 14,
          "source": "realtime",
          "trip_id": "trip-002"
        }
      ]
    }
  ]
}
```

### GET /v1/board/{key}

Returns a single configured stop by its config key.

Response 200: same as one item from the board response (a `BoardItemResponse`).

Response 404 (unknown key):

```json
{
  "detail": "Board item 'unknown_key' not found"
}
```

### GET /health

Response 200:

```json
{
  "status": "healthy"
}
```

Always returns 200. No authentication required.

## Status values

| Status       | Meaning                                        |
| ------------ | ---------------------------------------------- |
| `ok`         | Fresh data from MBTA, arrival is present       |
| `no_service` | No upcoming buses found                        |
| `stale`      | Serving cached data, MBTA is unreachable       |
| `error`      | Cannot serve data, see error object             |

## Source values

| Source       | Meaning                          |
| ------------ | -------------------------------- |
| `realtime`   | MBTA realtime prediction         |
| `schedule`   | MBTA static schedule (fallback)  |

## Error response shape

When `status` is `error`:

```json
{
  "key": "route_1_inbound",
  "label": "Route 1 - Harvard Sq",
  "route_id": "1",
  "stop_id": "place-harsq",
  "direction_id": 0,
  "walk_minutes": 4,
  "status": "error",
  "arrival": null,
  "alternatives": [],
  "error": {
    "code": "mbta_unreachable",
    "message": "Upstream MBTA API request failed"
  }
}
```

Error codes: `mbta_unreachable`, `mbta_rate_limited`, `unknown`.

## Versioning rules

- No breaking changes within `/v1`.
- Additive fields are allowed.
- Breaking changes require `/v2`.
