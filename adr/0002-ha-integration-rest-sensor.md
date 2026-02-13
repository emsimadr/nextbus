
# ADR 0002: Home Assistant uses REST sensor polling

## Context
Home Assistant should read next arrival data for dashboards and automations with minimal work.

## Options
1. REST sensor polling JSON
2. Custom Home Assistant integration
3. MQTT push

## Decision
Choose option 1 for Iteration 01: REST sensor polling.

## Consequences
- Fastest path with minimal code.
- Polling frequency needs server side caching.
- MQTT or custom integration can be added later if needed.
