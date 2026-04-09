# Architecture – Testownik (Current State and Planned Changes, Stage 2)

**Project:** Testownik | **Semester:** Summer 2026  
**Author:** Antoni Czaplicki

---

## 1. Context

Testownik is a quiz-based learning and knowledge verification platform.  
The project consists of three repositories:

| Repository | Technology | Role |
|---|---|---|
| `Solvro/web-testownik` | TypeScript, Next.js 15, React, TanStack Query | Web application (SPA/SSR) |
| `Solvro/backend-testownik` | Python 3.13, Django 6, DRF, PostgreSQL, Celery | REST API, business logic |
| `Solvro/pm-testownik` | Markdown | Project management, documentation |

The application is available at <https://testownik.solvro.pl/>.

---

## 2. Backend Architecture (`backend-testownik`)

### 2.1 Django Application Structure

```
backend-testownik/
├── testownik_core/   # project configuration, settings, URL router
├── users/            # user models, authentication, USOS OAuth
├── quizzes/          # quizzes, questions, answers, sessions
├── uploads/          # file handling (Pillow, S3/django-storages)
├── grades/           # grades and results
├── feedback/         # question error reports
├── alerts/           # alerts and notifications
└── maintenance/      # maintenance mode
```

### 2.2 Key Data Models

#### Users (`users`)

```
User
├── id (UUID, PK)
├── email (nullable – guest accounts have no email)
├── account_type: GUEST | EMAIL | STUDENT | LECTURER
├── account_level: BASIC | GOLD
├── root_folder (FK → Folder)
└── settings (1:1 → UserSettings)

AccountType.GUEST   →  anonymous account, created automatically
AccountType.EMAIL   →  email + OTP registration
AccountType.STUDENT →  verified via USOS
```

#### Quizzes and Questions (`quizzes`)

```
Folder ──< Quiz ──< Question ──< Answer
                └─< QuizSession ──< AnswerRecord
                └─< SharedQuiz
           └─< SharedFolder
```

- **Quiz**: `id`, `title`, `visibility` (0–3), `allow_anonymous`, `folder (FK)`
- **Question**: `question_type` (CLOSED=0, OPEN=1, TRUE_FALSE=2), `is_flashcard`, `is_markdown_enabled`
- **QuizSession**: current session state – `current_question`, `study_time`, `is_active`
- **AnswerRecord**: answer history – `selected_answers (JSON)`, `was_correct`

### 2.3 Authentication

- Authenticated users: JWT in HttpOnly cookie (`djangorestframework-simplejwt` + `users/auth_cookies.py`).
- Guests: server session backed by a UUID-identified anonymous `User` record (`AccountType.GUEST`); identified by a JWT cookie.
- USOS OAuth: `Authlib` + `usos-api`.

### 2.4 Asynchronous Tasks

Celery (with Redis or RabbitMQ broker) handles:
- expiry and cleanup of inactive guest accounts,
- email delivery (OTP, notifications),
- (planned) background AI calls.

---

## 3. Frontend Architecture (`web-testownik`)

### 3.1 Source Structure

```
src/
├── app/              # Next.js App Router – routes (page.tsx + client.tsx)
├── components/       # React components (shadcn/ui, Radix UI)
├── services/         # API communication layer
│   ├── base-api.service.ts   # HTTP client (fetch wrapper)
│   ├── quiz.service.ts       # quiz CRUD, sessions
│   ├── user.service.ts       # user data
│   └── image.service.ts      # image upload
├── hooks/            # custom React hooks (TanStack Query)
├── lib/              # utility helpers
├── types/            # TypeScript types
├── app-context.ts    # application context (auth, services)
└── proxy.ts          # Next.js → backend API proxy
```

### 3.2 Page Pattern: SSR + Client

Each route has two files:
- `page.tsx` – Server Component (SSR, initial data fetch),
- `client.tsx` – Client Component (interactivity, TanStack Query).

### 3.3 Backend Communication

- Requests to `/api/*` are proxied through Next.js Middleware (`proxy.ts`) to Django.
- Authentication via JWT cookie – no need to store tokens client-side.
- Server state managed by **TanStack Query** (cache, refetch, optimistic updates).

---

## 4. Integration Points (Course Scope)

### 4.1 Guest Mode

```
Browser                      Next.js (SSR)            Django API
───────                      ─────────────            ──────────
Unauthenticated visit
    │                              │                       │
    │──── GET / ──────────────────>│                       │
    │<─── HTML (home page) ────────│                       │
    │                              │                       │
    │─── POST /api/auth/guest/ ───────────────────────────>│
    │                              │   create_guest_user() │
    │<─── JWT cookie (guest) ──────────────────────────────│
    │                              │                       │
    │─── POST /api/uploads/ ──────────────────────────────>│  (file upload)
    │─── POST /api/quizzes/ ──────────────────────────────>│  (quiz creation)
    │─── GET  /api/quizzes/<id>/session/ ─────────────────>│  (session fetch)
    │─── POST /api/quizzes/<id>/session/answer/ ──────────>│  (answer record)
```

**Backend:** `users/views.py` – `POST /api/auth/guest/` creates a `User` with `account_type=GUEST` and returns a JWT.  
**Frontend:** `app-context-provider.tsx` – when no token is present, initiates a guest creation request; the cookie stores the JWT.

### 4.2 Backend-Only Persistence

Current state (to be changed): part of the quiz flow stores data in `localStorage` (offline questions, progress counters).

Target state:
- `QuizSession` is the single source of truth for the active session.
- `AnswerRecord` records each answer (analytics, session resumption).
- The frontend fetches the session via `GET /api/quizzes/<id>/session/` on every quiz view mount.
- No writes to `localStorage` for data available through the API.

### 4.3 AI Integration

```
Frontend                    Backend                   ML Service (external/internal)
────────                    ───────                   ──────────────────────────────
POST /api/ai/hint/          AiHintView                LLM API
  { question_id }      ──>  fetch question context ──> call LLM API
                             <── hint text ────────────<──

POST /api/ai/generate/      AiGenerateView            LLM API
  { file / text }      ──>  parse content ───────────> generate questions
                             <── question list for review <──
```

**Rate limiting:** DRF throttling per user on AI endpoints.  
**Feature flag:** Django Constance (`AI_ENABLED`, `AI_HINT_ENABLED`, `AI_GENERATE_ENABLED`).  
**Async:** LLM calls handled by `adrf` (async DRF views) or Celery tasks.

---

## 5. Guest Mode Approach

### Chosen variant: server-side guest account (JWT)

Instead of browser sessions (`sessionStorage` / `localStorage`) or Django sessions, a guest account is a full `User` record with `account_type=GUEST`:

**Advantages:**
- Uniform access control logic (same permission model as for authenticated users).
- Quiz progress stored in `QuizSession` / `AnswerRecord` (full analytics).
- Guest account can be upgraded to a full account while preserving data.
- No duplication of backend / frontend code.

**Constraints:**
- Guest accounts require periodic cleanup (Celery task).
- Guests cannot use features that require an email address (sharing, notifications).

---

## 6. Backend-Only Persistence Approach

### Current problem

The frontend stores quiz state in `localStorage`:
- list of questions to repeat,
- answer counters,
- current progress.

This results in duplicated business logic and no synchronisation across devices.

### Target solution

| Data | Storage |
|------|---------|
| Active quiz session | `QuizSession` (DB) |
| Answer history | `AnswerRecord` (DB) |
| Current question | `QuizSession.current_question` (FK) |
| Study time | `QuizSession.study_time` (DurationField) |
| Configuration (repetition thresholds) | `UserSettings` (DB) |

The frontend reads state via `GET /api/quizzes/<id>/session/` on every quiz view entry. Updates are sent through `PATCH` / `POST` to the same endpoint.

---

## 7. AI Integration Overview

### 7.1 Contextual Assistant

- **Input:** `question_id`, optionally `context` (answers selected by the user so far).
- **Processing (backend):** fetch question text and answers from DB → build prompt → call LLM API → sanitise response (`nh3`).
- **Output:** hint text (does not contain the correct answer directly).

### 7.2 Question Generator from Notes

- **Input:** file (PDF, TXT, MD) or plain text.
- **Processing (backend):** parse file → extract text → build prompt → call LLM API → parse question structure.
- **Output:** list of `{ text, answers: [{text, is_correct}] }` objects for user review before saving.

### 7.3 AI Security

- LLM responses sanitised by `nh3` before being returned to the client.
- Prompt injection mitigated by separating user-supplied context from system instructions.
- Rate limiting: 10 requests/minute per user (configurable via Constance).

---

## 8. Testing and Observability

| Layer | Tool | Scope |
|-------|------|-------|
| Backend (unit / integration) | `pytest-django` | models, services, API endpoints |
| Frontend (unit) | `vitest` | services, hooks, components |
| E2E | Playwright (planned) | guest mode end-to-end, session persistence |
| Application logs | Django logging → stdout → Sentry (planned) | runtime errors |
| Metrics | Django Admin + Celery Flower | queues, async tasks |

---

## 9. Security (Course Scope)

| Aspect | Mechanism |
|--------|-----------|
| Authentication | JWT in HttpOnly cookie; `djangorestframework-simplejwt` |
| Authorisation | Custom DRF permissions (`quizzes/permissions.py`); `account_type` checks |
| File uploads | Extension and MIME type validation; storage on S3 via `django-storages` |
| HTML sanitisation | `nh3` (Rust-backed) for AI-generated content |
| Rate limiting | `django-ratelimit` + DRF throttling |
| CORS | `django-cors-headers`; domain whitelist |
| Guest accounts | Expire after 30 days of inactivity; no access to other users' resources |
