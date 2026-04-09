# Requirements – Testownik (Course Scope, Stage 2)

**Project:** Testownik | **Semester:** Summer 2026  
**Author:** Antoni Czaplicki

---

## 1. Purpose and Scope

This document describes requirements for the three technical areas implemented as part of the course:

1. **Guest mode** – server-side, based on anonymous user accounts.
2. **Backend-only persistence** – unified server-side data storage.
3. **AI integration** – contextual assistant and automatic question generation.

The document does not cover requirements for other project features (new design, PWA, flashcards, LaTeX, statistics) delivered by other team members.

---

## 2. Actors

| Actor | Description |
|-------|-------------|
| **Guest** | Unauthenticated user with a server-side anonymous session (`AccountType.GUEST`) |
| **Authenticated user** | User with an email or USOS account (`AccountType.EMAIL` / `AccountType.STUDENT`) |
| **AI system** | External or internal ML service providing hints and generating questions |
| **Administrator** | Staff user managing the platform via Django Admin |

---

## 3. Functional Requirements

### 3.1 Guest Mode

| ID | Requirement |
|----|-------------|
| FG-01 | The system automatically creates an anonymous guest account (`AccountType.GUEST`) on the user's first visit without logging in. |
| FG-02 | A guest can browse public quizzes. |
| FG-03 | A guest can upload question files and create a quiz from the uploaded data. |
| FG-04 | A guest's session progress (`QuizSession`, `AnswerRecord`) is stored server-side and remains accessible for the duration of the anonymous session. |
| FG-05 | A guest can optionally upgrade their account to a full account, retaining all existing data. |
| FG-06 | The frontend displays an appropriate UI for guest mode (limited options, call-to-action to register). |

### 3.2 Backend-Only Persistence

| ID | Requirement |
|----|-------------|
| FP-01 | Quiz progress (current state, correct/incorrect answer counts) is stored exclusively in `QuizSession` / `AnswerRecord` on the server. |
| FP-02 | The frontend retrieves session state from the backend (TanStack Query) and does not duplicate data in `localStorage` / `sessionStorage` for the supported scenarios. |
| FP-03 | Progress is saved through the existing API (`/api/quizzes/<id>/session/`). |
| FP-04 | A one-time migration of existing `localStorage` state to the backend occurs on first login after deployment (where applicable). |

### 3.3 AI Integration

| ID | Requirement |
|----|-------------|
| FA-01 | A user can activate the AI assistant during a quiz to receive a contextual hint for the current question. |
| FA-02 | The assistant does not reveal the correct answer directly – it provides a clue or explanation. |
| FA-03 | A user can upload a notes file (PDF / TXT / Markdown) and initiate automatic question set generation. |
| FA-04 | Generated questions are presented to the user for review before being saved to the database. |
| FA-05 | AI calls are subject to rate limiting to control costs and prevent abuse. |
| FA-06 | AI features are optional and can be disabled by an administrator (Django Constance / feature flag) without redeployment. |

---

## 4. Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NF-01 | Consistency with the existing architecture: Django 6 + DRF backend, Next.js (App Router) + TypeScript frontend. |
| NF-02 | No regression in existing user flows (logged-in users, quiz sharing, statistics). |
| NF-03 | API response time (guest mode, session save) < 500 ms for typical requests. |
| NF-04 | Guest accounts expire after 30 days of inactivity (Celery periodic task). |
| NF-05 | AI endpoint calls are limited to a defined per-user request rate (DRF throttling). |
| NF-06 | AI calls do not block the main request thread (async view or Celery task). |
| NF-07 | Individual contributions are clearly identifiable in the commit and pull request history. |

---

## 5. Acceptance Criteria (General)

1. Guest mode works end-to-end: entry → file upload → quiz → result saved server-side.
2. Authenticated users do not rely on `localStorage` for quiz progress storage within the backend-managed scenarios.
3. The AI assistant responds to contextual queries; the generator produces questions from an uploaded file.
4. Documentation (`architecture.md`, `adr.md`) is consistent with the implementation.
