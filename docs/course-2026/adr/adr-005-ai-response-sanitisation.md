# ADR-005: AI response sanitisation using `nh3`

**Status:** Accepted  
**Date:** 2026-04-09  
**Author:** Antoni Czaplicki

---

## Context

LLM responses may contain arbitrary HTML or JavaScript. These responses are rendered in the browser (hint text, generated question content), creating an XSS risk if returned unsanitised. The project already depends on `nh3==0.3.4` — a Rust-backed HTML sanitiser used elsewhere in the backend.

## Considered Options

| Option | Description | Drawbacks |
|--------|-------------|-----------|
| A. No sanitisation | Simplest | Direct XSS exposure |
| B. Python `bleach` | Established Python library | Slower than Rust; deprecated upstream |
| **C. `nh3`** | Rust-backed, already in `requirements.txt`, fast | Allowlist must be explicitly configured |
| D. Frontend-only sanitisation (DOMPurify) | Sanitise in the browser | Server still sends unsafe content; defence-in-depth is weaker |

## Decision

**Option C** was chosen. `nh3` is already a project dependency and provides fast, safe HTML sanitisation on the server before any content reaches the client. Frontend rendering of AI content may still apply DOMPurify as an additional layer.

## Consequences

- An HTML tag and attribute allowlist must be defined (e.g. `<b>`, `<i>`, `<code>`, `<ul>`, `<li>`, `<p>`).
- All AI-generated text (hint responses, generated question text, answer text) passes through `nh3.clean()` before serialisation.
- Unit tests are required to verify that disallowed tags are stripped and allowed tags are preserved.
