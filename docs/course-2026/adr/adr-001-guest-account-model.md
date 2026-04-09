# ADR-001: Guest mode implemented as a full `AccountType.GUEST` user record

**Status:** Accepted  
**Date:** 2026-04-09  
**Author:** Antoni Czaplicki

---

## Context

The platform requires unauthenticated users to be able to upload files, create quizzes, and save progress without registering an account. A mechanism for identifying a guest session had to be chosen.

## Considered Options

| Option | Description | Drawbacks |
|--------|-------------|-----------|
| A. Django sessions (`sessionid` cookie) | Session stored in Redis/DB; data tied to `sessionid` | Separate access-control logic; hard to migrate to a full account |
| B. LocalStorage / IndexedDB | No server session; data lives in the browser only | No cross-device sync; does not solve the persistence problem |
| **C. Guest account (`AccountType.GUEST`) + JWT** | Full `User` record with `account_type=GUEST`, no email; identified by JWT in an HttpOnly cookie | Requires periodic cleanup of expired accounts |

## Decision

**Option C** was chosen. The `User` model already has an `account_type` field with a `GUEST` value and a `create_guest_user()` method on `CustomUserManager`. This approach:
- reuses the existing authorisation infrastructure (same permission model),
- allows data to be preserved when upgrading to a full account,
- enables `QuizSession` / `AnswerRecord` storage without separate logic.

## Consequences

- A Celery periodic task is required to delete expired guest accounts (`account_type=GUEST`, no activity > 30 days).
- The `POST /api/guest/create/` endpoint must be publicly accessible (`AllowAny`).
- The frontend must render a limited UI for guests (no email, no sharing features).
- The `useAutoGuest` hook in the frontend automatically triggers guest creation on first unauthenticated visit.
