# ADR-002: Backend as the single source of truth – elimination of dual storage logic

**Status:** Accepted  
**Date:** 2026-04-09  
**Author:** Antoni Czaplicki

---

## Context

Quiz progress was partly stored in `localStorage` (question lists, counters) and partly on the backend (`QuizSession`, `AnswerRecord`). This led to duplicated code, data desynchronisation, and difficult debugging. The existing `QuizSession` model tracks `current_question`, `study_time`, and `is_active`; `AnswerRecord` tracks every submitted answer. These models are complete enough to reconstruct full session state.

## Considered Options

| Option | Description | Drawbacks |
|--------|-------------|-----------|
| A. Keep `localStorage` as the primary store | Fast offline access, no network requests | No cross-device sync; impossible to access from other devices; duplicated logic |
| B. Hybrid approach (localStorage + backend sync) | Offline-first with synchronisation | Complex implementation; state conflicts |
| **C. Backend as single source of truth** | `QuizSession` and `AnswerRecord` are the only state; frontend fetches via API | Requires network connectivity; slight latency |

## Decision

**Option C** was chosen for both authenticated users and guest accounts. TanStack Query provides client-side caching, minimising the number of network requests and making the transition transparent to the user.

## Consequences

- `localStorage` may remain as a read-only offline cache for question content, but not as a source of progress data.
- The API provides `GET /api/quizzes/<id>/` with `?include=current_session` returning full session state (current question, counters, study time).
- One-time migration: on first login after deployment, existing `localStorage` state may be synced to the backend via `useGuestQuizMigration`.
- `buildFallbackSession` in `quiz.service.ts` handles the case where no server session exists yet.
