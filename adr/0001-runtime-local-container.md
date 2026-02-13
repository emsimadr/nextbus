
# ADR 0001: Run the service as a local container first

## Context
We want a stable API for ESP32 and Home Assistant. We want quick iteration and low ops overhead.

## Options
1. Local container on LAN (Docker)
2. Home Assistant add-on
3. Cloud hosted service

## Decision
Choose option 1 for Iteration 01: local container on LAN.

## Consequences
- Simplifies development and keeps data local.
- External subscribers are not supported yet without additional network exposure.
- We can revisit cloud hosting later without changing the API contract.
