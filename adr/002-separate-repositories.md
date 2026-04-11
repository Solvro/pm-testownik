# ADR-002: Separate Frontend and Backend Repositories

**Status:** Accepted  
**Date:** 2025-01-10  
**Author:** Antoni Czaplicki

---

## Context

With Django chosen as the backend and a modern React framework (Next.js) for the frontend, we needed to decide how to structure the source code across the organization for future scalability and ease of development.

## Considered Options

| Option                                  | Description                                                       | Drawbacks                                                              |
|-----------------------------------------|-------------------------------------------------------------------|------------------------------------------------------------------------|
| A. Monocore (Django templates)          | Frontend rendered completely inside Django templates              | Severely limits UI interactivity and modernization                     |
| B. Monorepo (Single Repository)         | SPA/Next.js and Django live under one root folder                 | Complex tooling required to split CI/CD pipelines correctly; noisy PRs |
| **C. Polyrepo (Separate Repositories)** | Distinct repositories for `backend-testownik` and `web-testownik` | Issue tracking and PRs can sometimes span across multiple repos        |

## Decision

**Option C** was chosen. The architecture is split into focused repositories: `backend-testownik`, `web-testownik`, and `pm-testownik`. 

- This ensures a strict separation of concerns between frontend client and backend API.
- It allows for completely independent deployment lifecycles and highly specialized CI/CD pipelines.
- Engineers can use domain-specific linters and formats (e.g. `ruff` vs `eslint`/`prettier`) from the root of their specific tech stack repo.

## Consequences

- Overall project management requires coordinating tasks across multiple repositories, handled via shared project.
- E2E tests covering both sides (Playwright etc.) will be slightly more complex to setup locally as they require booting both the Node and Python environments.
- There is separation of concerns and clear ownership of codebases, which is beneficial for scaling the team and maintaining code quality.
