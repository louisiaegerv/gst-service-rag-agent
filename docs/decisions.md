# Decision log

Lightweight ADR-style log, written as decisions are made rather than reconstructed afterward. See `.plans/PROJECT_PLAN.md` for the full plan; this file captures the *why* behind choices as they happen.

---

## 2026-08-22 — Tag every chunk with `make`/`model`/`year`/`doc_type` metadata, even in v1

**Context:** v1's corpus is a single model year (2026) across five vehicles (Corolla, 4Runner, Sienna, Tundra, Supra). A natural question: does a single-year demo need per-chunk metadata at all, or can that be added later if the project ever grows to cover multiple model years?

**Decision:** Every chunk gets `make`, `model`, `year`, and `doc_type` (e.g. `warranty_maintenance_guide`, `faq`) as Qdrant payload fields at upsert time in the Phase 1 ingestion script — not deferred.

**Why:**
- The metadata is free to attach now — it's already known per source file at ingest time (filename encodes model/year).
- Retrofitting it later means reprocessing every previously-ingested document, which is real, avoidable rework.
- It's the actual mechanism a production version would need: retrieval for a specific vehicle should be a single **filtered vector search** (Qdrant payload filter combined with the ANN search in one query — `make=Toyota AND model=Corolla AND year=2004`, ranked by embedding similarity within that filter), not three sequential searches, and not one giant unfiltered collection where "oil change interval" could match the wrong model year.
- Consistent with the plan's five-vehicle-type selection (§4) being a deliberate test of *cross-vehicle-type* retrieval discrimination — production would add a *cross-year* discrimination dimension via this same metadata, not a redesign.

**How to apply:** the Phase 1 ingestion script (`/scripts/ingest`) must attach this metadata to every chunk it upserts, regardless of how small the current corpus is. The agent's retrieval-tool logic (Phase 2/3) should extract make/model/year from the user's question (or ask a clarifying question when missing, same pattern as the VIN-for-recalls flow) before building the filtered query — never do an unfiltered similarity search across the whole collection once more than one model year exists.

---

## 2026-08-22 — Write tests alongside each phase, wire up CI early — don't batch testing into Phase 5

**Context:** the original plan (§5) grouped "Vitest + Playwright tests, GitHub Actions CI" entirely into Phase 5 ("Polish for interview readiness"), implying all testing work happens after the app is otherwise done.

**Decision:** split this up —
- GitHub Actions CI (lint + build) moves to Phase 0/1, wired up early so it validates every commit from the start, not just the final state.
- Vitest unit tests are written in the same phase as the logic they cover: chunking tests in Phase 1, retrieval/prompt-construction tests in Phase 2, tool-call routing/validation tests in Phase 3, mocked-tool tests in Phase 4.
- Playwright stays in Phase 5, genuinely — there's no end-to-end chat flow to smoke-test until Phase 2's UI exists, so this isn't a scheduling choice, it's a hard dependency.
- Phase 5 becomes: Playwright smoke test, extending CI to run the full suite, deploy, and the case-study README — not "write all the tests."

**Why:** tests written long after the code they cover are shallower (the author has forgotten the edge cases that mattered) or require re-deriving them from scratch. CI that only turns on at the end provides zero regression protection during the actual build, which is when it's most useful. This project is explicitly meant to mirror how a real engineering team works (see `CLAUDE.md`/`AGENTS.md` working-mode notes) — a real team ships tests in the same PR as the feature, not in a separate "add tests" phase.

**How to apply:** when writing code for any phase from here on, the corresponding Vitest tests are part of that phase's definition of done, not a follow-up task. Don't defer them "to keep momentum" — that's exactly the pattern this decision was meant to avoid.
