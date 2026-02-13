
# Backlog

## Iteration 01 (current)

- /v1/board and /v1/board/{key} endpoints with realtime-first policy
- Walk time computation (leave_in_minutes)
- Config loading from config.yaml
- Caching and stale mode with re-filtering
- Unit, integration, and E2E tests
- Docker support
- API key authentication (optional)

## Future

- TRMNL e-ink dashboard plugin (separate project consuming the API)
- ESP32 e-ink display firmware (separate project consuming the API)
- Publish Docker image to GitHub Container Registry
- Community documentation (finding your stop/route IDs)
- Structured JSON logging with request IDs
- Prometheus metrics endpoint
- Rate limiting on the service itself
- Config hot-reload without restart
