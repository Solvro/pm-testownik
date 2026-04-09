# Architecture Decision Records – Testownik, Stage 2

**Project:** Testownik | **Semester:** Summer 2026  
**Author:** Antoni Czaplicki

Each record follows the ADR format: context, considered options, decision, and consequences.

---

## ADR-001: Guest mode implemented as a full `AccountType.GUEST` user record

**Status:** Accepted  
**Date:** 2026-04-09

### Context

The platform requires unauthenticated users to be able to upload files, create quizzes, and save progress without registering an account. A mechanism for identifying a guest session had to be chosen.

### Considered Options

| Option | Description | Drawbacks |
|--------|-------------|-----------|
| A. Django sessions (`sessionid` cookie) | Session stored in Redis/DB; data tied to `sessionid` | Separate access-control logic; hard to migrate to a full account |
| B. LocalStorage / IndexedDB | No server session; data lives in the browser only | No cross-device sync; does not solve the persistence problem |
| **C. Guest account (`AccountType.GUEST`) + JWT** | Full `User` record with `account_type=GUEST`, no email; identified by JWT in an HttpOnly cookie | Requires periodic cleanup of expired accounts |

### Decision

**Option C** was chosen. The `User` model already has an `account_type` field with a `GUEST` value and a `create_guest_user()` method on `CustomUserManager`. This approach:
- reuses the existing authorisation infrastructure (same permission model),
- allows data to be preserved when upgrading to a full account,
- enables `QuizSession` / `AnswerRecord` storage without separate logic.

### Consequences

- A Celery periodic task is required to delete expired guest accounts (`account_type=GUEST`, no activity > 30 days).
- The `POST /api/auth/guest/` endpoint must be publicly accessible (`AllowAny`).
- The frontend must render a limited UI for guests (no email, no sharing features).

---

## ADR-002: Backend as the single source of truth – elimination of dual storage logic

**Status:** Accepted  
**Date:** 2026-04-09

### Context

Quiz progress was partly stored in `localStorage` (question lists, counters) and partly on the backend (`QuizSession`, `AnswerRecord`). This led to duplicated code, data desynchronisation, and difficult debugging.

### Considered Options

| Option | Description | Drawbacks |
|--------|-------------|-----------|
| A. Keep `localStorage` as the primary store | Fast offline access, no network requests | No cross-device sync; impossible to access from other devices; duplicated logic |
| B. Hybrid approach (localStorage + backend sync) | Offline-first with synchronisation | Complex implementation; state conflicts |
| **C. Backend as single source of truth** | `QuizSession` and `AnswerRecord` are the only state; frontend fetches via API | Requires network connectivity; slight latency |

### Decision

**Option C** was chosen for both authenticated users and guest accounts. The `QuizSession` and `AnswerRecord` models are complete and sufficient to reconstruct full session state. TanStack Query provides client-side caching, minimising the number of network requests.

### Consequences

- `localStorage` may remain as a read-only offline cache for question content, but not as a source of progress data.
- The API must provide a `GET /api/quizzes/<id>/session/` endpoint returning the full session state (current question, counters, study time).
- One-time migration: on first login after deployment, existing `localStorage` state may be synced to the backend.

---

## ADR-003: AI calls proxied through the Django backend, not directly from the frontend

**Status:** Accepted  
**Date:** 2026-04-09

### Context

AI features (assistant, question generator) require communication with an external LLM API. The integration point had to be chosen: frontend or backend.

### Considered Options

| Option | Description | Drawbacks |
|--------|-------------|-----------|
| A. Direct calls from the frontend | LLM API key in the browser | Secret exposure; no server-side rate limiting; no response sanitisation |
| **B. Django backend as proxy** | Frontend → Django → LLM API | Additional network hop |
| C. Next.js API Routes as proxy | Frontend → Next.js API → LLM API | Key in the Next.js environment; no shared rate limiting with the main API |

### Decision

**Option B** was chosen. The backend holds the LLM API key, builds the prompt from database data, sanitises the response (`nh3`), and applies rate limiting (DRF throttling). AI logic is isolated and independently testable.

### Consequences

- The LLM API key is stored exclusively in backend environment variables (`.env`).
- AI endpoints: `POST /api/ai/hint/` and `POST /api/ai/generate/`.
- Async handling required: LLM calls can take > 5 s; `adrf` (async views) or Celery with polling is used.
- Feature flag: `AI_ENABLED` in Django Constance allows disabling the feature without redeployment.

---

## ADR-004: Async DRF views (`adrf`) for interactive AI requests instead of Celery

**Status:** Accepted  
**Date:** 2026-04-09

### Context

LLM calls typically take 2–10 seconds. A mechanism was needed to handle these requests without blocking a worker thread.

### Considered Options

| Option | Description | Drawbacks |
|--------|-------------|-----------|
| A. Synchronous DRF views | Simplest | Blocks the thread; limited scalability |
| **B. Async views (`adrf`)** | `adrf` is already a dependency; `async def` views; `await` on LLM call | Requires ASGI server (Uvicorn / Daphne) instead of WSGI (Gunicorn) |
| C. Celery + polling | Background task; client polls for status | Additional complexity; extra status endpoints |

### Decision

**Option B** (`adrf`) was chosen for short interactive requests (assistant, typically < 10 s). The project already lists `adrf==0.1.12` in `requirements.txt`. For long-running tasks (generating many questions from large files), Celery may be adopted in the future.

### Consequences

- The application server must run in ASGI mode (e.g. Uvicorn or Daphne) instead of WSGI Gunicorn.
- AI views are `async def`; ORM access uses `sync_to_async` or Django's async ORM methods.

---

## ADR-005: AI response sanitisation using `nh3`

**Status:** Accepted  
**Date:** 2026-04-09

### Context

LLM responses may contain potentially dangerous HTML or code. These are rendered in the browser, creating an XSS risk.

### Decision

AI responses are sanitised by `nh3` (Rust-backed HTML sanitizer, already in `requirements.txt`) before being returned to the client. Disallowed tags and attributes are stripped. This applies to both the assistant and the question generator.

### Consequences

- Negligible performance overhead (Rust implementation).
- An allowlist of HTML tags must be defined (e.g. `<b>`, `<i>`, `<code>`, `<ul>`, `<li>`).
- Unit tests required for the sanitisation step.

---

## ADR-006: Feature flags managed via Django Constance

**Status:** Accepted  
**Date:** 2026-04-09

### Context

New features (AI, guest mode) may need to be disabled in production without a new deployment (e.g. during security audits or LLM API outages).

### Decision

Feature flags are managed by **Django Constance** (`django-constance>=4.3.2` in `requirements.txt`). Administrators can toggle features from Django Admin without restarting the server.

Example flags:
- `AI_ENABLED` (bool, default `False`)
- `GUEST_MODE_ENABLED` (bool, default `True`)

### Consequences

- AI views and guest endpoints check the flag at the start of request handling and return `503 Service Unavailable` when disabled.
- The frontend can query a `/api/config/` endpoint for feature availability and hide the corresponding UI accordingly.
