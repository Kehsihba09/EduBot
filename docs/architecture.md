# System Architecture Document: EduBot

**Version:** 1.0 (MVP)
**Status:** Draft for Review
**Owner:** Abhishek
**Related documents:** EduBot PRD v1.0, EduBot SRS v1.0

---

## 1. Design Principles

This architecture is deliberately minimal for MVP. It favors:
- **Fewer moving parts** over flexibility — no microservices, no message queues, no multi-tenant infra.
- **Managed/free-tier services** over self-hosted infrastructure, since there is no auth, no PII, and low concurrency requirements (per SRS: 5 concurrent users, 95th percentile < 10s).
- **A single deployable unit** where practical, splitting frontend/backend only where the existing stack (Streamlit + FastAPI) already implies a split.

Anything below marked "future" is explicitly **not** built in MVP — it's noted only so the MVP doesn't paint itself into a corner.

---

## 2. Recommended Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| Frontend | Streamlit | Already the chosen stack; fastest path to a usable chat UI with no separate frontend build step. |
| Backend | FastAPI | Already chosen; lightweight, async-capable, easy to expose a single `/chat` endpoint. |
| Orchestration | LangChain (or direct SDK calls if LangChain overhead isn't earning its keep) | Handles retrieval + prompt assembly; keep usage thin — avoid LangChain's heavier agent/chain abstractions for a single-purpose Q&A flow. |
| LLM provider | Anthropic Claude API (or equivalent hosted LLM) | Hosted, no model-serving infra needed. |
| Embeddings | A hosted embedding API (same provider family as the LLM, or a lightweight open-source model run at ingest-time only) | Embeddings are computed once at corpus-build time, not per request, so cost/latency here is a one-time ingest cost. |
| Vector store | A lightweight embedded vector store (e.g., Chroma or FAISS running in-process) | Corpus size is a fixed Class 9–11 science syllabus — small enough that a hosted vector DB (Pinecone, Weaviate, etc.) is over-engineering for MVP. |
| Session state | In-memory / Streamlit session state | No persistence requirement per SRS (EDU-FR-004, EDU-FR-014). |
| Evaluation logging | Flat file (JSONL) or SQLite | Requirement is just "log enough to compute two metrics" — no need for a full logging pipeline. |
| Hosting | Streamlit Community Cloud or Hugging Face Spaces | Matches PRD's stated deployment target; free tier is acceptable per SRS performance requirements. |

**Explicitly avoided for MVP:** Kubernetes, managed vector databases, message queues, separate auth service, multi-region deployment, dedicated logging/observability stack (e.g., full ELK). These would be justified at scale, not at 5 concurrent anonymous users.

---

## 3. System Components

```
┌─────────────────────────────────────────────────────────┐
│                     Streamlit Frontend                   │
│  - Chat input box                                        │
│  - Chat history display (session-scoped)                 │
│  - "New conversation" control                             │
└───────────────────────┬───────────────────────────────────┘
                         │ HTTPS (REST call)
                         ▼
┌─────────────────────────────────────────────────────────┐
│                     FastAPI Backend                       │
│                                                            │
│  ┌───────────────┐   ┌────────────────┐   ┌────────────┐ │
│  │ Input          │→ │ Retrieval       │→ │ Answer /   │ │
│  │ Validation     │  │ Service         │  │ Refusal    │ │
│  │ (EDU-VAL-*)    │  │ (vector search) │  │ Generator  │ │
│  └───────────────┘   └────────┬───────┘   └─────┬──────┘ │
│                                │                  │        │
│                                ▼                  ▼        │
│                       ┌────────────────┐  ┌──────────────┐│
│                       │ Vector Store   │  │ LLM API call ││
│                       │ (Chroma/FAISS) │  │ (Anthropic)  ││
│                       └────────────────┘  └──────────────┘│
│                                                            │
│                       ┌────────────────────────────────┐  │
│                       │ Evaluation Logger (JSONL/SQLite)│  │
│                       └────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

| Component | Responsibility |
|---|---|
| Frontend (Streamlit) | Renders chat UI, sends question + short conversation history to backend, displays answer/refusal and source label. |
| Input Validation | Enforces EDU-VAL-001–004: non-empty, ≤500 chars, sanitized. |
| Retrieval Service | Embeds the incoming question, queries the vector store, returns top-k chunks with relevance scores and topic labels. |
| Answer/Refusal Generator | Applies the relevance threshold (EDU-BR-003); if above threshold, constructs a context-constrained prompt and calls the LLM; if below, returns a refusal message without calling the LLM (saves cost and latency). |
| Vector Store | Holds pre-computed embeddings for the Class 9–11 science corpus, built once at ingest time, loaded at backend startup. |
| Evaluation Logger | Appends question, retrieved chunk IDs, relevance scores, and answered/refused classification for later metric computation — no user identity attached. |

---

## 4. Data Flow

1. Student types a question in the Streamlit UI.
2. Frontend sends the question (plus last 1–2 turns of session context) to `POST /chat` on the FastAPI backend.
3. Backend validates input (length, non-empty, sanitized).
4. Backend embeds the question and queries the vector store for top-k relevant syllabus chunks.
5. Backend checks the top result's relevance score against a fixed threshold:
   - **Below threshold** → backend returns a refusal response immediately (no LLM call).
   - **At/above threshold** → backend assembles a prompt containing only the retrieved chunks + question, calls the LLM, and returns the generated answer plus the source topic/chapter label.
6. Backend asynchronously writes an evaluation log entry (question, chunk IDs, scores, classification).
7. Frontend appends the response to the chat window and updates in-memory session history.
8. On "new conversation," frontend clears its local session state; nothing to clear server-side since no server-side session persistence exists.

---

## 5. APIs

| Endpoint | Method | Purpose |
|---|---|---|
| `/chat` | POST | Accepts `{question: str, history: list[turn]}`, returns `{type: "answer"\|"refusal", content: str, source_topic: str\|null}`. |
| `/health` | GET | Basic liveness check for the hosting platform / uptime monitoring. |

No other endpoints are needed for MVP — specifically, no corpus-management, no user-management, no analytics-retrieval endpoints (those would serve an Admin role, which is out of scope per SRS).

---

## 6. Authentication and Authorization

Per SRS (EDU-AUTH-001–004): **no authentication or authorization in MVP.** The `/chat` and `/health` endpoints are open. This is acceptable because:
- No PII is collected or stored.
- No destructive or admin actions are reachable from any endpoint.
- The corpus is read-only from the application's perspective (updated only via redeployment, not via API).

**Future (not built now):** if Admin/Teacher roles are introduced, add a standard token-based auth (e.g., JWT or API-key) gate in front of any new management endpoints — do not retrofit auth onto `/chat` unless a real need (e.g., abuse) emerges.

---

## 7. Storage

| Data | Storage mechanism | Persistence |
|---|---|---|
| Syllabus corpus (raw text + chunks) | Flat files (Markdown/text) in the repo or a `data/` directory bundled with deployment | Persistent, versioned via git |
| Embeddings / vector index | Chroma/FAISS index file, built at ingest time, loaded at app startup | Persistent, rebuilt when corpus changes |
| Chat session state | In-memory / Streamlit session state | Ephemeral — cleared on session end |
| Evaluation logs | JSONL file or SQLite file, written by backend | Persistent, but not user-identifiable |

No relational database is introduced for MVP — there is no user data, no multi-table relational structure to justify one. SQLite is offered as an option for evaluation logs only if JSONL becomes inconvenient to query.

---

## 8. Security

Maps to SRS Section 11 (EDU-SEC-001–006):

- **Input handling:** validate and sanitize all input server-side; never execute or `eval()` user-supplied text.
- **Prompt injection resistance:** system prompt and user input are kept in clearly separated message roles (system vs. user) per the LLM API's structure; the system prompt instructs the model to ignore any user attempt to override scope/behavior instructions.
- **Secrets management:** LLM/embedding API keys stored as environment variables / platform secrets (Streamlit Cloud secrets manager or HF Spaces secrets), never committed to the repo or exposed in client-side code.
- **Transport security:** all calls (frontend→backend, backend→LLM/embedding provider) over HTTPS.
- **No error leakage:** internal exceptions caught and translated into generic user-facing messages (EDU-ERR-003); stack traces never sent to the client.
- **No PII:** since there's no auth, ensure logs (Section 4, step 6) never capture IP address or any client-identifying header alongside question content.

---

## 9. Error Handling (Architecture-Level)

- LLM API timeout/failure → backend catches exception, returns a structured error response; frontend shows "temporary issue, please retry."
- Vector store unavailable at startup → backend fails health check (`/health` returns unhealthy) rather than serving degraded/ungrounded answers.
- Malformed relevance scores → treated as retrieval failure, routed to the same error path as vector store unavailability, not silently defaulted to "answer anyway."

---

## 10. Deployment

- **Target:** Streamlit Community Cloud or Hugging Face Spaces (single deployment, matches PRD).
- **Build:** Vector index (Chroma/FAISS) is built as part of a pre-deployment ingest script, not at request time — keeps cold-start and per-request latency low.
- **Configuration:** API keys and thresholds (relevance cutoff, max input length) stored as environment variables / platform secrets, not hardcoded.
- **CI:** Not required for MVP; a simple pre-deploy checklist (run ingest script → run 20-question eval set → deploy) is sufficient given the small scope.

**Future (not built now):** containerize (Docker) and move to a platform like Render/Fly.io/AWS if traffic or reliability needs outgrow the free-tier hosting; add CI/CD only once there's a team or release cadence that benefits from it.

---

## 11. Monitoring

MVP-appropriate, not enterprise-grade:

- **Uptime:** rely on the hosting platform's built-in status page, plus the `/health` endpoint for manual/external uptime checks (e.g., a free uptime-monitoring service pinging `/health`).
- **Quality monitoring:** the evaluation logger (Section 7) doubles as the primary "monitoring" tool — periodically re-run the labeled test set and inspect the JSONL/SQLite logs for drift in hallucination or refusal rates.
- **Error visibility:** rely on the hosting platform's log viewer (Streamlit Cloud / HF Spaces logs) for exceptions; no separate log aggregation service needed at this scale.

**Future (not built now):** structured logging + a dashboard (e.g., Grafana) or an APM tool, only if usage grows beyond what manual log review can handle.

---

## 12. Scalability

Current design targets SRS thresholds: 5 concurrent users, <10s response (95th percentile), <2s retrieval.

- **Retrieval:** in-process vector store (Chroma/FAISS) scales fine for a syllabus-sized corpus and this concurrency level; no need for a distributed vector DB.
- **Backend:** FastAPI's async support handles a handful of concurrent requests on free-tier compute without additional scaling work.
- **Bottleneck to watch:** LLM API latency/rate limits are the most likely constraint as usage grows — not the retrieval layer or hosting platform.

**Future scaling path (not built now, sequenced by actual need):**
1. If concurrency grows beyond free-tier limits → move backend to a small paid container (Render/Fly.io) with autoscaling.
2. If corpus grows beyond a single syllabus (multi-subject/grade) → migrate from an in-process vector store to a managed vector DB.
3. If usage/PII requirements grow → introduce authentication and a proper relational database for user/session data at that point, not preemptively.

---

## 13. Explicit Non-Goals (Avoiding Over-Engineering)

To keep this architecture honest about what it is *not* doing:

- No microservices split beyond the natural frontend/backend boundary already implied by Streamlit + FastAPI.
- No managed vector database, message queue, or container orchestration for a single-corpus, low-concurrency MVP.
- No authentication system for an app with no user accounts or PII.
- No dedicated observability stack when platform logs and a periodic eval-set re-run cover MVP needs.
- No multi-region or high-availability design for a free-tier demo deployment.

---

## 14. Acceptance Criteria

- [ ] Frontend and backend are deployed as a single reachable app via a public URL (Streamlit Community Cloud or HF Spaces).
- [ ] `/chat` endpoint implements the full data flow in Section 4, including the no-LLM-call refusal shortcut.
- [ ] `/health` endpoint correctly reflects vector store availability.
- [ ] No API keys or secrets appear in the repository or client-side code.
- [ ] Evaluation logs are produced in a format that can compute hallucination rate and scope-refusal accuracy (per SRS EDU-DATA-004/005).
- [ ] System meets SRS performance thresholds (EDU-PERF-001–004) under a 5-concurrent-user test.
- [ ] Architecture introduces no component (vector DB, auth service, queue, orchestration platform) that isn't directly required by a stated MVP requirement.