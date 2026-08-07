# System Architecture — JobTrackr AI

## 1. High-Level Architecture Diagram (ASCII)

```
┌─────────────────────────────────────────────────────────────────────┐
│                              CLIENT (Browser)                        │
│  React 18 + TypeScript + Vite                                        │
│  - React Router (routing)      - Zustand (UI state: board, modals)   │
│  - React Query (server state)  - Context API (auth state)            │
│  - Axios (HTTP) + native WebSocket client                            │
└───────────────┬───────────────────────────────┬─────────────────────┘
                │ HTTPS (REST, JSON)             │ WSS (WebSocket)
                ▼                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     FastAPI Backend (Python, async)                  │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────────────────┐  │
│  │   Routers     │  │   Services    │  │   Agents (Agentic AI)   │  │
│  │ (HTTP layer)  │─▶│ (business     │─▶│  - Tool-calling loop     │  │
│  │ auth/boards/  │  │  logic)       │  │  - RAG retriever         │  │
│  │ applications/ │  │               │  │  - Anthropic API client  │  │
│  │ ai            │  │               │  │                          │  │
│  └───────┬───────┘  └───────┬───────┘  └────────────┬─────────────┘  │
│          │                  │                        │                │
│          ▼                  ▼                        ▼                │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │   Core: security (JWT/bcrypt), config (env), database (Motor)   │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│  WebSocket Connection Manager (broadcasts board changes per user)     │
└───────────────┬───────────────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     MongoDB (Atlas or self-hosted)                   │
│  Collections: users, boards, cards, resumes, ai_conversations        │
│  In-memory / file-backed vector index for RAG (see rag.py)           │
└─────────────────────────────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Anthropic API (Claude) — external                   │
│  Called by the agent module for both plain chat and tool-calling     │
└─────────────────────────────────────────────────────────────────────┘
```

## 2. Why This Stack (the "why", not just the "what" — this is what interviewers actually probe)

| Choice | Why (trade-offs considered) |
|---|---|
| **FastAPI over Flask/Django** | Native async support (matters for I/O-bound work like DB calls + external AI API calls happening concurrently), automatic OpenAPI docs, and Pydantic-based validation baked in — teaches modern Python idioms (type hints, `async`/`await`) that most 2026 backend job specs list explicitly. Django would be a better choice if we needed a built-in admin panel and heavier "batteries included" ORM; Flask would be better for something intentionally minimal. |
| **MongoDB over PostgreSQL** | Our core entity (a "card") has a naturally nested, evolving shape (notes, interview rounds, AI conversation history) that maps cleanly to a flexible document model. A relational DB would need several joined tables for the same data. Trade-off: we lose cross-collection transactional guarantees and joins — addressed via deliberate embedding-vs-referencing decisions (see `03_Database_Schema.md`). |
| **Motor (async MongoDB driver) over an ODM like Beanie** | We use Motor directly (not an ODM) *specifically for this learning project* so every database call is visible and explainable — an ODM is convenient in production but hides the exact query being run, which is bad for interview prep where you need to explain what's happening. |
| **JWT (stateless) auth over server-side sessions** | Scales horizontally without a shared session store; standard for SPA + API architectures. Trade-off: revocation before expiry is harder — we mitigate with short-lived access tokens (see `02_Architecture.md` §5). |
| **Zustand (UI state) + React Query (server state) instead of Redux for everything** | Server data (cards, boards) has its own lifecycle (fetch, cache, invalidate, refetch) that React Query is purpose-built for; Zustand handles genuinely local UI state (which modal is open, drag state) with far less boilerplate than Redux. This mirrors the real-world "split state by kind" pattern covered in your senior interview prep (advanced questionnaire Q13). |
| **WebSockets for board sync** | Board changes are bidirectional-feeling (multiple tabs/devices) and need low-latency push — a good, real use case for WebSockets rather than polling (directly maps to the advanced questionnaire's Q38 on WebSockets vs SSE vs polling). |
| **Hand-rolled agent loop instead of LangChain/LangGraph** | For a *learning* project, wiring the ReAct loop (reason → act → observe) ourselves in ~150 lines of Python teaches the mechanism that LangGraph abstracts away. In a real job, you'd likely use LangGraph for anything beyond a toy — the `docs/07_Production_Notes.md` file explains exactly how you'd swap this implementation for LangGraph in a real production system, so you can speak to both. |

## 3. Layered Backend Architecture

We use a classic **3-layer architecture** — the same pattern used in most real backend codebases, and a very common system-design interview topic:

```
Router (HTTP concerns: parsing requests, status codes, auth dependency)
   ↓
Service (business logic: "what does creating an application actually involve")
   ↓
Database (Motor async calls to MongoDB)
```

**Why layers matter (interview talking point):** a router should never contain business logic, and a service should never know about HTTP (no `Request`/`Response` objects) — this separation means you can unit-test business logic without spinning up a web server, and you can change the HTTP framework without rewriting business rules.

## 4. Real-Time Architecture (WebSockets)

- Each authenticated user, upon opening the board, opens **one WebSocket connection** to `/ws/board`.
- A `ConnectionManager` (see `backend/app/websocket_manager.py`) keeps an in-memory map of `user_id → [WebSocket connections]` (a user might have 2 tabs open).
- When any REST endpoint mutates a card (create/update/move/delete), the service layer calls `connection_manager.broadcast_to_user(user_id, event)` **after** the MongoDB write succeeds — the WebSocket push is a *side effect* of a successful write, not a replacement for it. This is a deliberate architectural choice: **REST is still the source of truth; WebSocket is purely a notification channel** telling other open tabs "go refetch" or "here's the diff." This avoids the classic bug of WebSocket and REST state diverging.

## 5. Authentication & Security Architecture

- **Passwords:** hashed with `bcrypt` (via `passlib`) — never stored or logged in plaintext.
- **Access token:** JWT, 30-minute expiry, signed with HS256 using a server-side secret (`SECRET_KEY` env var).
- **No refresh token in v1** (documented trade-off): to keep the learning scope focused, v1 requires re-login after 30 minutes. `docs/07_Production_Notes.md` explains exactly how you'd add a refresh-token rotation flow (httpOnly cookie) for a real production system — directly reusing the pattern from your advanced interview prep (Q43).
- **Authorization:** every card/board query is filtered by `owner_id == current_user.id` at the **database query level** (not just checked after fetching) — this is the Broken Object-Level Authorization (BOLA) prevention pattern from your advanced interview prep (Q41), applied for real here.
- **CORS:** explicit allow-list of frontend origins (no wildcard `*` in production).

## 6. Agentic AI Architecture

Three distinct mechanisms, each individually explainable:

1. **Tool-calling loop** (`backend/app/agents/agent.py`): implements the ReAct pattern (Thought → Action → Observation) manually using Claude's native tool-use API. Tools: `get_interview_questions(role, level)`, `search_resume(query)` (RAG-backed), `get_company_overview(company_name)` (pluggable — mocked with a static lookup + clearly marked extension point for a real web-search API).
2. **RAG** (`backend/app/agents/rag.py`): the user's resume text is chunked, embedded (via Anthropic's embeddings or a local sentence-transformers model — see code comments for the trade-off), and stored in a simple **in-memory cosine-similarity index** (no external vector DB needed at this scale — this is itself an interview talking point: know when you *don't* need Pinecone/Chroma).
3. **Streaming responses**: the chat endpoint streams tokens back to the client via Server-Sent Events, exactly matching the pattern in your advanced interview prep (Q50).

## 7. Deployment Architecture

```
GitHub repo
   │  (push to main)
   ▼
GitHub Actions CI  →  runs backend pytest + frontend vitest + lint
   │  (on success, on main branch)
   ▼
Backend: Docker image → pushed to a container registry → deployed to Render/Railway/Fly.io
Frontend: Vite production build (static files) → deployed to Vercel/Netlify/S3+CloudFront
MongoDB: Atlas (managed, separate from app deployment)
```

See `06_Roadmap_Sprints.md` for the sprint in which each piece gets built, and `07_Production_Notes.md` for what changes if this were a real multi-team org project (multi-tenancy, refresh tokens, LangGraph, a real vector DB, rate limiting, observability).

## 8. Scaling Considerations (explicitly, since this is a common senior-interview follow-up)

- **Read-heavy board queries** → add a MongoDB index on `{owner_id: 1, board_id: 1}` (see `03_Database_Schema.md`) — already included in v1 since it's cheap and correct from day one, not deferred.
- **WebSocket horizontal scaling** → v1 keeps connections in a single process's memory, which is fine for one backend instance. If we ever ran multiple backend instances behind a load balancer, we'd need a shared pub/sub layer (Redis) so a broadcast from instance A reaches a user connected to instance B — documented as a v2 concern, not built now, to keep the learning scope focused.
- **AI cost/latency** → the agent loop caps tool-calling iterations at `MAX_AGENT_STEPS = 5` to prevent runaway loops/cost (directly reusing the advanced questionnaire's Q49 guardrail pattern).
