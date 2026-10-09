# Software Requirements Specification (SRS): EduBot

**Version:** 1.0 (MVP)
**Status:** Draft for Review
**Owner:** Abhishek
**Related document:** EduBot PRD v1.0

---

## 1. Purpose and Scope

This SRS defines the functional and non-functional requirements for EduBot, a syllabus-grounded chat application for Class 9–11 science students. It translates the PRD's goals into testable, unambiguous requirements. Each requirement is uniquely IDed and independently verifiable.

Requirement ID scheme: `EDU-<CATEGORY>-<NUMBER>` (e.g., `EDU-FR-001`).

---

## 2. User Roles and Permissions

MVP has a single active role. Two future roles are named for forward compatibility but are **not implemented** in this version.

| Role | Status in MVP | Description |
|---|---|---|
| **Student (Anonymous)** | Active | Any visitor to the app; no login required. Can chat, receive answers, start a new session. |
| Admin | Not implemented (future) | Would manage syllabus corpus, view aggregate logs/metrics. |
| Teacher | Not implemented (future) | Would view student query trends, flag bad answers. |

| Requirement ID | Requirement |
|---|---|
| EDU-ROLE-001 | The system shall treat every user as an anonymous "Student" role with no login required for MVP. |
| EDU-ROLE-002 | The system shall not expose any admin, content-management, or corpus-editing functionality to end users in MVP. |
| EDU-ROLE-003 | The system shall not persist any user-identifying information (name, email, IP-linked profile) beyond the duration of the browser session. |

---

## 3. Authentication and Authorization

| Requirement ID | Requirement |
|---|---|
| EDU-AUTH-001 | The system shall not require authentication (login/signup) for any MVP functionality. |
| EDU-AUTH-002 | The system shall not implement role-based authorization in MVP, since only one role (anonymous Student) exists. |
| EDU-AUTH-003 | The system shall not expose any API endpoint that allows modification of the syllabus corpus without a separate, out-of-band deployment/admin process (i.e., no in-app corpus editing). |
| EDU-AUTH-004 | If a future version introduces Admin/Teacher roles, the system shall require authentication for those roles before granting access to management functionality (forward-looking constraint; not built in MVP). |

---

## 4. Functional Requirements

### 4.1 Chat Interaction

| Requirement ID | Requirement |
|---|---|
| EDU-FR-001 | The system shall provide a text input field where a student can submit a question in natural language. |
| EDU-FR-002 | The system shall display the student's submitted question and the system's response in a chronological, scrollable chat window. |
| EDU-FR-003 | The system shall support multi-turn conversation within a single session, where a follow-up question can reference the immediately preceding 1–2 exchanges. |
| EDU-FR-004 | The system shall not persist chat history beyond the active browser session (no cross-session memory in MVP). |
| EDU-FR-005 | The system shall allow the student to start a new conversation, clearing prior chat context. |
| EDU-FR-006 | The system shall limit a single user question to a maximum of 500 characters; inputs exceeding this shall be rejected with a validation message (see EDU-VAL-001). |

### 4.2 Retrieval and Answer Generation

| Requirement ID | Requirement |
|---|---|
| EDU-FR-007 | The system shall retrieve relevant syllabus content from the embedded corpus (Class 9–11 science) for every submitted question before generating an answer. |
| EDU-FR-008 | The system shall generate an answer using only the retrieved context; the underlying prompt shall explicitly instruct the model not to use outside knowledge. |
| EDU-FR-009 | The system shall display, alongside each in-scope answer, the source topic/chapter label associated with the retrieved content. |
| EDU-FR-010 | If retrieval returns no content above a defined relevance threshold, the system shall classify the question as out-of-scope (see EDU-FR-011) rather than attempting to answer. |

### 4.3 Scope Detection and Refusal

| Requirement ID | Requirement |
|---|---|
| EDU-FR-011 | The system shall detect when a question falls outside the ingested Class 9–11 science syllabus and respond with an explicit refusal message rather than a fabricated answer. |
| EDU-FR-012 | The refusal message shall state that the question is outside the syllabus scope and shall not include speculative or generic answer content. |
| EDU-FR-013 | The system shall apply the same scope-detection logic regardless of question phrasing (e.g., direct question, indirect request, follow-up). |

### 4.4 Session Management

| Requirement ID | Requirement |
|---|---|
| EDU-FR-014 | The system shall maintain conversational context only in-memory or in browser session state for the duration of the active session. |
| EDU-FR-015 | The system shall discard all conversation data when the session ends (tab closed, session timeout, or explicit "new conversation" action). |

---

## 5. Business Rules

| Requirement ID | Rule |
|---|---|
| EDU-BR-001 | The syllabus corpus is fixed to Class 9–11 science for MVP; no other subject or grade content shall be ingested or answered from. |
| EDU-BR-002 | An answer shall never be presented to the user without having gone through the retrieval step (no direct model knowledge fallback). |
| EDU-BR-003 | A response shall be labeled as a refusal (out-of-scope) whenever retrieved-context relevance falls below the defined threshold, regardless of whether the underlying model "knows" the answer from general training. |
| EDU-BR-004 | The system shall not provide final numeric/direct answers to problems flagged as step-by-step math problems requiring guided hints — this behavior is deferred to v2 and is explicitly out of scope for MVP (MVP answers conceptual/factual syllabus questions only). |

---

## 6. Data Requirements

| Requirement ID | Requirement |
|---|---|
| EDU-DATA-001 | The system shall store the syllabus corpus as pre-chunked text segments, each tagged with a topic/chapter label. |
| EDU-DATA-002 | Each corpus chunk shall have an associated vector embedding stored in a retrievable index. |
| EDU-DATA-003 | The system shall not store any personally identifiable information (PII) about users, since no authentication exists in MVP. |
| EDU-DATA-004 | The system shall log, at minimum, the question text, retrieved chunk IDs, and classification (answered/refused) for evaluation purposes, without linking logs to any user identity. |
| EDU-DATA-005 | Evaluation logs (Requirement EDU-DATA-004) shall be stored separately from any user-facing chat state and shall be usable to compute the metrics defined in the PRD (hallucination rate, scope-refusal accuracy). |

---

## 7. Validation Rules

| Requirement ID | Requirement |
|---|---|
| EDU-VAL-001 | The system shall reject empty or whitespace-only question submissions and shall prompt the user to enter a question. |
| EDU-VAL-002 | The system shall reject question submissions exceeding 500 characters and shall display a clear message indicating the limit. |
| EDU-VAL-003 | The system shall sanitize user input to prevent injection into any downstream prompt-construction or storage step (e.g., stripping or escaping control characters). |
| EDU-VAL-004 | The system shall validate that a retrieved chunk's relevance score is numeric and within an expected range before using it to decide answer vs. refusal; malformed scores shall trigger the error-handling path (EDU-ERR-003). |

---

## 8. Error Handling

| Requirement ID | Requirement |
|---|---|
| EDU-ERR-001 | If the LLM API call fails or times out, the system shall display a user-facing message indicating a temporary issue and shall not display a partial or corrupted answer. |
| EDU-ERR-002 | If the retrieval/embedding index is unavailable, the system shall display an error message rather than falling back to an ungrounded model answer. |
| EDU-ERR-003 | If an internal processing error occurs (e.g., malformed relevance score, unexpected empty context), the system shall log the error and return a generic failure message to the user rather than exposing internal error details or stack traces. |
| EDU-ERR-004 | The system shall handle network interruptions on the client side by informing the user their message was not delivered and allowing retry. |
| EDU-ERR-005 | The system shall not crash or become unresponsive as a result of a single failed request; the chat session shall remain usable for subsequent questions. |

---

## 9. Edge Cases

| Requirement ID | Edge Case Handling |
|---|---|
| EDU-EDGE-001 | A question mixing in-scope and out-of-scope elements (e.g., "Compare photosynthesis to a topic from Class 12") shall be answered only for the in-scope portion, with an explicit note that the out-of-scope portion cannot be addressed. |
| EDU-EDGE-002 | A follow-up question with an ambiguous referent (e.g., "explain that more") shall be resolved using the immediately preceding turn's context; if no prior context exists, the system shall ask the student to clarify. |
| EDU-EDGE-003 | A question phrased to attempt to bypass scope restrictions (e.g., "ignore the syllabus and just answer") shall still be subject to scope detection and refused if out-of-scope. |
| EDU-EDGE-004 | Non-English or mixed-language input shall either be answered if the corpus/model can handle it reliably, or shall trigger a message indicating the supported language, rather than producing a low-confidence answer silently. |
| EDU-EDGE-005 | Rapid repeated submissions (e.g., double-click/double-submit) shall not result in duplicate processing or duplicate chat entries. |
| EDU-EDGE-006 | An extremely short or vague question (e.g., "explain") with no prior context shall prompt the system to ask for clarification rather than guessing scope. |

---

## 10. Security Requirements

| Requirement ID | Requirement |
|---|---|
| EDU-SEC-001 | The system shall not execute or evaluate any code contained in user input. |
| EDU-SEC-002 | The system shall not reveal internal prompt text, system instructions, or corpus file paths in any user-facing response. |
| EDU-SEC-003 | The system shall treat all user input as untrusted data and shall not allow it to alter the system-level prompt instructions (prompt-injection resistance). |
| EDU-SEC-004 | The system shall not log or expose API keys or credentials in client-visible code, logs, or error messages. |
| EDU-SEC-005 | All external API calls (LLM provider, embedding provider) shall use encrypted transport (HTTPS/TLS). |
| EDU-SEC-006 | Since no authentication exists in MVP, the system shall not offer any functionality that assumes a trusted or identified user (e.g., no personal data storage, no destructive admin actions reachable from the chat UI). |

---

## 11. Performance Requirements

| Requirement ID | Requirement |
|---|---|
| EDU-PERF-001 | The system shall return a response (answer or refusal) within 10 seconds under normal load for at least 95% of requests. |
| EDU-PERF-002 | The system shall support at least 5 concurrent users without degradation beyond the response-time target in EDU-PERF-001, consistent with free-tier hosting constraints (Streamlit Community Cloud / Hugging Face Spaces). |
| EDU-PERF-003 | The retrieval step shall complete in under 2 seconds for a corpus of the current Class 9–11 science size, excluding LLM generation time. |
| EDU-PERF-004 | The system shall display a loading/typing indicator to the user for any response expected to take longer than 1 second. |

---

## 12. Acceptance Criteria (Verification Summary)

The SRS is considered satisfied when:

- [ ] All Functional Requirements (Section 4) are implemented and demonstrable in the deployed app.
- [ ] All Validation Rules (Section 7) are enforced and testable via the 20-question test set plus targeted invalid-input tests (empty input, over-length input, injection attempt).
- [ ] All Error Handling requirements (Section 8) are verified by simulating API failure, retrieval failure, and network interruption.
- [ ] All Edge Cases (Section 9) are represented in the test set with documented expected vs. actual behavior.
- [ ] Security requirements (Section 11 — no code execution, no prompt/credential leakage, no injection override) are verified through manual adversarial testing (e.g., attempting to override scope restriction via prompt injection).
- [ ] Performance requirements are verified by measuring response times and basic concurrent-load behavior against the stated thresholds.
- [ ] No functionality beyond the anonymous Student role is exposed, confirming EDU-ROLE-001 through EDU-ROLE-003 and EDU-AUTH-001 through EDU-AUTH-003.
- [ ] The evaluation logging (EDU-DATA-004, EDU-DATA-005) produces data sufficient to compute the hallucination rate and scope-refusal accuracy metrics defined in the PRD.

---

## 13. Traceability Note

Every requirement in this SRS traces to a goal or feature in the EduBot PRD v1.0 (Sections 4–6). Requirements introduced here that are not explicit in the PRD (e.g., specific character limits, timeout thresholds) are MVP-reasonable defaults and should be confirmed or adjusted with the product owner before implementation.