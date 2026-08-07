# Roadmap & Sprint Plan — JobTrackr AI

This is structured as a real Agile team would plan it — useful both as a build order for you, and as material for "walk me through how you'd plan this project" system-design/behavioral interview questions.

## Sprint 0 — Setup & Foundations (this delivery)
- [x] Requirements (`01_PRD.md`), architecture (`02_Architecture.md`), DB schema (`03_Database_Schema.md`), API design (`04_API_Design.md`), frontend architecture (`05_Frontend_Architecture.md`).
- [x] Repo scaffolding: backend (`FastAPI` + `Motor`) and frontend (`React` + `TS` + `Vite`) project skeletons.
- [x] Core cross-cutting concerns: config/env handling, DB connection, JWT security utilities, CORS.
- **Definition of Done:** Both apps boot locally (`uvicorn` / `npm run dev`) with a health-check endpoint returning 200.

## Sprint 1 — Auth (Must Have)
- [x] Backend: `User` model, register/login/me endpoints, password hashing, JWT issuance and verification dependency.
- [x] Frontend: `AuthContext`, `LoginForm`, `RegisterForm`, `ProtectedRoute`, Axios interceptor attaching the JWT to every request.
- **Definition of Done:** A new user can register, log in, land on a protected dashboard, and get redirected to `/login` if the token is missing/expired. Backend tests cover: duplicate email rejected, wrong password rejected, valid login issues a token.

## Sprint 2 — Board & Cards CRUD (Must Have)
- [x] Backend: `Board`/`Card` models, board auto-creation on first access, full card CRUD, ownership-scoped queries (BOLA prevention).
- [x] Frontend: `KanbanBoard`, `Column`, `ApplicationCard`, `CardModal` (create/edit form), React Query hooks (`useApplications`), drag-and-drop with optimistic updates.
- **Definition of Done:** A user can create, edit, delete, and drag cards between columns; refreshing the page preserves state; another user cannot see or modify this user's cards even with a guessed ID (tested explicitly).

## Sprint 3 — Real-Time Sync (Must Have)
- [x] Backend: `ConnectionManager`, `/ws/board` endpoint, broadcast-after-write in the card service.
- [x] Frontend: `useWebSocket` hook, wiring board refetch on incoming events.
- **Definition of Done:** Two browser tabs open to the board; moving a card in tab A updates tab B within ~1 second with no manual refresh.

## Sprint 4 — Agentic AI Coach (Must Have)
- [x] Backend: resume upload + chunking + embedding (`rag.py`), tool definitions (`tools.py`), ReAct agent loop (`agent.py`), streaming chat endpoint.
- [x] Frontend: `AIAssistantPanel`, `ChatMessage`, SSE consumption with incremental rendering.
- **Definition of Done:** Uploading a resume and asking a scoped question produces a grounded answer that references resume content; the agent correctly chooses to call a tool when the question needs one (e.g., "what questions might they ask a senior React dev") and answers directly when it doesn't need a tool.

## Sprint 5 — Dashboard & Polish (Must Have)
- [ ] Frontend: `StatsPanel` (applications by status, response-rate chart via `recharts`).
- [ ] Backend: an aggregation-pipeline endpoint computing those stats server-side (not client-side — a deliberate choice; see code comments in `services/application_service.py` on why aggregation belongs in the database layer at scale).
- **Definition of Done:** Dashboard loads in under 300ms with realistic data (50+ cards).

## Sprint 6 — Testing & CI (Must Have)
- [ ] Backend: `pytest` unit tests for services (business logic) + integration tests for routers (using `httpx.AsyncClient` against a test DB).
- [ ] Frontend: `Vitest` + `React Testing Library` for `KanbanBoard` drag logic and `LoginForm` validation.
- [ ] GitHub Actions workflow running both test suites on every PR.
- **Definition of Done:** CI is green on `main`; coverage report generated.

## Sprint 7 — Containerization & Deployment (Must Have)
- [ ] `Dockerfile` for backend (multi-stage: install deps → copy app → run with `uvicorn`/`gunicorn`).
- [ ] `Dockerfile` + Nginx config for frontend (multi-stage: `npm run build` → serve static files via Nginx).
- [ ] `docker-compose.yml` wiring backend + frontend + a local MongoDB for full local parity with production.
- [ ] Deployment: backend container → Render/Railway/Fly.io; frontend static build → Vercel/Netlify; MongoDB → Atlas.
- [ ] Environment variable management: `.env.example` documented, real secrets never committed, injected via the hosting platform's secret manager.
- **Definition of Done:** A public URL serves the working app end-to-end against production infrastructure.

## Sprint 8 (Stretch / v1.1) — Should-Have features
- [ ] Follow-up reminders.
- [ ] CSV export.
- [ ] Interview scheduling view.

---

## How to Talk About This Roadmap in an Interview

A common senior-level question is *"walk me through how you'd plan and ship a feature/project from scratch."* Your answer, using this exact roadmap as your example:
1. **Start with requirements and a written PRD** — forces you to define Must/Should/Could/Won't before writing code, preventing scope creep.
2. **Design the architecture and data model before the UI** — the data shape (see `03_Database_Schema.md`'s embedding-vs-referencing decisions) determines what's easy or hard to build later; UI is comparatively cheap to change.
3. **Build vertically (thin slices through the whole stack), not horizontally (all backend, then all frontend)** — notice each sprint above delivers a complete, demoable feature end-to-end (backend + frontend + a Definition of Done you can actually click through), not "finish the whole API" then "finish the whole UI." This is a deliberate, defensible choice you can explain: vertical slices surface integration problems early, when they're cheap to fix.
4. **Testing and deployment are their own sprints, not an afterthought "if there's time" — but they also aren't done first**, because there's nothing meaningful to test/deploy yet in Sprint 0. Ordering deployment concerns AFTER the feature is real (rather than "deploy on day one") is itself a defensible sequencing choice worth being able to explain.
