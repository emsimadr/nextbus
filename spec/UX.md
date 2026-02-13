
# UX: client display rules

This document defines how clients should display the API output. Most logic lives server-side.

## ESP32 e-ink display

Dedicated mini screen in a 3D-printed case. Shows one stop.

- Call `GET /v1/board/{key}` for the configured stop.
- Primary display: `leave_in_minutes` (when to leave the house).
- Secondary: `label` and `minutes` (bus ETA at stop).
- Show stale marker when `status` is `stale`.

Recommended layout:

```
Route 1 - Harvard Sq
Leave in: 2 min
Bus in:   6 min
```

Stale marker: append `*` or show a small icon when `status` is `stale`.

## TRMNL e-ink dashboard (future)

One tile among many on the TRMNL dashboard.

- A TRMNL plugin polls `GET /v1/board` and renders HTML markup.
- Show each stop's `label` and `leave_in_minutes`.
- Compact layout suitable for a dashboard tile.

## Home Assistant

- Sensor state: `arrival.leave_in_minutes` as an integer (minutes until you leave).
- Attributes:
  - `arrival.minutes` (bus ETA)
  - `arrival.time`
  - `arrival.source`
  - `status`
  - `label`
  - `alternatives`

## Display states

| Status       | ESP32 shows       | HA sensor state |
| ------------ | ----------------- | --------------- |
| `ok`         | `leave_in_minutes` | integer         |
| `no_service` | "No bus"           | `unknown`       |
| `stale`      | `leave_in_minutes*` | integer         |
| `error`      | "Err"              | `unknown`       |
| Loading      | "..." or last value | `unavailable`  |
