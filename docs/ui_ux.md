# UI/UX Design Document: EduBot

**Version:** 1.0 (MVP)
**Status:** Draft for Review
**Owner:** Abhishek
**Related documents:** EduBot PRD v1.0, SRS v1.0, Architecture v1.0

---

## 1. Design Principles

1. **Clarity over decoration** — this is a study tool; every visual choice should reduce friction to reading and understanding an answer, not compete for attention.
2. **One screen, one job** — MVP is a single chat screen. Resist adding navigation, tabs, or settings that aren't required by the SRS.
3. **Trust through transparency** — since grounding/refusal is a core feature, the UI must make it visually obvious whether an answer is grounded, refused, or still loading. No ambiguous states.
4. **Calm, not clinical** — a student-facing study tool should feel approachable (soft rounded corners, warm neutral palette) rather than looking like a raw developer console.
5. **Low cognitive load** — minimal chrome, generous whitespace, restrained color usage reserved for meaningful states (answer vs. refusal vs. error).

---

## 2. User Journey

```
Land on app → Read short intro/instructions → Type a question → 
Send → See loading indicator → Receive answer (with source label) 
   OR receive refusal (out-of-scope) OR see error →
Ask follow-up (same session) → 
Optionally start "New conversation" → Repeat
```

Single persona (Student, anonymous), single primary journey — no branching by role since MVP has only one role (per SRS Section 2).

---

## 3. Navigation

MVP has **no multi-page navigation**. It is a single-screen application.

| Element | Behavior |
|---|---|
| App header | Static; shows app name/logo and a brief one-line description of scope ("Ask about your Class 9–11 science syllabus"). |
| "New conversation" control | Top-right of header or chat panel; clears current session's chat state (per EDU-FR-005). |
| No sidebar, no settings, no login | Consistent with SRS: no auth, no user accounts, no admin features exposed. |

**Future (not built now):** if v2 (Socratic hints) or v3 (guardrail toggles) are added, they may warrant a lightweight tab or mode-switcher — not designed here to avoid over-scoping MVP.

---

## 4. Screens

### 4.1 Screen: Chat Home (the only screen)

**Layout regions (top to bottom):**

1. **Header bar** — app name, one-line scope description, "New conversation" button.
2. **Chat panel** — scrollable message list (student questions right-aligned, bot responses left-aligned).
3. **Composer** — text input + send button, pinned to the bottom of the viewport.

```
┌──────────────────────────────────────────────┐
│  EduBot   Ask about Class 9–11 Science  [New] │  ← Header
├──────────────────────────────────────────────┤
│                                                │
│                          [Your question]  🧑  │  ← User bubble
│  🤖  [Bot answer]                             │  ← Bot bubble
│      📘 Source: Chapter 4 – Motion            │  ← Source tag
│                                                │
│                          [Follow-up]      🧑  │
│  🤖  ⚠ This is outside your syllabus scope.   │  ← Refusal state
│                                                │
├──────────────────────────────────────────────┤
│  [ Type your question...           ] [ Send ] │  ← Composer
└──────────────────────────────────────────────┘
```

### 4.2 Empty State (first load, no messages yet)

- Centered placeholder in the chat panel:
  - Short friendly headline: "Ask me anything from your Class 9–11 science syllabus."
  - 2–3 example question chips (tappable, pre-fill the composer): e.g., "What is Newton's Third Law?", "Explain photosynthesis."
  - No source tag, no timestamp — this is a pre-conversation state, not a message.

---

## 5. User Flows

### 5.1 Ask a question (happy path)
1. Student types in composer → clicks Send (or presses Enter).
2. User message bubble appears immediately (optimistic render).
3. Loading indicator appears in place of the bot's next bubble.
4. Backend responds → loading indicator replaced with either:
   - Answer bubble + source tag, or
   - Refusal bubble (distinct visual treatment).
5. Composer clears and refocuses for the next input.

### 5.2 Follow-up question
1. Same as 5.1, but the request includes prior 1–2 turns (handled server-side per architecture doc; no UI change needed).
2. If the system cannot resolve an ambiguous follow-up (EDU-EDGE-002), it returns a clarification-request bubble instead of an answer — rendered with the same visual treatment as a refusal but different copy (see Section 8).

### 5.3 Start a new conversation
1. Student clicks "New conversation."
2. Confirmation is **not** required (no destructive/irreversible data — nothing is saved anyway) — clear immediately.
3. Chat panel resets to the Empty State (Section 4.2).

### 5.4 Error recovery
1. Request fails (timeout, network, server error).
2. Loading indicator is replaced with an inline error bubble + a "Retry" button.
3. Clicking Retry re-sends the same question without requiring the student to retype it.

---

## 6. Components

| Component | States | Notes |
|---|---|---|
| **Message bubble (user)** | default | Right-aligned, filled with primary-light background. |
| **Message bubble (bot – answer)** | default | Left-aligned, neutral background, includes source tag below text. |
| **Message bubble (bot – refusal)** | default | Left-aligned, distinct left border/icon (amber), copy per Section 8. |
| **Message bubble (bot – error)** | default, with retry | Left-aligned, distinct left border/icon (red), includes Retry button. |
| **Source tag** | default | Small pill/label under an answer bubble, e.g., "📘 Chapter 4 – Motion." |
| **Composer input** | default, focused, disabled (while awaiting response), error (over char limit) | Disabled state prevents double-submit (EDU-EDGE-005). |
| **Send button** | default, disabled (empty input or awaiting response), loading | Mirrors composer state. |
| **Character counter** | default, warning (near limit), error (over limit) | Appears only when approaching the 500-char limit (e.g., last 50 chars). |
| **Example question chip** | default, hover/pressed | Only shown in Empty State; clicking fills composer, does not auto-send. |
| **"New conversation" button** | default, hover/pressed | Always visible in header. |
| **Loading indicator** | typing-dots or skeleton bubble | Shown per EDU-PERF-004 for any wait >1s. |
| **Toast (transient, optional)** | e.g., "Message not delivered" | Used only for connectivity-level issues, not for scope refusals (which are conversational, not transient). |

---

## 7. Interactions

- **Send question:** Enter key or Send button; Shift+Enter inserts a newline (if multi-line input is supported) — otherwise Enter always sends for a single-line composer.
- **Disabled state while waiting:** composer and send button are disabled between submit and response to prevent duplicate submissions (maps to EDU-EDGE-005).
- **Auto-scroll:** chat panel auto-scrolls to the latest message on new content; if the student has scrolled up to review history, auto-scroll pauses until they scroll back to bottom (or a "Jump to latest" pill appears).
- **Example chip click:** fills composer text but requires an explicit Send action — never auto-sends on behalf of the student.
- **Copy answer (optional, low-effort add):** small copy icon on bot answer bubbles for students who want to paste into notes.

---

## 8. Forms and Input Validation (Composer)

| Condition | UI feedback |
|---|---|
| Empty submit attempt | Send button stays disabled; no error toast needed — absence of affordance is the signal. |
| Approaching 500-char limit | Counter appears, turns amber at ~450 chars. |
| Exceeds 500-char limit | Counter turns red; Send button disables; inline message: "Please shorten your question (500 character limit)." |
| Submission while awaiting prior response | Composer and Send button disabled; no new input accepted until current turn resolves. |

**Refusal / clarification copy (bot bubble text, not a form validation, but content-critical):**
- Out-of-scope: "That's outside your Class 9–11 science syllabus, so I can't answer it here."
- Ambiguous follow-up: "Could you clarify what you're referring to? I want to make sure I answer the right question."
- Mixed-scope question: "I can answer the part about [in-scope topic], but the rest is outside your syllabus."

---

## 9. Loading, Error, and Empty States

| State | Trigger | Visual treatment |
|---|---|---|
| **Empty** | First load / after "New conversation" | Centered headline + example chips (Section 4.2). |
| **Loading** | Awaiting backend response | Typing-dots animation inside a placeholder bot bubble; composer disabled. |
| **Answer** | Successful, in-scope response | Standard bot bubble + source tag. |
| **Refusal** | Out-of-scope classification | Bot bubble with amber accent + icon, no source tag. |
| **Clarification** | Ambiguous follow-up | Bot bubble with neutral/blue accent, question-mark icon. |
| **Error (recoverable)** | API timeout, network drop | Bot bubble with red accent, error icon, Retry button. |
| **Error (session-breaking)** | Vector store/backend fully unavailable | Full-panel banner above composer: "EduBot is temporarily unavailable. Please try again shortly." Composer disabled entirely in this case. |

---

## 10. Responsive Behavior

Single breakpoint strategy — this is a chat UI, which is inherently responsive-friendly:

| Breakpoint | Behavior |
|---|---|
| **Mobile (<600px)** | Full-width chat panel; header collapses app name + icon-only "New conversation" button; composer pinned to bottom with safe-area padding. |
| **Tablet (600–1024px)** | Chat panel max-width ~700px, centered; header shows full text labels. |
| **Desktop (>1024px)** | Chat panel max-width ~800px, centered with generous side margins; no multi-column layout needed (avoids over-engineering a single-purpose chat screen). |

- Message bubbles: max-width 75% of chat panel width at all breakpoints, to preserve readability (avoid full-width text blocks on desktop).
- Composer: full width of chat panel at all breakpoints, sticky to viewport bottom on mobile.

---

## 11. Accessibility

| Requirement | Implementation |
|---|---|
| Keyboard navigation | Composer, Send, "New conversation," and example chips are all reachable and operable via Tab/Enter. |
| Screen reader support | Each new bot message is announced via an `aria-live="polite"` region so screen reader users are notified of new responses without needing to poll. |
| Color contrast | All text/background pairs meet WCAG AA (4.5:1 for body text, 3:1 for large text/icons). |
| Non-color state indication | Refusal/error/answer states are distinguished by icon + text, not color alone (for colorblind users). |
| Focus management | Focus returns to the composer input after each response is rendered, so keyboard/screen-reader users can immediately type a follow-up. |
| Touch target size | All interactive elements (buttons, chips) minimum 44×44px tap area on mobile. |
| Text resizing | Layout remains usable at up to 200% browser zoom without horizontal scrolling or clipped text. |

---

## 12. Typography

| Use | Font | Weight | Size (desktop) |
|---|---|---|---|
| App name/header | System UI sans-serif (e.g., Inter, or platform default) | Semi-bold (600) | 18px |
| Message body text | Same family | Regular (400) | 16px |
| Source tag / metadata | Same family | Medium (500) | 13px |
| Character counter / helper text | Same family | Regular (400) | 12px |
| Buttons | Same family | Medium (500) | 14px |

Line height: 1.5 for message body text (readability for longer explanations). Avoid decorative or serif fonts — keep it plain and legible, consistent with Principle 1 (clarity over decoration).

---

## 13. Colors

A restrained palette; color is reserved for meaningful state, not decoration.

| Token | Purpose | Example (light mode) |
|---|---|---|
| `--color-bg` | App background | Off-white (#FAFAFA) |
| `--color-surface` | Card/panel background | White (#FFFFFF) |
| `--color-text-primary` | Main text | Near-black (#1A1A1A) |
| `--color-text-secondary` | Metadata, helper text | Medium gray (#6B7280) |
| `--color-primary` | Primary actions (Send button, user bubble accent) | Indigo/blue (#4F46E5) |
| `--color-user-bubble` | User message background | Light indigo tint (#EEF2FF) |
| `--color-bot-bubble` | Bot answer background | Light gray (#F3F4F6) |
| `--color-success/answer-accent` | Grounded-answer indicator (subtle, e.g., source tag) | Green-gray (#10B981, used sparingly) |
| `--color-warning/refusal-accent` | Refusal state | Amber (#F59E0B) |
| `--color-error` | Error state | Red (#EF4444) |
| `--color-border` | Dividers, input borders | Light gray (#E5E7EB) |

Dark mode (optional, future): invert surface/background, keep accent colors roughly consistent with adjusted lightness for contrast compliance.

---

## 14. Spacing

8px base spacing unit, consistent across the single screen:

| Token | Value | Use |
|---|---|---|
| `--space-xs` | 4px | Icon-to-text gaps |
| `--space-sm` | 8px | Inline element spacing (e.g., counter to input) |
| `--space-md` | 16px | Padding inside bubbles, between form elements |
| `--space-lg` | 24px | Padding around chat panel edges |
| `--space-xl` | 32px | Spacing between header and chat panel; empty-state vertical spacing |

Message bubble internal padding: 12px vertical, 16px horizontal. Gap between consecutive messages: 12px. Gap between message groups (user turn + bot turn): 20px.

---

## 15. Implementation Notes (Streamlit-specific, per Architecture doc)

Since the recommended stack is Streamlit for MVP:
- Use `st.chat_message()` and `st.chat_input()` primitives, which natively support the bubble layout, auto-scroll, and composer-pinning described above — avoid custom HTML/CSS chat implementations unless a specific state (e.g., refusal accent color) isn't achievable with native theming.
- Use Streamlit's theming config (`.streamlit/config.toml`) to set the color tokens in Section 13 as the app's primary/background colors, keeping bot-message color variants (answer/refusal/error) handled via conditional icon + caption text rather than fighting Streamlit's default bubble styling.
- Loading state: Streamlit's built-in spinner (`st.spinner`) or a simple typing-dots placeholder rendered before the response streams in.
- If richer custom styling becomes necessary later (e.g., true colored bubble borders for refusal/error), inject scoped CSS via `st.markdown(..., unsafe_allow_html=True)` — keep this minimal and isolated so it doesn't turn into an unmaintainable style layer.

---

## 16. Acceptance Criteria

- [ ] Single-screen chat interface implemented with header, scrollable chat panel, and pinned composer.
- [ ] Empty state shows headline + example chips; chips fill but don't auto-send.
- [ ] All defined states (loading, answer, refusal, clarification, recoverable error, session-breaking error) are visually distinct and implemented per Section 9.
- [ ] Composer enforces 500-character limit with counter and disables Send appropriately (Section 8).
- [ ] Composer and Send are disabled while awaiting a response, preventing duplicate submits.
- [ ] Layout is verified at mobile, tablet, and desktop breakpoints per Section 10.
- [ ] Keyboard-only navigation and screen-reader announcements verified per Section 11.
- [ ] Color contrast verified at WCAG AA for all text/background combinations in Section 13.
- [ ] Retry action on recoverable errors re-sends the original question without requiring retyping.
- [ ] "New conversation" clears chat state immediately and returns to the Empty State.