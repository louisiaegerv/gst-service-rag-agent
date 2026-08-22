# GST Service RAG Agent — Implementation Plan

**Purpose of this project:** a learning-by-building tutorial that results in a real, deployable RAG + agentic-tool-calling app, closely modeled on the Full-Stack AI Engineer role at Gulf States Toyota / Friedkin Group. Secondary goal: every architecture decision below should be something you can explain and defend out loud in a technical interview.

**Learning mode:** User creates the files and writes the code provided by the LLM (Next.js files, RAG pipeline, agent logic) in the chat. Claude explains concepts, shows commented reference examples, and reviews your code on request — Claude does not create and modify the files unless requested by the user. The user wants to learn about LLM concepts and how to implement them in a real-world application.

---

## 1. What we're building

A customer-service assistant modeled on a GST/Toyota dealership virtual service advisor:

1. **RAG over static knowledge** — answers questions about vehicle maintenance schedules and warranty basics, grounded in real, publicly published Toyota owner's-manual PDFs and official FAQ content.
2. **Agentic tool use** — looks up open safety recalls by VIN or make/model/year using the free public **NHTSA recall API**. This is the part that makes it agentic rather than pure retrieval: the model has to decide *when* to call a tool vs. answer from the knowledge base.
3. **Mocked action** — "schedule a service appointment," a stubbed tool call (no real backend integration needed) that demonstrates the agent taking an action, not just answering.

Out of scope for v1: Option B (AS Roma assistant), real appointment booking, auth/multi-user accounts, Databricks integration (handled separately per the prep plan, not inside this app).

---

## 2. Architecture

```
┌─────────────┐      ┌──────────────────┐      ┌───────────────────┐
│  Next.js UI │◄────►│ Next.js API route │◄────►│  OpenRouter (LLM)  │
│  (chat)     │      │  /api/chat        │      │  generation +      │
└─────────────┘      │  (agent loop)     │      │  tool-call routing │
                      └─────┬─────┬──────┘      └────────────────────┘
                            │     │
                 ┌──────────┘     └───────────┐
                 ▼                            ▼
        ┌─────────────────┐          ┌─────────────────────┐
        │ Qdrant Cloud      │          │ NHTSA Recall API     │
        │ (vector search)   │          │ (real, free, no key)  │
        └────────┬─────────┘          └─────────────────────┘
                 ▲
                 │ embeddings at query + ingest time
        ┌─────────────────┐
        │ OpenAI Embeddings │
        │ (text-embedding-  │
        │  3-small)          │
        └─────────────────┘
                 ▲
                 │ chunk + embed (offline script)
        ┌─────────────────────┐
        │ Ingestion pipeline    │
        │ (Python, one-time)    │
        │ owner's manual PDFs + │
        │ FAQ pages → chunks     │
        └─────────────────────┘
```

Backend = Next.js API routes (this satisfies the JD's "backend technologies" requirement without forcing in FastAPI/Django — consistent with the prep plan's read of the JD).

---

## 3. Stack decisions (with interview rationale)

| Layer | Choice | Why (your talking point) |
|---|---|---|
| Frontend + backend | Next.js (App Router, TypeScript) | Matches your existing production stack; API routes = real backend, not just UI. |
| LLM access | OpenRouter | You already hold a key; lets you swap models (Claude, GPT, etc.) without vendor lock-in — good "engineering judgment" story, and lets you demo with Claude specifically since the JD is Claude Code-centric. |
| Vector store | **Qdrant Cloud** (1GB free tier) | Open-source, self-hostable, Rust-performance-focused — a deliberate choice over Pinecone's proprietary black box. Gives you a real "why not the popular default" trade-off story, which fits a JD that explicitly wants an *engineer*, not someone who just picks the trendiest managed SaaS. Pinecone/Chroma noted as alternatives you evaluated. |
| Embeddings | **OpenAI `text-embedding-3-small`** | OpenRouter doesn't serve embeddings endpoints, so this needs a separate provider. `text-embedding-3-small` is OpenAI's cheap, high-throughput embedding model (~$0.02 / 1M tokens) — plenty for a curated manual/FAQ corpus, and you already hold a key so there's no new account to set up. `text-embedding-3-large` is the pricier, higher-fidelity alternative if retrieval quality ever needs a bump. |
| Recall lookup | NHTSA Recalls API (`api.nhtsa.gov`) | Real government API, free, no key — this is the tool-calling / agentic piece. |
| Ingestion | Small Python script (offline, run once/on-demand) | Matches the prep plan's suggestion to get a low-stakes Python touchpoint without forcing Python into the live app. |
| Testing | Vitest (unit tests on chunking/retrieval logic), Playwright (one smoke test on the chat flow) | Proportionate to a portfolio project while still hitting the JD's "automated testing" bullet. |
| CI/CD | GitHub Actions (lint + test on push) → Vercel deploy | Hits the JD's DevOps bullet with minimal overhead. |
| Deployment | Vercel | Matches your existing hosting experience. |

---

## 4. Data sourcing (needs a short research pass before Phase 1)

- Toyota owner's manuals: publicly downloadable PDFs from toyota.com (per model/year) — using **Corolla, 4Runner, Sienna, Tundra, and Supra**, a deliberately varied set (sedan, SUV, minivan, truck, sports car) so retrieval has to actually discriminate between vehicle types rather than trivially matching a single-model corpus.
- Toyota official FAQ / maintenance-schedule pages: public support content, not scraped from GST's live dealer site (keeps this clean of ToS concerns per the prep plan).
- NHTSA recall API requires no content sourcing — it's called live.

This is a good first "guided" task: you pick and download 2-3 manuals + FAQ pages, Claude helps you scope what's reasonable to chunk for a demo (not the entire manual — a curated subset: maintenance schedule tables, warranty basics sections).

---

## 5. Phased build plan

**Phase 0 — Project setup**
- `git init`, GitHub repo, Next.js + TypeScript scaffold, `.env.example`, base folder structure.
- Create `CLAUDE.md`, `AGENTS.md`, and the two subagents (§6) *before* writing app code, so the guided sessions from here on have real context to work from.

**Phase 1 — Ingestion pipeline**
- Python script: load PDFs/FAQ → chunk → embed (OpenAI `text-embedding-3-small`) → upsert into Qdrant.
- You write this yourself; Claude explains chunking strategy trade-offs (fixed-size vs. semantic/section-based — manuals have natural headers, which matters for quality).

**Phase 2 — Core RAG chat**
- `/api/chat` route: embed the user query, retrieve top-k chunks from Qdrant, construct a grounded prompt, call OpenRouter, stream the response.
- Minimal chat UI (single page, message list + input).

**Phase 3 — Agentic tool-calling**
- Add the NHTSA recall lookup as a tool the model can call (OpenRouter/Claude tool-use format).
- Prompt/system design so the model reliably chooses: answer from retrieval vs. call the recall tool vs. ask for a VIN/model-year when missing.

**Phase 4 — Mocked action tool**
- "Schedule a service appointment" as a third tool — no real backend, just a structured confirmation response. Demonstrates action-taking, completes the "agentic workflow" story.

**Phase 5 — Polish for interview readiness**
- Vitest + Playwright tests, GitHub Actions CI, Vercel deploy.
- Case-study README: problem, architecture diagram, key decisions/trade-offs (pull straight from §3 of this doc), what you'd change at scale (auth, real booking integration, evaluation/observability for RAG quality).
- Optional: a short `docs/decisions.md` (lightweight ADR log) capturing decisions *as you make them* during the build — more authentic and easier to talk about than reconstructing rationale after the fact.

---

## 6. Claude Code project-context files

Two files, created in Phase 0, before any app code:

### `CLAUDE.md` (root)
Claude Code-specific project memory. Should capture:
- One-paragraph project description and goal (learning + interview artifact — say so explicitly, it changes how Claude should behave, e.g. "explain before showing code, don't just implement").
- Stack summary (table from §3, kept in sync as decisions solidify).
- Your stated working mode: **you write the code, Claude explains/reviews, does not write app code unless explicitly asked.** This is the single most important line in the file — it overrides Claude Code's normal default of just implementing things.
- Folder structure conventions once they exist (`/app`, `/lib/rag`, `/lib/agent`, `/scripts/ingest`, `/data/sources`).
- Commands: how to run ingestion, dev server, tests.
- Known constraints: no auth in v1, mocked appointment tool, curated subset of manuals only.

### `AGENTS.md` (root)
Cross-tool standard (Codex, Cursor, and other AI coding tools read this format; the GST JD explicitly names Claude Code, Codex, *and* Augment Code, so having a tool-agnostic file — not just a Claude-specific one — is itself a small interview talking point about working in a multi-tool AI-assisted team). Content overlaps with CLAUDE.md but stays tool-neutral: project purpose, stack, conventions, how to run/test. CLAUDE.md then only needs to hold the Claude-Code-specific working-mode instruction and anything Claude-specific (subagent references, etc.).

### Subagents (`.claude/agents/*.md`)

Chosen to look like something a real team would check into a repo — not interview scaffolding. Both are read-only reviewers/checkers, not code generators, which keeps the "you write the code" working mode intact.

**`code-reviewer`** — general-purpose review pass for what you write.
- Tools: Read, Grep, Glob (read-only — it reviews, doesn't edit).
- Triggered when you ask for a review of a piece of code before you consider it done.
- Checks for: correctness bugs, consistency with the conventions documented in `CLAUDE.md`/`AGENTS.md`, obvious security issues (e.g. unvalidated tool-call input, secrets in code), and general code quality. This is the standard "AI-assisted dev team" pattern the JD's "maintain strong engineering standards when using AI-generated code" bullet is describing.

**`rag-eval`** — retrieval/answer-quality regression checker, specific to this project.
- Tools: Read, Grep, Glob, Bash (needs to run the eval script), Write (to save the eval report).
- Runs a small fixed set of test questions (e.g. "what's the maintenance interval for a Corolla's oil change," "does the Tundra have an open recall for X") against the live pipeline and checks whether retrieval pulled the right source chunks and the answer stayed grounded in them.
- This is a real production pattern for RAG systems — teams maintain small eval harnesses to catch silent regressions when chunking, prompts, or models change — and it maps directly to the JD's "Evaluate and integrate appropriate AI tools, APIs, and models for business use cases" bullet. It's also a legitimately good engineering habit to demonstrate you *have*, independent of the interview.

This matches your answer of "CLAUDE.md + a couple of scoped subagents" — no slash commands or nested per-directory CLAUDE.md files for v1, since a single-app portfolio project doesn't have the multi-package complexity that justifies them.

---

## 7. Decisions locked in

| Decision | Choice |
|---|---|
| Embeddings | OpenAI `text-embedding-3-small` (key already held) |
| Vehicle models for manuals | Corolla, 4Runner, Sienna, Tundra, Supra |
| GitHub repo timing | Scaffold locally first, then `gh repo create` / push from local |
| Vector store | Qdrant Cloud |

---

## 8. Immediate next steps (Phase 0 checklist)

- [x] `git init` locally
- [x] Scaffold Next.js (TypeScript, App Router)
- [x] Sign up for Qdrant Cloud, get API key, add to `.env.example`/`.env.local` (OpenAI + OpenRouter keys already held — add those too)
- [x] Write `CLAUDE.md` and `AGENTS.md`
- [ ] Write the two subagent definitions (`code-reviewer`, `rag-eval`)
- [ ] Push local repo to GitHub via `gh repo create` / `gh` CLI
- [ ] Source and download owner's manuals + FAQ pages for Corolla, 4Runner, Sienna, Tundra, Supra (Phase 1 prep)



---

Reference Files: 
- `C:\Users\louis\Documents\Personal\Jobs\Gulf States Toyota\Full Stack AI Engineer - Gulf States Toyota.md`
- `C:\Users\louis\Documents\Personal\Jobs\Gulf States Toyota\Prep Plan Full-Stack AI Engineer.md`