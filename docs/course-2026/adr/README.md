# ADR Index – Testownik

**Project:** Testownik | **Semester:** Summer 2026  
**Author:** Antoni Czaplicki

Each decision follows the ADR format: context, considered options, decision, and consequences.

| ID | Title | Status |
|----|-------|--------|
| [ADR-001](adr-001-guest-account-model.md) | Guest mode implemented as a full `AccountType.GUEST` user record | Accepted |
| [ADR-002](adr-002-backend-single-source-of-truth.md) | Backend as the single source of truth – elimination of dual storage logic | Accepted |
| [ADR-003](adr-003-ai-proxy-through-backend.md) | AI calls proxied through the Django backend, not directly from the frontend | Accepted |
| [ADR-004](adr-004-async-views-for-ai.md) | Async DRF views (`adrf`) for interactive AI requests instead of Celery | Accepted |
| [ADR-005](adr-005-ai-response-sanitisation.md) | AI response sanitisation using `nh3` | Accepted |
| [ADR-006](adr-006-feature-flags-constance.md) | Feature flags managed via Django Constance | Accepted |
