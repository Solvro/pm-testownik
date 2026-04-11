# ADR-004: Guest mode implemented as a full `AccountType.GUEST` user record

**Status:** Accepted  
**Date:** 2026-03-02  
**Author:** Antoni Czaplicki

---

## Context

The platform requires unauthenticated users to be able to create quizzes, and save progress without registering an account. Until now, the frontend has been using `localStorage` to store quiz progress and quizzes for guests. However, this approach has limitations:

## Considered Options

| Option                                                 | Description                                                                                      | Drawbacks                                                                                                       |
|--------------------------------------------------------|--------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| A. LocalStorage (current implementation) / IndexedDB   | No server session; data lives in the browser only, enchanced privacy for guests                  | Separate logic for handling guest data; complex migrations of quiz schema; no option to eaisly store statistics |
| **B. Server side guest account (`AccountType.GUEST`)** | Full `User` record with `account_type=GUEST`, no email; identified by JWT token stored on client | Requires periodic cleanup of expired accounts, guest data gets sent to the server |
## Decision

**Option B** was chosen. Added `account_type` field with a `GUEST` value  `User` model. This approach:
- reuses the existing authorisation infrastructure (same permission model),
- allows data to be easily preserved when upgrading to a full account,
- enables `Quiz` / `QuizSession` / `AnswerRecord` storage without separate logic.

## Consequences

- A periodic task is required to delete expired guest accounts (`account_type=GUEST`, no activity > 30 days).
- The `POST /api/guest/create/` endpoint must be accessible to unauthenticated users only allowed through Next.js server side.
- The frontend must render a limited UI for guests.
- The frontend should automatically trigger guest creation on first visit (except of excluded routes like `/`).
