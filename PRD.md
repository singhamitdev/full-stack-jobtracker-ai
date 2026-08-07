# Product Requirements Document (PRD)
## JobTrackr AI — Kanban Job Application Tracker with an AI Interview Coach

**Document owner:** You (Product Engineer)
**Status:** Approved for build
**Version:** 1.0

---

## 1. Problem Statement

Job seekers juggling multiple applications lose track of where each application stands (applied → interviewing → offer → rejected), forget to follow up, and walk into interviews under-prepared because researching each company and rehearsing answers is time-consuming and unstructured.

**JobTrackr AI** solves this with:
1. A **Kanban board** to visually track every application through its pipeline stage.
2. An **AI Interview Coach** — an agentic assistant that can search for company information, generate tailored interview questions, and answer questions about an uploaded resume/job description using **RAG** (Retrieval-Augmented Generation).

This mirrors exactly the kind of product a real company would build (internal tools, CRMs, ATS systems all share this "board + records + AI assistant" shape), which is why it's a strong interview-prep project: the patterns transfer directly to real job requirements.

---

## 2. Target Users (Personas)

| Persona | Description | Key Need |
|---|---|---|
| **Active job seeker** | Applying to 10-50+ roles over a few months | Single source of truth for pipeline status |
| **Career switcher** | Fewer applications, but each needs deep prep | AI-assisted interview prep per company |
| **You (the builder)** | Learning full-stack + AI engineering for interviews | A codebase that *teaches* every concept it uses |

---

## 3. Goals & Non-Goals

**Goals**
- Ship a working, deployable product covering the full stack: React/TS frontend, FastAPI/Python backend, MongoDB, JWT auth, WebSockets, and an agentic AI feature with tool-calling + RAG.
- Every non-trivial piece of code is commented to explain **what** it does, **why** it's written that way, and **what interview question it maps to.**
- Follow patterns a real engineering org would follow: layered architecture, typed contracts, tests, CI/CD, containerized deployment.

**Non-Goals (explicitly out of scope for v1)**
- Payments/billing (not needed for this product).
- Native mobile apps (responsive web only).
- Multi-tenant "company" accounts — this is a single-user personal tool (though the architecture is designed so multi-tenancy could be added later — see `02_Architecture.md`).

---

## 4. Feature List (MoSCoW Prioritization)

### Must Have (v1 — this build)
- **Auth:** Register, login, JWT-based sessions, protected routes.
- **Kanban Board:** Columns (Wishlist → Applied → Interviewing → Offer → Rejected), drag-and-drop cards between columns.
- **Application Cards:** Create/edit/delete a job application (company, role, link, notes, dates, salary range, status).
- **Real-time sync:** If you have the board open in two tabs, moving a card in one updates the other instantly (WebSockets).
- **AI Interview Coach (agentic):**
  - Chat interface scoped to a specific application.
  - The agent can call **tools**: look up general company info, generate interview questions by role/level, and search the user's own uploaded resume content.
  - **RAG:** upload a resume (text) once; the agent retrieves relevant chunks of it when answering "How should I frame my experience with X for this role?"-style questions.
- **Dashboard:** Aggregate stats (applications by status, response rate, applications-per-week trend).

### Should Have (v1.1 — roadmap, scaffolded but not fully built)
- Email/browser reminders for follow-ups.
- Interview scheduling with calendar view.
- Export board to CSV/PDF.

### Could Have (v2+)
- Browser extension to "clip" a job posting directly into the board.
- Multi-user shared boards (mentor/mentee).

### Won't Have (v1)
- Native mobile app.
- Payments.

---

## 5. User Stories (sample — the full backlog lives in `06_Roadmap_Sprints.md`)

- *As a job seeker*, I want to add a new application card with company/role/link so I can track it without leaving a spreadsheet mindset behind.
- *As a job seeker*, I want to drag a card from "Applied" to "Interviewing" so the board always reflects reality at a glance.
- *As a job seeker*, I want to ask the AI coach "What should I expect in a system design round at this company?" and get a grounded, useful answer.
- *As a job seeker*, I want the AI coach to reference my actual resume when suggesting how to answer "Tell me about a time you..." questions.
- *As a returning user*, I want my board to load exactly as I left it, synced across devices.

---

## 6. Success Metrics (how a real PM would measure this)

| Metric | Target |
|---|---|
| Time to add a new application | < 15 seconds |
| API p95 latency (non-AI endpoints) | < 200ms |
| AI Coach response time to first token | < 2s (streaming) |
| Test coverage (backend core logic) | > 80% |
| Uptime (post-deployment) | 99.5% |

---

## 7. Constraints & Assumptions

- Single deployable Python backend (FastAPI) + single deployable frontend (static build served via CDN/Nginx).
- MongoDB Atlas free/shared tier is sufficient for this scale (no sharding needed yet — see scaling notes in `02_Architecture.md`).
- The AI Coach uses the Anthropic API (Claude) as the underlying model; tool-calling and RAG are implemented by us, not delegated to a heavy framework, specifically so every mechanism is visible and explainable in an interview.
