# ADR-005: AI Integration Strategy

**Status:** Accepted  
**Date:** 2026-05-17  
**Author:** Antoni Czaplicki

---

## Context

Testownik is a quiz-based learning platform for university students. We want to integrate AI to enhance the learning experience - helping students understand quiz material, providing contextual hints, and in the future automating quiz creation from lecture materials.

Key constraints:
- AI must remain optional - users can disable it entirely
- Costs must stay manageable (thousands of students, multiple requests per session)
- The assistant must stay scoped to learning - not become a general-purpose chatbot
- AI should never be forced or used without a clear purpose - it's only allowed if it provides a real value

## Considered Options

| Option | Description | Drawbacks |
|--------|-------------|-----------|
| A. Link to external ChatGPT | Redirect users to ChatGPT with pre-filled context, previously implemented as it | No deep integration; user leaves the app; no tool use; no context |
| B. Client-side inference (WebLLM) | Run a small model in the browser | Models too large to download; quality insufficient for Polish academic content; no image support |
| C. Self-hosted open-source model | Deploy Llama/Mistral on own infrastructure | Worse quality; significant maintenance burden; long wait until it would get set up |
| **D. Server-side streaming via Vercel AI SDK + OpenAI** | Next.js API routes stream responses from OpenAI models with tool use | Depends on external provider; ongoing API cost per request |

## Decision

**Option D** was chosen. The Vercel AI SDK provides a provider-agnostic abstraction layer, and OpenAI's `gpt-5.4-mini` offers the best cost/quality/speed ratio for our use case.

### Architecture

- **API routes** in Next.js (`/ai/chat`, `/ai/explain`) handle authentication, permission checks, and stream responses to the client
- **Model selection** is centralized in `src/lib/ai/model.ts` - easy to swap providers
- **Image handling** (`src/lib/ai/images.ts`): fetches images from trusted origins only, converts to base64 via `sharp`, injects as multimodal content
- **System prompts** (`src/lib/ai/prompts.ts`): dynamically built with quiz context, question details, and user permissions

### Implemented Features

1. **AI Chat** - A conversational assistant embedded as a floating panel (`@assistant-ui/react`). Has access to tools:
   - `generate_practice_questions` - creates interactive quiz cards for self-testing that can be later added to current quiz
   - `get_question` - fetches any question from the quiz by number
   - `edit_question` - proposes edits (permission-gated, requires user confirmation)
   - `disable_ai` - lets users opt out from within the chat

2. **AI Explain** - A one-shot explanation triggered per question:
   - Post-answer mode: explains why answers are correct/incorrect
   - Pre-answer mode: provides hints without revealing answers (structured XML output parsed client-side)

### Access Control & Security

- Requires authenticated user with `AI_FEATURES` permission (guests excluded)
- Feature-flagged via `NEXT_PUBLIC_AI_ENABLED` environment variable
- Only images from trusted origins (API server, S3) are forwarded to the model
- System prompt constrains the assistant to quiz/learning topics only
- Users can disable AI via chat tool call or settings page

### Planned Features

- **Quiz generation from notes/slides** (Issue #26): Upload lecture PDFs → AI generates a complete quiz. Probably will be done entirely on Django backend side.
- **AI-assisted suggestions in comments** (Issue #37): When reporting question errors, AI drafts a suggested fix for the quiz owner to accept.

## Consequences

- Ongoing per-request cost to OpenAI (mitigated by using mini model; may need rate limiting at scale)
- External dependency on OpenAI availability - the AI SDK abstraction allows swapping providers if needed
- The tool-use pattern makes the assistant actionable but increases prompt complexity and token usage
- Future quiz-generation feature will require evaluating larger/more capable models and potentially a different cost model (batch processing)
