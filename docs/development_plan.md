# Development Plan: EduBot

**Version:** 1.0 (MVP)
**Status:** Draft for Review
**Owner:** Abhishek
**Related documents:** PRD v1.0, SRS v1.0, Architecture v1.0, UI/UX v1.0

---

## 1. Purpose

This plan converts the PRD, SRS, Architecture, and UI/UX documents into an ordered, dependency-aware build sequence with clear milestones and a Definition of Done (DoD) for each. It exists so implementation can start immediately without re-deriving scope or sequencing decisions.

**Time budget context (from prior notes):** v1 MVP is scoped for roughly 4–6 focused hours of build time, so phases below are intentionally small and sequential, not parallelizable across a team.

---

## 2. Guiding Priorities

1. **Grounding correctness before polish.** A working retrieval + refusal loop matters more than UI styling.
2. **Measure before declaring done.** No phase involving answer quality is "done" until it's been run against the test set.
3. **Avoid scope creep.** Anything in the PRD's "Out-of-Scope" list (OCR, hints, auth, multi-subject) is not touched in this plan.
4. **Deploy early, iterate live.** Get a bare-bones version deployed before polishing, so integration/hosting risk surfaces early rather than at the end.

---

## 3. Phase Overview and Dependencies

```
Phase 0: Setup
    ↓
Phase 1: Data Layer (corpus + embeddings + vector store)
    ↓
Phase 2: Backend Core (retrieval + answer/refusal logic)
    ↓
Phase 3: Frontend (chat UI)          ─┐
    ↓                                 │ (can start once Phase 2's
Phase 4: Integration (frontend ↔      │  API contract is defined,
         backend wired end-to-end)   ─┘  even before full logic works)
    ↓
Phase 5: Evaluation & Bug Fixing
    ↓
Phase 6: Deployment
    ↓
Phase 7: Post-Deploy Verification & Handoff
```

**Key dependency notes:**
- Phase 3 (Frontend) only needs the `/chat` API *contract* (request/response shape) from Phase 2 to begin — it does not need to wait for retrieval logic to be fully correct. This lets frontend and backend logic proceed with minimal blocking.
- Phase 5 (Evaluation) cannot start meaningfully until Phase 4 (Integration) is complete, since it tests the whole loop.
- Phase 6 (Deployment) requires Phase 5's baseline hallucination-rate measurement to exist (per PRD acceptance criteria — before/after comparison).

---

## 4. Phase 0 — Setup

**Goal:** A clean, reproducible project skeleton with no leftover code from the prior haphazard build (per existing notes: complete rebuild, not a patch).

| Task | Priority | Depends on |
|---|---|---|
| Initialize new repo with clean file structure (`/backend`, `/frontend` or single Streamlit app, `/data`, `/eval`) | Must | — |
| Set up Python environment + dependency file (requirements.txt / pyproject.toml) | Must | Repo init |
| Configure environment variable handling for API keys (no hardcoded secrets) | Must | Repo init |
| Set up `.gitignore` (exclude secrets, vector index artifacts if large, `__pycache__`) | Must | Repo init |
| Create README skeleton (to be filled in Phase 6/7) | Should | Repo init |

**Definition of Done (Phase 0):**
- [ ] Repo exists with the agreed folder structure and no residual code from the previous version.
- [ ] Dependencies install cleanly in a fresh environment.
- [ ] No API key or secret appears in any tracked file.
- [ ] A placeholder README exists with project name and one-line description.

---

## 5. Phase 1 — Data Layer

**Goal:** Class 9–11 science syllabus content is chunked, embedded, and queryable.

| Task | Priority | Depends on |
|---|---|---|
| Confirm source syllabus content is available in a usable text format | Must | Phase 0 |
| Define chunking strategy (chunk size, overlap) — starting defaults, to be tuned later | Must | Source content confirmed |
| Write ingest script: chunk → embed → write to vector store (Chroma/FAISS) | Must | Chunking strategy |
| Tag each chunk with topic/chapter metadata (required for source-label display, EDU-FR-009) | Must | Ingest script |
| Run ingest script, confirm vector store loads and returns results for a sample query | Must | Ingest script + tagging |

**Definition of Done (Phase 1):**
- [ ] Ingest script runs end-to-end from raw syllabus text to a loadable vector index.
- [ ] Every chunk has an associated topic/chapter label (EDU-DATA-001).
- [ ] A manual test query against the vector store returns plausible, relevant chunks.
- [ ] Ingest is a separate, re-runnable step from the runtime app (per Architecture Section 10 — not built at request time).

---

## 6. Phase 2 — Backend Core

**Goal:** A working `/chat` endpoint implementing retrieval, the relevance-threshold decision, answer generation, and refusal logic — independent of any frontend.

| Task | Priority | Depends on |
|---|---|---|
| Define `/chat` request/response contract (per Architecture Section 5) | Must | Phase 0 |
| Implement input validation (non-empty, ≤500 chars, sanitize) — EDU-VAL-001–003 | Must | Contract defined |
| Implement retrieval service: embed question, query vector store, return top-k + scores | Must | Phase 1 |
| Implement relevance-threshold decision logic (answer vs. refusal) — EDU-BR-003 | Must | Retrieval service |
| Implement context-constrained prompt construction + LLM call for in-scope questions | Must | Threshold logic |
| Implement refusal response path (no LLM call) — EDU-FR-011, EDU-FR-012 | Must | Threshold logic |
| Implement `/health` endpoint | Should | Phase 1 vector store loaded |
| Implement evaluation logger (question, chunk IDs, scores, classification) — EDU-DATA-004 | Must | Retrieval + decision logic |
| Implement error handling (LLM timeout, vector store failure, malformed scores) — EDU-ERR-001–003 | Must | Core logic above |
| Implement basic mixed-scope and ambiguous-follow-up handling — EDU-EDGE-001, EDU-EDGE-002 | Should | Core logic above |

**Definition of Done (Phase 2):**
- [ ] `/chat` endpoint callable via curl/Postman, returns correct shape for both answer and refusal cases.
- [ ] Validation rejects empty, over-length, and malicious/injection-style input as specified.
- [ ] Refusal path never calls the LLM (verifiable via logs).
- [ ] Evaluation logger writes an entry for every request, without user-identifying data.
- [ ] Simulated failures (kill LLM call, corrupt vector store path) produce the specified generic error response, not a crash or stack trace.

---

## 7. Phase 3 — Frontend

**Goal:** The single-screen chat UI per the UI/UX document, wired to a mocked or early version of the `/chat` contract.

| Task | Priority | Depends on |
|---|---|---|
| Build header, chat panel, composer layout (Streamlit primitives) | Must | Phase 2 contract defined (not full logic) |
| Implement Empty State with example chips | Must | Layout |
| Implement message bubble rendering (user, bot-answer, bot-refusal) | Must | Layout |
| Implement composer validation (char limit, disable-on-submit) — mirrors EDU-VAL | Must | Layout |
| Implement loading indicator | Should | Layout |
| Implement "New conversation" reset | Should | Layout |
| Implement error/retry bubble UI | Should | Layout |
| Apply typography, color tokens, spacing per UI/UX Sections 12–14 | Should | All components above |
| Verify responsive behavior at mobile/tablet/desktop breakpoints | Should | All components above |

**Definition of Done (Phase 3):**
- [ ] All states from UI/UX Section 9 (empty, loading, answer, refusal, clarification, error) are visually implemented, even if wired to mock data.
- [ ] Composer enforces validation rules matching backend rules (defense-in-depth, not a substitute for backend validation).
- [ ] Layout verified at the three breakpoints defined in UI/UX Section 10.
- [ ] Basic keyboard operability confirmed (Tab/Enter through composer and buttons).

---

## 8. Phase 4 — Integration

**Goal:** Frontend and backend connected end-to-end; a real question typed in the UI produces a real grounded answer or refusal.

| Task | Priority | Depends on |
|---|---|---|
| Wire frontend composer submit → backend `/chat` call | Must | Phase 2 + Phase 3 |
| Wire backend response → correct bubble state rendering (answer/refusal/error) | Must | Above |
| Wire source-topic label display from backend response | Must | Above |
| Implement multi-turn context passing (last 1–2 turns) frontend → backend | Must | Above |
| Implement Retry action re-sending the failed question | Should | Above |
| Smoke-test full loop manually with 5–10 ad hoc questions (in-scope and out-of-scope) | Must | All above |

**Definition of Done (Phase 4):**
- [ ] A student can ask an in-scope question in the deployed-locally app and receive a grounded answer with a source label.
- [ ] An out-of-scope question produces a refusal bubble, not a fabricated answer.
- [ ] A follow-up question correctly uses prior turn context.
- [ ] Simulated backend failure surfaces the error/retry UI correctly (not a frozen or broken UI).

---

## 9. Phase 5 — Evaluation and Bug Fixing

**Goal:** Objective measurement of quality, and fixes targeted at the top hallucination cause — matching the PRD's core success metrics.

| Task | Priority | Depends on |
|---|---|---|
| Build the 20-question labeled test set (in-scope, out-of-scope, edge cases) | Must | Phase 4 |
| Run baseline evaluation pass, log correct/partial/hallucinated per question | Must | Test set + Phase 4 |
| Analyze failures, identify top hallucination cause (chunking, prompt constraint, threshold miscalibration) | Must | Baseline pass |
| Apply targeted fix (e.g., adjust chunk size/overlap, tighten prompt instructions, adjust relevance threshold) | Must | Root cause identified |
| Re-run the same 20-question test set, compare before/after | Must | Fix applied |
| Fix any UI/UX bugs surfaced during evaluation (e.g., broken states, layout issues) | Should | Evaluation runs |
| Verify edge cases from SRS Section 9 explicitly (mixed-scope, ambiguous follow-up, rapid double-submit, vague question) | Must | Phase 4 |
| Adversarial test: attempt prompt-injection / scope-override phrasing (EDU-EDGE-003, EDU-SEC-003) | Must | Phase 4 |

**Definition of Done (Phase 5):**
- [ ] Documented before/after hallucination rate on the identical 20-question set.
- [ ] Scope-refusal accuracy ≥ 90% on the test set (per SRS EDU-PERF/Acceptance targets).
- [ ] All SRS edge cases (Section 9) manually verified with documented expected vs. actual outcome.
- [ ] No prompt-injection attempt in testing successfully overrides scope restriction.
- [ ] All "Must"-priority bugs found during evaluation are fixed; "Should"-priority bugs are logged if deferred.

---

## 10. Phase 6 — Deployment

**Goal:** Publicly reachable, stable deployment matching the Architecture document's target platform.

| Task | Priority | Depends on |
|---|---|---|
| Configure secrets (API keys) in the hosting platform's secrets manager | Must | Phase 5 complete |
| Deploy to Streamlit Community Cloud or Hugging Face Spaces | Must | Secrets configured |
| Verify `/health` (or equivalent readiness check) on the deployed instance | Must | Deployed |
| Re-run the 20-question test set against the deployed instance (not just local) | Must | Deployed |
| Verify performance thresholds informally (response time under normal use) — EDU-PERF-001 | Should | Deployed |
| Finalize README (problem, approach, before/after metrics, screenshots, live link) | Must | Deployed + verified |
| Push final code to public/shareable GitHub repo | Must | README finalized |

**Definition of Done (Phase 6):**
- [ ] App is live at a public URL and functions identically to the locally tested version.
- [ ] Test set results on the deployed instance match (or are close to) local results — no deployment-induced regression.
- [ ] README is complete per PRD acceptance criteria (problem, approach, metrics, screenshots).
- [ ] Repository is public/shareable with no secrets committed.

---

## 11. Phase 7 — Post-Deploy Verification and Handoff

**Goal:** Final sign-off against the PRD/SRS acceptance criteria as a single checklist, and a clear record of what's deferred to v2/v3.

| Task | Priority | Depends on |
|---|---|---|
| Walk through PRD Section 11 acceptance criteria end-to-end on the live app | Must | Phase 6 |
| Walk through SRS Section 12 acceptance criteria end-to-end on the live app | Must | Phase 6 |
| Document known limitations (free-tier hosting, 20-question eval size, English-only, etc.) in README | Should | Phase 6 |
| Log deferred items explicitly (OCR, Socratic hints, guardrail expansion) as "future work," not silently dropped | Should | Phase 6 |

**Definition of Done (Phase 7):**
- [ ] Every PRD and SRS acceptance-criteria checkbox is verifiably true on the deployed app (not just "should work").
- [ ] README explicitly states known MVP limitations, so they read as intentional scoping, not oversights.
- [ ] A short "Future Work" section lists v2 (Socratic hints) and v3 (guardrails/expanded eval) as named next phases.

---

## 12. Milestones Summary

| Milestone | Marks completion of | Exit condition |
|---|---|---|
| **M0 – Project Skeleton Ready** | Phase 0 | Clean repo, dependencies install, no secrets committed. |
| **M1 – Corpus Queryable** | Phase 1 | Vector store returns relevant, labeled chunks for sample queries. |
| **M2 – Backend Functional (Standalone)** | Phase 2 | `/chat` works correctly via direct API calls, including refusal and error paths. |
| **M3 – Frontend Built (Standalone)** | Phase 3 | All UI states implemented against mock data. |
| **M4 – End-to-End Working** | Phase 4 | A real question in the UI produces a real, correct answer or refusal. |
| **M5 – Quality Verified** | Phase 5 | Documented before/after metrics meet PRD/SRS targets. |
| **M6 – Live and Documented** | Phase 6 | Public URL live, README complete, repo public. |
| **M7 – MVP Sign-Off** | Phase 7 | All PRD/SRS acceptance criteria checked off on the live app. |

---

## 13. MVP Scope Recap (Do-Not-Exceed List)

To prevent scope drift mid-build, this plan builds **only**:
- Single-corpus (Class 9–11 science), text-only, single-session chat.
- Retrieval-grounded answers with source labeling.
- Explicit scope refusal for out-of-scope questions.
- A single deployed Streamlit app with no auth, no accounts, no admin surface.
- A 20-question evaluation set and before/after quality measurement.

**Not built in this plan** (confirmed out-of-scope per PRD Section 9): OCR/image input, Socratic hint system, multi-subject/grade support, user accounts, real-time co-working rooms, voice I/O, native mobile app, advanced multi-layer guardrails.

---

## 14. Overall Definition of Done (MVP)

The EduBot MVP is complete when **all** of the following hold simultaneously:

- [ ] All Phase 0–7 phase-level DoD checklists are satisfied.
- [ ] All PRD Section 11 acceptance criteria pass on the live deployment.
- [ ] All SRS Section 12 acceptance criteria pass on the live deployment.
- [ ] All UI/UX Section 16 acceptance criteria pass on the live deployment.
- [ ] All Architecture Section 14 acceptance criteria pass on the live deployment.
- [ ] Public GitHub repo + live demo link both work when tested from a clean browser session (no cached state assumptions).
- [ ] Documented hallucination-rate and scope-refusal-accuracy numbers exist and meet the PRD's stated targets.