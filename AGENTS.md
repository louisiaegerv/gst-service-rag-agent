<!-- BEGIN:nextjs-agent-rules -->

# This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` (resolved from this file's directory; in monorepos the `next` package may not be visible from the repo root) before writing any code. Heed deprecation notices.

This block is written and re-added by `next dev` — verify at `node_modules/next/dist/server/lib/generate-agent-files.js`. Removing it from a diff only re-creates the uncommitted change; committing it with your work keeps the tree clean.

<!-- END:nextjs-agent-rules -->

# GST Service RAG Agent

## Purpose

A customer-service assistant modeled on a GST/Toyota dealership virtual service advisor. It combines:

1. **RAG over static knowledge** — vehicle maintenance schedules and warranty basics, grounded in real Toyota owner's-manual PDFs and official FAQ content.
2. **Agentic tool use** — looks up open safety recalls by VIN or make/model/year via the free public NHTSA Recalls API. The model decides when to call this tool vs. answer from retrieval.
3. **Mocked action** — a stubbed "schedule a service appointment" tool call, demonstrating action-taking rather than just answering.

This is a portfolio/learning project — a working, deployable artifact used to prepare for and discuss in a technical interview. Full context and rationale live in `.plans/PROJECT_PLAN.md`.

## Stack

| Layer | Choice |
|---|---|
| Frontend + backend | Next.js (App Router, TypeScript) |
| LLM access | OpenRouter (model-agnostic; targets Claude specifically for demos) |
| Vector store | Qdrant Cloud (free tier) |
| Embeddings | OpenAI `text-embedding-3-small` |
| Recall lookup | NHTSA Recalls API (`api.nhtsa.gov`, no key required) |
| Ingestion | Python script, offline/one-time |
| Testing | Vitest (unit), Playwright (smoke test) |
| CI/CD | GitHub Actions → Vercel |

## Folder structure

- `/app` — Next.js App Router routes, including `/api/chat`
- `/lib/rag` — retrieval logic: embedding calls, Qdrant queries, prompt construction
- `/lib/agent` — tool-calling loop, tool definitions (recall lookup, mocked appointment)
- `/scripts/ingest` — offline Python ingestion pipeline (PDF/FAQ → chunk → embed → upsert)
- `/data/sources` — curated source PDFs/FAQ pages (Corolla, 4Runner, Sienna, Tundra, Supra)

## Commands

- `npm run dev` — start the Next.js dev server (localhost:3000)
- `npm run build` / `npm run start` — production build and start
- `npm run lint` — ESLint
- Ingestion and test commands will be added here once `/scripts/ingest` and the test suites exist (Phases 1 and 5).

## Constraints

- No auth / multi-user accounts in v1.
- "Schedule a service appointment" is a mocked tool — no real backend integration.
- Only a curated subset of each owner's manual is ingested (maintenance schedule tables, warranty basics), not full manuals.
- Out of scope: real appointment booking, Databricks integration (handled outside this app).

## Secrets

- **Never read, print, cat, or otherwise output the contents of `.env`.** It holds live API keys (Qdrant, OpenAI, OpenRouter). Reference `.env.example` for what variables are expected and their purpose — it holds placeholder values only and is safe to read.
- When adding a new required variable, update `.env.example` (with a placeholder) in the same change, not just `.env`.
