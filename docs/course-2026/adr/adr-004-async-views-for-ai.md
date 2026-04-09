# ADR-004: Async DRF views (`adrf`) for interactive AI requests instead of Celery

**Status:** Accepted  
**Date:** 2026-04-09  
**Author:** Antoni Czaplicki

---

## Context

LLM calls typically take 2–10 seconds. A mechanism was needed to handle these requests without blocking a WSGI worker thread, given that `adrf==0.1.12` is already listed in `requirements.txt` and the project has an ASGI entry point (`testownik_core/asgi.py`).

## Considered Options

| Option | Description | Drawbacks |
|--------|-------------|-----------|
| A. Synchronous DRF views | Simplest implementation | Blocks the thread; limits concurrent request capacity |
| **B. Async views (`adrf`)** | `async def` views; `await` on LLM call; no thread blocking | Requires ASGI server (Uvicorn / Daphne) instead of WSGI Gunicorn |
| C. Celery + polling | Background task; client polls a status endpoint | Additional complexity; extra endpoints; worse UX for short requests |

## Decision

**Option B** (`adrf`) was chosen for short interactive requests (assistant, typically < 10 s). The project already includes `adrf` and an ASGI configuration. For long-running tasks (bulk question generation from large files), Celery may be adopted in the future.

## Consequences

- The application server must run in ASGI mode (e.g. Uvicorn or Daphne) rather than WSGI Gunicorn.
- AI views use `async def`; Django ORM access uses `sync_to_async` or the built-in async ORM methods available in Django 6.
- Non-AI views are unaffected and continue to work under both ASGI and WSGI.
