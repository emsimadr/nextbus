
# ADR 0003: Optional shared API key authentication

## Context
The API may be exposed beyond localhost. We want a simple auth scheme compatible with ESP32 and Home Assistant.

## Options
1. No auth on LAN
2. Shared API key in header
3. OAuth and user accounts

## Decision
Choose option 2 as optional. If API_KEY env var is set, require X-API-Key header.

## Consequences
- Simple and works with Home Assistant headers and ESP32.
- Not multi-tenant. Rotation is manual.
- If we go cloud and need real users, we will revisit.
