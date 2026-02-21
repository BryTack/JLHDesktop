# JLH Desktop — Claude Code Context

## First: Read the Spec
The full project specification is in `SPEC.md`. Read it before making any suggestions or changes.

---

## Project in One Sentence
A standalone Windows desktop app (.exe) that lets a non-technical user (Joanna) define reusable AI-powered tasks once, through a guided conversation, and use them repeatedly via a clean generated form — without ever needing to understand the AI or repeat herself.

## Current Status
**Planning phase.** No code written yet. The spec is being actively developed. Do not begin building until instructed.

## Key Principles (Never Violate These)
- **AI agnostic** — never assume or hardcode a specific AI provider. Provider and model are configured in one place (admin page). Current default is Claude (Anthropic) but this is a config choice, not architecture.
- **Non-technical UX** — Joanna is not technical. Every user-facing message must be plain English. No jargon, no stack traces, no technical detail visible to her.
- **AI does the thinking** — the AI suggests, categorises, decides field types, writes summaries. Joanna confirms. She does not configure.
- **Define once, reuse forever** — tasks must persist permanently and be fully self-explanatory when revisited months later.
- **Start minimal** — data tables, event types, and documents grow on demand. Do not add things speculatively.

## Tech Stack (Confirmed)
- Electron + electron-vite + TypeScript + React
- Shadcn/UI + TailwindCSS
- SQLite via better-sqlite3 + Drizzle ORM
- react-jsonschema-form (RJSF) for form rendering
- Vercel AI SDK (provider-agnostic)
- Tavily for web search
- Electron `safeStorage` for API keys

## Key Terminology
- **Task** — a reusable, saved unit of work Joanna defines and runs
- **Safety Agent** — the hidden AI reviewer that checks all actions against safety rules
- **Cog** — the per-task refinement dialog (⚙ icon)
- **User mode / Admin mode** — two app modes with different safety rules applied
- **Documents table** — stores persistent text documents (safety rules, prompts etc.)
- **Event log** — records `ai_interaction` and `error` events (grows on demand)

## What's Not Built Yet
Everything. Build order is in SPEC.md Section 19. Start with Phase 0 (companion test harness) when instructed.
