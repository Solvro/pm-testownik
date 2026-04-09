# ADR-003: AI calls proxied through the Django backend, not directly from the frontend

**Status:** Accepted  
**Date:** 2026-04-09  
**Author:** Antoni Czaplicki

---

## Context

AI features (contextual assistant, question generator) require communication with an external LLM API. The integration point had to be chosen: browser, Next.js API routes, or the Django backend.

## Considered Options

| Option | Description | Drawbacks |
|--------|-------------|-----------|
| A. Direct calls from the frontend | LLM API key in the browser | Secret exposure; no server-side rate limiting; no response sanitisation |
| **B. Django backend as proxy** | Frontend → Django → LLM API | Additional network hop |
| C. Next.js API Routes as proxy | Frontend → Next.js API → LLM API | Key in the Next.js environment; no shared rate limiting with the main API |

## Decision

**Option B** was chosen. The backend holds the LLM API key, builds the prompt from database data, sanitises the response (`nh3`), and applies rate limiting (DRF throttling). AI logic is isolated, independently testable, and covered by the same auth/permission stack as the rest of the API.

## Consequences

- The LLM API key is stored exclusively in backend environment variables (`.env`), never exposed to the client.
- AI endpoints: `POST /api/ai/hint/` and `POST /api/ai/generate/`.
- Async handling required: LLM calls can take > 5 s; `adrf` async views or Celery with polling is used (see ADR-004).
- Feature flag `AI_ENABLED` in Django Constance allows disabling the feature without redeployment (see ADR-006).
