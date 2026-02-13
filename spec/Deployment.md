
# Deployment spec

## Target runtime

A small HTTP service that runs as a Docker container on the home network.

Primary: `docker compose up -d` on any machine (Raspberry Pi, NAS, old laptop, etc.)

## Community distribution

The Docker image is published to GitHub Container Registry. Setup for a new user:

1. Clone the repo (or just copy `docker-compose.yml` and `config.example.yaml`).
2. Copy `config.example.yaml` to `config.yaml` and edit with your stops.
3. Run `docker compose up -d`.

## Configuration

### config.yaml (primary)

Mounted into the container at `/app/config.yaml`. See `config.example.yaml` for the full template.

Key fields:

- `mbta_api_key`: MBTA v3 API key (strongly recommended; unauthenticated = 20 req/min limit)
- `cache_ttl`: seconds between MBTA refreshes (default 20)
- `stale_max_age`: seconds to serve stale data when MBTA is down (default 300)
- `api_key`: optional; if set, all `/v1/*` requests require `X-API-Key` header
- `stops`: list of BoardItemConfig objects (key, label, route_id, stop_id, direction_id, walk_minutes)

### Environment variables (overrides)

Useful for Docker secrets or quick tweaks without editing the config file.

- `MBTA_API_KEY` -- overrides `mbta_api_key` in config
- `API_KEY` -- overrides `api_key` in config
- `PORT` -- HTTP listen port (default `8080`)
- `LOG_LEVEL` -- logging level (default `info`)
- `CONFIG_PATH` -- path to config file (default `/app/config.yaml`)

## Health check

`GET /health` returns `{"status": "healthy"}` with HTTP 200. Always unauthenticated.

Use in `docker-compose.yml`:

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
  interval: 30s
  timeout: 5s
  retries: 3
```

## Observability

- Structured logs in JSON format.
- Log fields: request_id, route_id, stop_id, direction_id, status, cache_hit.
- No secrets in logs.
- Log level controlled by `LOG_LEVEL` env var.
