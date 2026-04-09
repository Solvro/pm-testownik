# ADR-006: Feature flags managed via Django Constance

**Status:** Accepted  
**Date:** 2026-04-09  
**Author:** Antoni Czaplicki

---

## Context

New features (AI assistant, guest mode, maintenance mode) may need to be toggled in production without a new deployment — for example during security audits, LLM API outages, or gradual rollouts. The project already uses `django-constance>=4.3.2` with a `DatabaseBackend` and has an existing flag (`MAINTENANCE_MODE`) as precedent.

## Considered Options

| Option | Description | Drawbacks |
|--------|-------------|-----------|
| A. Environment variables | Simple; no runtime changes | Requires restart to toggle; no admin UI |
| B. Database-backed settings table (custom) | Full control | Reinvents what Constance already provides |
| **C. Django Constance** | Already in use; admin UI; no restart needed | Flags are global (not per-user); adds a DB read per flagged request |
| D. External feature flag service (LaunchDarkly, Unleash) | Per-user targeting, rich rollout controls | External dependency; overkill for current scale |

## Decision

**Option C** was chosen. Constance is already installed and configured with `DatabaseBackend`. Administrators can toggle flags instantly from Django Admin without restarting the server, matching the existing `MAINTENANCE_MODE` pattern.

Example flags:
- `MAINTENANCE_MODE` (bool, default `False`) — existing flag
- `AI_ENABLED` (bool, default `False`) — gates all AI endpoints
- `GUEST_MODE_ENABLED` (bool, default `True`) — gates guest account creation

## Consequences

- Each flagged view reads its Constance value at request time; disabled endpoints return `503 Service Unavailable`.
- The frontend can query a `/api/config/` endpoint to retrieve active flags and conditionally hide UI elements.
- Flag values are cached by Constance to minimise database overhead.
