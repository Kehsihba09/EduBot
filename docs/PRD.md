# Product Requirements Document: EduBot

**Version:** 1.0 (MVP)
**Status:** Draft for Review
**Owner:** Abhishek

---

## 1. Problem Statement

Students preparing for Class 9–11 science syllabi rely on scattered resources — textbooks, YouTube videos, tuition notes, and general-purpose chatbots — to get their doubts resolved. General-purpose AI chatbots are not grounded in the student's actual syllabus, which leads to two failure modes: (1) they answer confidently with content outside the syllabus scope (extra depth, wrong curriculum, hallucinated facts), and (2) they hand over final answers directly, which shortcuts learning and reduces retention.

There is no lightweight, syllabus-scoped conversational tool that lets a student ask a question, get an answer grounded strictly in their curriculum, and — when working through a problem — get guided toward the solution rather than just told the answer.

## 2. Target Users

| User | Description | Core need |
|---|---|---|
| Primary: Student (Class 9–11, science stream) | Preparing for school exams or foundational competitive exam prep | Fast, trustworthy, in-scope answers to syllabus questions |
| Secondary: Self-learner using the syllabus as a structured curriculum | Uses NCERT-aligned material outside formal schooling | Same as above, plus confidence that answers won't mislead them off-syllabus |

Out of scope for this PRD: teachers/admins, parents, and any users outside the Class 9–11 science scope.

## 3. Goals

1. Let a student have a natural chat conversation about topics within their syllabus and get answers grounded in that syllabus content (not hallucinated, not off-scope).
2. Make the system explicitly refuse or redirect when a question falls outside the syllabus scope, rather than answering anyway.
3. Ship a working, demoable, deployed product within a small time budget, with measurable quality (hallucination rate, scope-refusal accuracy) rather than a vague "it works" claim.

### Non-goals (for this version)
- Teaching methodology innovation (e.g., adaptive learning paths, spaced repetition).
- Replacing a tutor for step-by-step problem solving (that's a v2 concern — Socratic hints — not this MVP).
- Supporting every subject/grade — scope is intentionally narrow.

## 4. Core Features (MVP)

1. **Syllabus-grounded Q&A chat** — student types a question; system retrieves relevant syllabus content (via embeddings + retrieval) and answers using only that retrieved context.
2. **Scope detection & refusal** — if a question is outside the ingested syllabus (wrong subject, wrong grade, or general knowledge unrelated to the corpus), the bot says so explicitly instead of guessing.
3. **Source/context transparency (lightweight)** — answer indicates which topic/chapter it drew from, so the student can cross-check against their textbook.
4. **Simple chat UI** — single conversation window, no login/auth complexity, deployed and publicly reachable via a shareable link.

## 5. MVP Scope

**In scope:**
- Single syllabus corpus: Class 9–11 science (as already chunked/ingested).
- Text-only input and output (no OCR, no image upload, no voice).
- Single-turn-aware chat (can reference immediately preceding turns, not long-term memory across sessions).
- One deployed instance (Streamlit Community Cloud or Hugging Face Spaces), no user accounts.
- A fixed evaluation set (~20 questions: in-scope, out-of-scope, edge cases) used to measure and report hallucination/groundedness before and after tuning.

**Explicitly excluded from MVP** (see Section 9): step-by-step guided hint system, OCR/image input, multi-subject support, user accounts/progress tracking, real-time co-working rooms, mobile app.

## 6. User Stories

1. *As a student*, I want to ask a question about a topic in my syllabus and get a clear, correct answer, so that I can resolve a doubt without searching multiple sources.
2. *As a student*, I want the bot to tell me when my question is outside my syllabus, so that I don't get misled into thinking incorrect or irrelevant content is part of my exam scope.
3. *As a student*, I want to see which chapter/topic an answer came from, so that I can verify it against my textbook.
4. *As a student*, I want to ask a natural follow-up question in the same conversation, so that I don't have to repeat context every time.
5. *As the product owner*, I want a labeled test set and hallucination-rate metric, so that I can objectively demonstrate the bot's reliability rather than relying on anecdotal testing.

## 7. Success Metrics

| Metric | Target for MVP |
|---|---|
| Hallucination rate (answers not grounded in retrieved context) | Reduced from baseline measurement to a documented, meaningfully lower rate on the same 20-question test set |
| Scope-refusal accuracy (correctly declines out-of-scope questions) | Bot correctly identifies and refuses ≥ 90% of out-of-scope test questions |
| Answer relevance (in-scope questions answered correctly/partially) | ≥ 80% of in-scope test questions answered correctly or partially correctly |
| Deployment | Publicly accessible demo link + GitHub repo with README (problem, approach, metrics, screenshots) |

## 8. Assumptions

- The Class 9–11 science syllabus content is already available and can be chunked and embedded (existing project asset).
- A single LLM (via API) is available and affordable enough for both answering and any groundedness-check calls.
- Users access the tool via a browser; no native mobile app is needed for MVP.
- 20 test questions are sufficient to demonstrate directional quality improvement for MVP purposes (not a statistically rigorous benchmark).
- No sensitive personal/student data is collected — no login means no PII to protect at this stage.

## 9. Out-of-Scope Features (for MVP)

- OCR / image-based question input (e.g., photographing a handwritten problem).
- Step-by-step Socratic hint system for math problem-solving (planned as a later phase, not MVP).
- Multi-subject or multi-grade support beyond Class 9–11 science.
- User accounts, login, progress tracking, or personalized learning paths.
- Real-time collaborative/co-working study rooms.
- Voice input/output.
- Mobile-native application.
- Advanced guardrail layers beyond basic scope-refusal (e.g., dedicated hint-leakage detection, multi-model groundedness voting) — a simple, single-pass grounding check is sufficient for MVP.

## 10. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Retrieval quality is poor (wrong chunks returned) → hallucinations persist | High | Tune chunk size/overlap; validate against test set before declaring MVP done |
| LLM answers fluently even when context is insufficient (doesn't "know" to refuse) | High | Explicit prompt constraint: answer only from provided context, refuse otherwise; validate via test set |
| Small test set (20 questions) gives a false sense of confidence | Medium | Clearly label metrics as directional/MVP-level in README; expand test set in future phase |
| Scope creep into hint-system or OCR mid-build | Medium | This PRD explicitly excludes them; defer to a separate PRD/phase |
| Hosting limits (Streamlit Community Cloud / HF Spaces free tier) cause downtime or slow responses | Low–Medium | Acceptable for MVP demo purposes; document as known limitation |

## 11. Acceptance Criteria

The MVP is considered complete when all of the following are true:

- [ ] Student can open the deployed app and have a multi-turn chat conversation about Class 9–11 science topics.
- [ ] For in-scope questions, answers are grounded in retrieved syllabus content and indicate the source topic/chapter.
- [ ] For out-of-scope questions, the bot explicitly states the question is outside its syllabus scope rather than attempting to answer.
- [ ] A documented 20-question test set exists with logged results (correct / partial / hallucinated) for both a baseline run and a post-fix run.
- [ ] Hallucination rate on the test set is measurably lower after the fix than the baseline, with both numbers reported.
- [ ] Scope-refusal accuracy on the test set is ≥ 90%.
- [ ] App is deployed and reachable via a public link (Streamlit Community Cloud or Hugging Face Spaces).
- [ ] GitHub repository is public (or shareable) with a README covering: problem statement, approach, metrics (before/after), and screenshots of the working chat.

---

*Next phases (post-MVP, separate PRDs): v2 — Socratic Hint System for step-by-step math guidance; v3 — expanded guardrails and evaluation framework (~40–50 test cases, groundedness/hint-leakage checks).*