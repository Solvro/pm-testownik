# ADR-002: Migration of the Frontend Framework from Vite to Next.js

**Status:** Accepted  
**Date:** 2026-01-12  
**Author:** Antoni Czaplicki

---

## Context

The initial frontend for Testownik was built as a React Single Page Application (SPA) using Vite. As the application's scope expanded to include open-graph, prefetching and now we are adding heavily integrated AI features (like a contextual assistant and question generation via files), a purely client-side rendering model became problematic.

## Considered Options

| Option                      | Description                                                       | Drawbacks                                                                                                                          |
|-----------------------------|-------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| A. Remain with React + Vite | Continue using the existing fast Vite setup; pure client-side SPA | SEO, open-graph, prefetching and AI features would be harder to implement securely and efficiently                                 |
| **B. Migrate to Next.js**   | Adopt Next.js for its hybrid SSR, Middleware, and AI routes       | Higher deployment complexity (requires a sharing auth tokens between client and  Node server instead of just static file delivery) |

## Decision

**Option B** was chosen. The frontend framework was switched from Vite to Next.js.

- **AI Security & Streaming:** Next.js allows us to keep the LLM interaction securely on it's server-side. We can construct AI prompts and natively handle HTTP streams to the browser without exposing sensitive API keys. We can also use `ai-sdk' for easy integration with LLM providers.
- **Proxy Middleware:** Next.js provides excellent edge middleware allowing us to neatly intercept and proxy network requests to the Django backend, it allows us to add aditional protection by ensuring some requests can only be made from Next.js server.
- **Performance:** While primarily a logged-in app, Next.js Server Components and SSR help with initial load delivery.

## Consequences

- The frontend deployment model shifted from a purely static CDN host to a Node.js runtime.
- Existing Vite configuration and hooks had to be adapted for SSR environments.
- The Next.js route handlers operate as a distinct intermediary layer between the user client and the core Django API backend.
