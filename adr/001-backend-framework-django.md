# ADR-001: Choice of Django as the Backend Framework

**Status:** Accepted  
**Date:** 2025-01-10
**Author:** Antoni Czaplicki

---

## Context

Testownik requires a robust backend capable of managing complex relational data (quizzes, folders, user progress) and offering a secure authentication system that handles various account roles (students, lecturers, admins, and guests).

## Considered Options

| Option                           | Description                                                                | Drawbacks                                                                |
|----------------------------------|----------------------------------------------------------------------------|--------------------------------------------------------------------------|
| A. Express.js (Node)             | Lightweight JavaScript backend framework                                   | Requires custom implementation of auth, ORM integration, and admin tools |
| B. NEXT.js + Prisma + PostgreSQL | Full-featured JavaScript framework with built-in ORM, Auth, and admin site | Monolithic nature can sometimes add overhead for simple endpoints        |
| **C. Django + DRF + PostgreSQL** | Full-featured Python framework with built-in ORM, Auth, and admin site     | Monolithic nature can sometimes add overhead for simple endpoints        |

## Decision

**Option C** was chosen. Django provides "batteries-included" capabilities:
- **Built-in Admin Panel:** Saves significant development time for managing users and platform data.
- **ORM:** The PostgreSQL integration perfectly models the deeply relational data (Folders -> Quizzes -> Questions -> Answers).
- **Authentication:** Django’s robust auth system, paired with DRF/SimpleJWT, perfectly handles our requirements for JSON tokens, guest accounts, and standard session logic.
- **Ecosystem:** Django has a rich ecosystem of plugins and tools for common web development tasks.
- **django-simple-history:** In the future we want to implement history of changes for quizzes, folders, questions and answers, django-simple-history is a great plugin that provides this functionality.

## Consequences

- The backend team must adhere to Django's structure and MVC patterns.
- Business logic is firmly centralized within Django apps (`users`, `quizzes`, `uploads`).
- Interaction with the DB utilizes Python’s synchronous ORM for the majority of the operations.
