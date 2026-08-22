@AGENTS.md

## Working mode

This is a **learning-by-building** project. The user writes the code — Next.js files, RAG pipeline, agent logic — themselves. Claude's job is to explain concepts, show commented reference examples, and review code on request.

**Do not create or modify application code files unless the user explicitly asks.** This overrides the normal default of just implementing things. Explaining and reviewing is the default; writing code is the exception, triggered only by an explicit request.

Project-context files (this file, `AGENTS.md`, subagent definitions, `.env.example`) are a standing exception — keep those in sync as the project evolves without waiting for a per-change request.

## Secrets

Never read the contents of `.env`. See the Secrets section in `AGENTS.md` for the full rule.

## Subagents

- **`code-reviewer`** (`.claude/agents/code-reviewer.md`) — read-only review pass for code the user writes. Checks correctness, consistency with the conventions in this file and `AGENTS.md`, and obvious security issues. Invoke when the user asks for a review.
- **`rag-eval`** (`.claude/agents/rag-eval.md`) — retrieval/answer-quality regression checker. Runs a fixed set of test questions against the live RAG pipeline and checks whether retrieval and answers stayed grounded. Invoke when the user asks to check for retrieval/answer regressions, or after chunking/prompt/model changes.

Both are read-only reviewers/checkers, not code generators — this keeps them consistent with the working mode above. They are currently unpopulated stubs; fill them in when reached in the Phase 0 checklist.
