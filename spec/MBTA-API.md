
# MBTA v3 API usage contract

This project uses the MBTA v3 API. Documentation: https://api-v3.mbta.com/

## API key

Strongly recommended. Without a key, the rate limit is 20 requests/min. With a key (free), the limit is 1000 requests/min. For a service polling 2 routes every 20 seconds, a key is effectively required.

Get a free key at: https://api-v3.mbta.com/

## Endpoints used

### Predictions: `GET /predictions`

Primary source (realtime data).

Filters:
- `filter[stop]` = stop_id
- `filter[route]` = route_id
- `filter[direction_id]` = 0 or 1
- `page[limit]` = 10
- `sort` = arrival_time

### Schedules: `GET /schedules`

Fallback source (static timetable).

Same filters as predictions.

## Timestamp resolution

Not all predictions include `arrival_time`. The service resolves timestamps in this order:

1. Use `attributes.arrival_time` if present.
2. Fall back to `attributes.departure_time` if arrival is null.
3. If both are null, discard the candidate.

The resolved timestamp is returned as `time` in the API response. Clients never see the arrival/departure distinction -- the service handles it internally.

## Fields consumed

Predictions:
- `data[].attributes.arrival_time`
- `data[].attributes.departure_time`
- `data[].relationships.trip.data.id`
- `data[].relationships.route.data.id`
- `data[].relationships.stop.data.id`

Schedules:
- `data[].attributes.arrival_time`
- `data[].attributes.departure_time`

## Time format

All MBTA time values are ISO 8601 timestamps with timezone offset (e.g. `2026-02-13T15:30:00-05:00`).

## Rate limiting

- Use caching (`cache_ttl`, default 20s) to avoid frequent MBTA calls.
- Each board item results in at most 1 predictions call per cache TTL.
- Schedule calls only happen when predictions yield no future arrivals.
- With 2 configured stops and 20s cache TTL, worst case is ~6 MBTA calls/min (well within the 1000/min authenticated limit).
