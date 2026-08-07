# Development Workflow Guide — How This Project Was Actually Built, File by File

**Purpose of this document:** Every other doc in `docs/` explains the FINAL design. This
one explains the **build order** — which file to write first, why it had to come before
the next one, what logic/technology each file introduces, and how the files connect to
each other. Read this alongside the actual code with two windows open: this guide on one
side, the file it's describing on the other.

By the end, you shouldn't just understand *this* project — you should be able to
replicate the same file-by-file reasoning process to build your OWN next feature, or your
own next project from scratch. The last section of this guide ("How to Repeat This
Process Yourself") makes that generalizable checklist explicit.

**If you are starting from a genuinely empty folder** (nothing created yet, not even
`git init`), start at **Part 0** below — it covers everything between "empty VS Code
window" and "the first application file is now the right thing to write." Parts 1 and 2
then pick up exactly where Part 0 leaves off.

---

## The Core Principle: Build Bottom-Up Within Each Vertical Slice

You can't write a function that depends on another function that doesn't exist yet. So
within any one feature, the build order is always:

```
Foundations (config/security/db) → Data shape (models) → API contract (schemas)
   → Business logic (services) → HTTP wiring (routers) → App assembly (main.py)
```

...and on the frontend, the mirror image:

```
Types (contract) → How to talk to the API (api/) → Where state lives (context/store/hooks)
   → Small reusable pieces (common components) → Feature components → Pages → App.tsx
```

We build **one full vertical slice** (e.g., "Auth," end-to-end, backend AND frontend)
before moving to the next slice (e.g., "Boards & Cards"), rather than building all
backend first, then all frontend — see `docs/06_Roadmap_Sprints.md` for why that ordering
choice matters at the sprint level. This guide operates at the file level, one layer
below that.

---

## PART 0 — Before `config.py`: From an Empty Folder to "Ready to Write Code"

This is the part most guides skip, and it's exactly where you asked to start: **you have
an empty folder open in VS Code. Nothing else exists yet.** Everything below happens in
your terminal (VS Code's integrated terminal is fine — `` Ctrl+` `` / `` Cmd+` ``) BEFORE
a single application file is written. Nothing here is "application logic" — it's the
scaffolding that has to exist so application logic has somewhere to live and something to
run on.

### 0.1 — Confirm your machine actually has the right tools installed

Run each of these in your terminal. If any command fails/isn't found, install that tool
before going further — there's no point planning a Python backend if Python itself isn't
installed.

```bash
python3 --version     # need 3.12+ for this project
node --version        # need 20+
npm --version         # comes bundled with Node
git --version
docker --version      # optional, but strongly recommended — see 0.5 below
```

**Why check this first, before creating a single folder:** every later step assumes these
exist. Discovering halfway through "oh, my Python is 3.9" is a much more annoying place
to find that out than right now.

**VS Code extensions worth installing now** (Extensions panel, `Ctrl+Shift+X`): *Python*
and *Pylance* (Microsoft) for backend work, *ESLint* and *Prettier* for frontend work.
None of this is required to write code — VS Code works without them — but they give you
real-time red-squiggle feedback on the exact kinds of mistakes this project's comments
warn about (an unused import, a type mismatch), which speeds up learning considerably.

### 0.2 — Create the project root and put it under version control

```bash
mkdir jobtrackr-ai
cd jobtrackr-ai
git init
```

**Why `git init` before ANY code exists, not after:** every single change from this point
forward — including the very first file — can now be committed. Starting version control
late means your first several hours of work have no history to fall back on if something
breaks. This is also the point where you'd create a GitHub repo (`gh repo create`, or via
github.com) and add it as a remote, if you want to push as you go — entirely optional for
a solo learning project, but it's how the CI workflow from `docs/06_Roadmap_Sprints.md`
Sprint 6 eventually gets triggered.

Add a `.gitignore` right away too (even a nearly-empty one to start) — otherwise your
FIRST `git add .` later risks accidentally staging a virtual environment or
`node_modules`, which you really don't want committed. You can start with just:
```
.env
__pycache__/
node_modules/
venv/
```
...and add more lines to it as new tools introduce new things to ignore (we'll add
`dist/`, `.pytest_cache/`, etc. later — see the full version in the project root).

### 0.3 — Decide the top-level shape of the repo

```bash
mkdir backend frontend docs
```

This single command is the FIRST real architectural decision of the whole project: a
**monorepo** with backend and frontend as siblings (see `docs/02_Architecture.md` for why
this shape, vs. two entirely separate repos). Everything else — every file this guide
describes — lives inside one of these three folders.

**Do the planning docs (`docs/01_PRD.md` through `docs/07_Production_Notes.md`) before
writing ANY code, including config.py.** This project's actual build order was: write the
requirements and architecture docs FIRST (while the `docs/` folder was still the only
non-empty thing in the repo), and only start touching `backend/` or `frontend/` once
there was a written plan to build against. If you're following along, this is the point
to write (or adapt) your own PRD and architecture doc — even a rough one — before
continuing to 0.4.

### 0.4 — Initialize the backend as a real, isolated Python project

```bash
cd backend

# Create a VIRTUAL ENVIRONMENT — an isolated copy of Python + its own package
# folder, separate from anything else on your machine. WHY THIS MATTERS: without it,
# every Python project on your computer would share ONE global set of installed
# packages, so two projects needing different versions of the same library would
# conflict. A venv gives THIS project its own sandbox.
python3 -m venv venv

# Activate it — your terminal prompt should now show "(venv)" at the start of the line,
# confirming `pip install` will install INTO this project's sandbox, not globally.
source venv/bin/activate        # macOS/Linux
venv\Scripts\activate           # Windows

# NOW install the packages this project needs. At this exact point in a real build, you
# often don't know the FULL list yet — you add packages as you discover you need them.
# For this project, we know the full list upfront (see backend/requirements.txt), so
# install it all now:
pip install fastapi "uvicorn[standard]" motor pydantic pydantic-settings \
            "python-jose[cryptography]" "passlib[bcrypt]" python-multipart \
            anthropic sentence-transformers \
            pytest pytest-asyncio httpx mongomock-motor

# Freeze the EXACT installed versions into requirements.txt, so anyone else (or you, on
# a different machine) can recreate this exact environment with `pip install -r
# requirements.txt` instead of guessing at versions.
pip freeze > requirements.txt
```

**Why a venv is created before a single `.py` file exists:** if you write `config.py`
first and it does `from pydantic_settings import BaseSettings`, that import will fail
with `ModuleNotFoundError` the instant you try to run anything — the library has to
already be installed for the code that imports it to mean anything at all.

Now create the folder skeleton the rest of this guide assumes already exists:

```bash
mkdir -p app/core app/models app/schemas app/routers app/services app/agents tests
```

And create empty **package marker files** — this is a Python-specific requirement worth
understanding, not just a rote step:

```bash
touch app/__init__.py app/core/__init__.py app/models/__init__.py app/schemas/__init__.py
touch app/routers/__init__.py app/services/__init__.py app/agents/__init__.py tests/__init__.py
```

**Why `__init__.py` files (even completely empty ones):** an `__init__.py` file is what
tells Python "treat this folder as an importable package." Without `app/core/__init__.py`
existing, a later line like `from app.core.config import settings` would fail — Python
wouldn't recognize `app.core` as something it can import FROM at all, regardless of
whether `config.py` inside it is correct. This is the Python equivalent of why a
JavaScript/TypeScript project needs a `package.json` for `npm` to recognize a folder as a
real package — a small piece of ceremony that has to exist before the "real" files can be
meaningfully imported by each other.

Finally, create a placeholder `.env.example` (even mostly empty — you'll add a line to it
every time `config.py` grows a new setting, rather than writing the whole thing upfront
and hoping you guessed every variable correctly):
```bash
touch .env.example
```

**At this point, `backend/` has a fully working Python environment and folder skeleton,
but ZERO application code.** This is the exact moment `core/config.py` (Part 1, Step 1)
becomes the right next file to write — everything it needs (an installed
`pydantic-settings`, a folder to live in that Python recognizes as a package) already
exists.

### 0.5 — Get MongoDB running (you need somewhere for data to eventually go)

You don't need this the SECOND you start writing code, but you'll need it by the time you
write `core/database.py` (Part 1, Step 3), so set it up now rather than getting blocked
later. Two options:

```bash
# Option A: Docker (recommended — matches docker-compose.yml exactly, no local install)
docker run -d --name jobtrackr-mongo -p 27017:27017 mongo:7

# Option B: MongoDB Atlas (a free-tier cloud cluster, no local install at all) —
# sign up at mongodb.com/atlas, create a free cluster, and copy its connection string
# for later use in your .env file's MONGODB_URL.
```

**Why set this up now, before writing any backend code, rather than after:** `core/
database.py` will try to CONNECT to whatever `MONGODB_URL` points to the first time the
app starts — if nothing is listening at that address, you'll get a confusing connection
error that has nothing to do with your actual Python code being wrong. Get the
infrastructure genuinely running first, so any error you hit later is a real code bug,
not a missing dependency.

### 0.6 — Initialize the frontend (this one is NOT built from a truly blank folder)

Here's an honest correction to how Part 2 of this guide reads: unlike the backend (where
we hand-created every config file), the frontend's `package.json`/`vite.config.ts`/
`tsconfig.json`/`index.html` are NOT written from scratch — a scaffolding tool generates
working starter versions of all of them instantly, which we then EDIT to match this
project's needs.

```bash
cd ../frontend    # back at the project root, into the frontend folder we made in 0.3

npm create vite@latest . -- --template react-ts
# The trailing "." tells it to scaffold INTO the current (already-created, currently
# empty) frontend folder, rather than creating a new subfolder — otherwise it would try
# to make frontend/frontend/, which isn't what we want since we already made this folder
# ourselves in 0.3.

npm install    # installs the baseline React/TypeScript/Vite deps the template just wrote
              # into package.json

# Now install the ADDITIONAL packages this specific project needs, beyond the bare
# template:
npm install react-router-dom @tanstack/react-query zustand axios recharts

npm install -D @testing-library/react @testing-library/jest-dom @testing-library/user-event \
               jsdom vitest
```

**What `npm create vite` just did for you, concretely:** it generated a working
`package.json`, `tsconfig.json`, `vite.config.ts`, `index.html`, and a placeholder
`src/App.tsx`/`src/main.tsx` that already run successfully together. Part 2's "Step 1"
in this guide is really about DELIBERATELY EDITING what this tool generated (turning on
TypeScript's `strict` mode, adding the dev-server API proxy, adding our own dependencies)
— not writing those files' first draft by hand. Knowing which parts of a real project are
hand-written vs. scaffolded-then-customized is itself useful interview-adjacent knowledge
("do you build every project completely from scratch, or use appropriate tooling?" — the
honest answer for a competent engineer is almost always "use the tooling, then customize
deliberately," not "type every character by hand for the sake of it").

### 0.7 — Sanity-check BOTH sides boot, before writing a single line of real logic

```bash
# Terminal 1 — backend
cd backend && source venv/bin/activate
python -c "import fastapi; print('FastAPI is installed and importable')"

# Terminal 2 — frontend
cd frontend
npm run dev
# Should open a working (if generic/placeholder) React app at http://localhost:5173
```

**Why this checkpoint matters:** if something is broken at the TOOLING level (a bad
install, a version mismatch, a missing system dependency), you want to find that out now,
against a trivial placeholder app, rather than 40 files into the real project when it
becomes much harder to tell "is my code wrong, or is my environment wrong?"

**Only once both of these genuinely work** are you at the starting line this guide's Part
1, Step 1 (`core/config.py`) actually assumes.

---

## PART 1 — BACKEND BUILD ORDER

### Step 1: `backend/app/core/config.py`
**Why this file is FIRST, before literally anything else:** every other backend file
eventually needs *some* piece of configuration (a database URL, a secret key). If you
don't nail this down first, you'll be hardcoding values everywhere and ripping them out
later. This file has **zero dependencies on any other file in the project** — it only
depends on the `pydantic-settings` library — which is exactly why it's buildable first.

**What logic is in it:** A `Settings` class (subclassing `BaseSettings`) declaring every
environment variable the app needs, with types and defaults. One module-level `settings`
instance, created once, imported everywhere else.

**Key technologies/terms:** `pydantic-settings`, environment variables, typed config,
the "singleton" pattern (one shared instance).

**What it unblocks:** Literally every other backend file, since almost all of them
eventually need `from app.core.config import settings`.

---

### Step 2: `backend/app/core/security.py`
**Why now:** Auth is the first REAL feature we'll build (Step 9), and auth needs two
primitives — password hashing and JWT handling — before we can write a single line of
"register a user" logic. This file depends on Step 1 (`settings.SECRET_KEY`,
`settings.ALGORITHM`) but nothing else.

**What logic is in it:** `hash_password()`/`verify_password()` (bcrypt via `passlib`),
`create_access_token()`/`decode_access_token()` (JWT via `python-jose`).

**Key technologies/terms:** bcrypt, JWT (JSON Web Tokens), signing vs encryption, token
expiry.

**What it unblocks:** `dependencies.py` (Step 5) and `services/auth_service.py` (Step 8)
both import functions from here directly.

---

### Step 3: `backend/app/core/database.py`
**Why now:** Before we can write ANY model or service that touches data, we need a way
to actually connect to MongoDB. Depends only on Step 1 (`settings.MONGODB_URL`).

**What logic is in it:** An async Motor client wrapped in a small `MongoDB` class,
`connect_to_mongo()`/`close_mongo_connection()` lifecycle functions, and `get_database()`
as the single accessor everything else uses.

**Key technologies/terms:** Motor (async MongoDB driver), connection pooling, async/await,
why sync DB calls would block an async framework's event loop.

**What it unblocks:** Every service file (Steps 8, 12, 14) needs a database handle, which
ultimately traces back to this file.

---

### Step 4: `backend/app/models/*.py` (`user.py`, `board.py`, `application.py`)
**Why now:** With config, security, and DB connectivity in place, the next question is
"what does our DATA actually look like?" — before writing any business logic, you need
to know the shape of what you're operating on. These three files have NO dependency on
each other (each is self-contained) except that `application.py`'s comment references
`board.py`'s embedding decision for consistency.

**What logic is in it:** Pydantic `BaseModel` classes mirroring exactly what's stored in
each MongoDB collection — `UserInDB`, `BoardInDB` + `Column`, `CardInDB` + `SalaryRange` +
`InterviewRound`.

**Key technologies/terms:** Pydantic models, MongoDB's `_id` field and the `alias`
pattern, embedding vs. referencing (see `docs/03_Database_Schema.md` for the full
decision framework these files implement).

**What it unblocks:** `schemas/*.py` (Step 5) import these models directly (e.g.,
`schemas/application.py` imports `InterviewRound` and `SalaryRange` from
`models/application.py`) — you need to know the internal shape before you can decide
what subset of it to expose at the API boundary.

---

### Step 5: `backend/app/schemas/*.py` (`user.py`, `application.py`, `ai.py`)
**Why now, right after models:** Now that we know the internal data shape, we decide
what a CLIENT is allowed to send us (Create/Update schemas) and see back (Out schemas).
This is a deliberate, distinct step from Step 4 — conflating "how it's stored" with "how
it's exposed" is the exact mistake this two-step split prevents (see `models/user.py`'s
docstring for the security reasoning).

**What logic is in it:** `UserCreate`/`UserOut`/`Token`, `CardCreate`/`CardUpdate` (every
field Optional — the PATCH pattern)/`CardOut`/`BoardOut`, `ResumeUpload`/`ChatMessageIn`/etc.

**Key technologies/terms:** Request/response DTOs, `EmailStr` validation, the
`exclude_unset` PATCH pattern (used later in Step 12's service, but the SCHEMA shape that
enables it is decided here).

**What it unblocks:** Routers (Steps 9, 13, 17) use these schemas as their
`response_model` and request-body type hints.

---

### Step 6: `backend/app/dependencies.py`
**Why now:** We have security utilities (Step 2) and a database connection (Step 3) —
now we combine them into the ONE thing every protected route will need:
"who is making this request, and are they real?" This file is a natural checkpoint
because it's the first place multiple earlier pieces (`security.py` + `database.py` +
`models/user.py`) come together into one reusable unit.

**What logic is in it:** `get_db()` (thin wrapper), `get_current_user()` (the core
FastAPI `Depends()` chain: extract token → decode it → look up the user → return it or
401).

**Key technologies/terms:** FastAPI dependency injection, `OAuth2PasswordBearer`,
`Depends()` chaining (a dependency depending on another dependency).

**What it unblocks:** EVERY protected router from this point forward imports
`get_current_user` and `get_db` from here.

---

### Step 7: `backend/app/websocket_manager.py`
**Why now (before the first real feature, not after):** We're about to build the Board/
Card feature (Steps 11-13), and the architecture decision (see `docs/02_Architecture.md`
§4) is that every mutating card operation broadcasts a WebSocket event AFTER a successful
write. That means the connection manager needs to exist BEFORE we write
`application_service.py`, or that service would have nothing to import.

**What logic is in it:** A `ConnectionManager` class tracking `user_id → [WebSocket, ...]`,
with `connect()`, `disconnect()`, `broadcast_to_user()`.

**Key technologies/terms:** WebSockets, in-memory connection registry, the "REST is
source of truth, WebSocket is notification" principle.

**What it unblocks:** `services/application_service.py` (Step 12) and
`routers/websocket.py` (Step 13).

---

### Step 8: `backend/app/services/auth_service.py` + `routers/auth.py`
**Why THIS is the first complete vertical slice:** Auth has no dependency on Board/Card/
AI features, and every OTHER feature needs an authenticated user to exist — it's the
natural first "real" feature to finish end-to-end. Notice the service (business logic:
"is this email taken? does this password match?") is written first, then the router
(HTTP layer: parse request → call service → translate result to HTTP response) — see
`services/auth_service.py`'s own docstring for exactly why business logic is written
BEFORE and SEPARATELY FROM the HTTP layer that will eventually call it.

**What logic is in it:** `register_user()`/`authenticate_user()` (service, raising plain
Python exceptions like `EmailAlreadyRegisteredError`); `/auth/register`, `/auth/login`,
`/auth/me` (router, catching those exceptions and turning them into `HTTPException`s).

**Key technologies/terms:** Layered architecture (router/service/DB), custom domain
exceptions, `OAuth2PasswordRequestForm`, the "don't leak which part of a login failed"
security principle.

**What it unblocks:** You can now register/login a user via `/docs` (Swagger UI) and get
a real JWT — the FIRST moment the backend is genuinely testable end-to-end.
**This is also the point at which `backend/tests/conftest.py` and `test_auth.py` (Step
16) COULD already be written** — testing infra is often built right alongside (or
immediately after) the first real feature, not deferred to the very end.

---

### Step 9: `backend/app/services/board_service.py` + `routers/boards.py`
**Why now:** With auth working, the next-simplest feature is the board (simpler than
cards — just a "get or create" operation, no CRUD, no WebSocket broadcast needed since
boards rarely change after creation).

**What logic is in it:** `get_or_create_board()` — an idempotent "get it if it exists,
create it if it doesn't" pattern.

**Key technologies/terms:** Idempotent operations, the get-or-create pattern.

**What it unblocks:** `services/application_service.py` (next step) needs a `board_id`
to attach new cards to, which comes from this service.

---

### Step 10: `backend/app/services/application_service.py` + `routers/applications.py`
**Why this is the biggest, most important step in the whole backend:** this is the
CORE product feature. It's the first service that (a) needs the WebSocket manager from
Step 7, (b) needs to get the board from Step 9, and (c) introduces the most important
security pattern in the whole project (query-level ownership filtering).

**What logic is in it:** Full CRUD (`list_cards`, `create_card`, `get_card`,
`update_card`, `delete_card`) plus `get_dashboard_stats()` (a MongoDB aggregation
pipeline). Every single query filters by `owner_id` — see the file's own docstring for
the BOLA (Broken Object-Level Authorization) reasoning. The router layer adds a
`_to_card_out()` mapper function and translates `CardNotFoundError` into a 404.

**Key technologies/terms:** BOLA prevention, MongoDB aggregation pipelines,
`find_one_and_update` with `return_document=True`, the `exclude_unset=True` PATCH
pattern (the single most important line in this entire file — re-read its comment).

**What it unblocks:** The frontend can now do real CRUD. This is also the point where
`backend/tests/test_applications.py` (Step 16) — including the explicit cross-user BOLA
test — becomes writable and meaningful.

---

### Step 11: `backend/app/routers/websocket.py`
**Why now, not earlier:** The `ConnectionManager` (Step 7) and the broadcast CALLS
(inside Step 10's service) already exist — this file is just the missing piece that lets
a CLIENT actually connect and register itself in that manager in the first place.

**What logic is in it:** The `/ws/board` endpoint — decode the token from a query param
(browsers can't set custom headers on a WS handshake), register the connection, loop on
`receive_text()` until disconnect.

**Key technologies/terms:** WebSocket handshake, query-param auth (and its trade-offs),
`WebSocketDisconnect`.

**What it unblocks:** The frontend's `useWebSocket` hook (Frontend Step 9) has something
real to connect to.

---

### Step 12: `backend/app/agents/rag.py`
**Why the AI feature is built LAST among backend features, and why RAG specifically
comes FIRST within the AI feature:** the AI Coach is the most complex, most dependency-
heavy feature — it needs auth (done), and it introduces a genuinely new discipline
(embeddings/retrieval) that the tool-calling loop will depend on. Within "AI," RAG comes
first because `tools.py`'s `search_resume` tool needs `retrieve_relevant_chunks()` to
already exist.

**What logic is in it:** `chunk_text()`, `embed_text()` (via `sentence-transformers`),
`cosine_similarity()` (hand-written, not imported from numpy — on purpose, so the math is
visible), `retrieve_relevant_chunks()`.

**Key technologies/terms:** RAG (Retrieval-Augmented Generation), text chunking with
overlap, embeddings, cosine similarity, in-memory vs. vector-database retrieval trade-offs.

**What it unblocks:** `agents/tools.py`'s `search_resume` function directly calls
`retrieve_relevant_chunks` from here.

---

### Step 13: `backend/app/agents/tools.py`
**Why now:** With RAG available, we can now define the actual TOOLS our agent will be
allowed to call — one of which (`search_resume`) is backed by Step 12's RAG code, and the
other two are self-contained (a static question bank, a mocked company lookup).

**What logic is in it:** `TOOL_SCHEMAS` (the JSON schema sent to Claude describing each
tool) and the three actual Python implementations (`get_interview_questions`,
`search_resume`, `get_company_overview`).

**Key technologies/terms:** Tool-use / function-calling schemas, the importance of
precise tool `description` fields, the "tools don't have to be AI-powered themselves"
distinction.

**What it unblocks:** `agents/agent.py` (next step) imports `TOOL_SCHEMAS` and dispatches
to these functions by name.

---

### Step 14: `backend/app/agents/agent.py`
**Why this is the capstone of the backend build:** every earlier AI-related piece
(RAG, tools) exists specifically to be USED by this file — the actual ReAct
(Reason→Act→Observe) loop that calls Claude, checks if it requested a tool, runs that
tool if so, and repeats until a final answer or `MAX_AGENT_STEPS` is hit.

**What logic is in it:** `run_agent_stream()` — an async generator yielding `token`/
`tool_call`/`done` events as they happen, so the router can forward them live.

**Key technologies/terms:** ReAct pattern, the Anthropic API's streaming + tool-use
interface, async generators, the iteration-cap guardrail against runaway cost/loops.

**What it unblocks:** `services/ai_service.py` and `routers/ai_assistant.py`.

---

### Step 15: `backend/app/services/ai_service.py` + `routers/ai_assistant.py`
**Why last among the "feature" files:** this is the thinnest possible wrapper around
Step 14's real intelligence — `ai_service.py` just persists resumes/conversations (dull
but necessary), and the router's job is almost entirely about the SSE wire format
(`event: ...\ndata: ...\n\n`), converting `agent.py`'s abstract event stream into bytes a
browser's `EventSource`/`fetch` can actually parse.

**What logic is in it:** `upload_resume()` (chunk+embed+upsert), `get_conversation_history
()`, `append_messages()` (atomic upsert with `$push`); the router's `event_generator()`
wraps everything into real SSE frames and persists the full message once streaming ends.

**Key technologies/terms:** Server-Sent Events (SSE) wire format, `StreamingResponse`,
atomic MongoDB upserts (`$push` + `$setOnInsert`), separating "streaming" concerns from
"persistence" concerns.

**What it unblocks:** The frontend's `api/ai.ts` (Frontend Step 3) has a real endpoint to
consume.

---

### Step 16: `backend/app/main.py`
**Why this is written LAST, not first (a common beginner instinct to build this too
early):** `main.py`'s only job is to WIRE UP routers that already exist. Writing it
first would mean importing routers that don't exist yet — it has to come after Steps
8-15 produced actual router objects to import.

**What logic is in it:** The `lifespan` context manager (startup/shutdown), CORS
middleware, the custom `HTTPException` handler (adds `error_code` to every error), and
`app.include_router(...)` calls for every router, all under one `/api/v1` prefix.

**Key technologies/terms:** FastAPI `lifespan`, middleware, centralized exception
handling, API versioning via a URL prefix.

**What it unblocks:** The backend can now actually RUN (`uvicorn app.main:app`).

---

### Step 17: `backend/tests/*.py`
**Why tests are shown last in this walkthrough even though (as noted in Step 8) you'd
often write SOME of them earlier in a real workflow:** for teaching purposes, it's
clearer to see the whole feature set first, then see how it's ALL verified. In your own
practice, write `test_auth.py` right after Step 8, and `test_applications.py` right after
Step 10 — don't actually wait until the end.

**What logic is in it:** `conftest.py`'s fixtures (`test_db` via `mongomock_motor`,
`client` via dependency override, `auth_headers`); unit tests calling services directly;
integration tests going through real HTTP; the explicit BOLA cross-user test.

**Key technologies/terms:** pytest fixtures, dependency overrides for testing, mocking a
database, unit vs. integration test scope.

---

## PART 2 — FRONTEND BUILD ORDER

### Step 1: `package.json`, `vite.config.ts`, `tsconfig.json`
**Why first, same reasoning as the backend's config.py:** nothing else can be written
until the project's dependencies and compiler rules are decided. `tsconfig.json`'s
`strict: true` in particular affects how EVERY subsequent file must be written (no
implicit `any`, mandatory null checks) — deciding this last would mean rewriting
everything.

**Key technologies/terms:** Vite (native ESM dev server vs. bundle-first tooling), the
dev-server proxy config (so `/api` calls don't need a hardcoded backend URL), TypeScript
strict mode.

---

### Step 2: `src/types/index.ts`
**Why immediately after config, before ANY component or API call:** this file is the
frontend's mirror of the backend's `schemas/*.py` — you need to know the exact shape of
`Card`, `User`, etc. before you can write a single API function or component that uses
them. Depends on nothing else in the frontend.

**What logic is in it:** Interfaces mirroring every backend schema, PLUS a discriminated
union (`BoardWebSocketEvent`) for incoming WebSocket messages.

**Key technologies/terms:** TypeScript interfaces, discriminated unions, the "types are
erased at runtime" caveat (why this alone isn't a safety guarantee).

**What it unblocks:** Every single other frontend file, directly or indirectly.

---

### Step 3: `src/api/client.ts`, `auth.ts`, `applications.ts`, `ai.ts`
**Why now, before any UI:** with types defined, the next question is "how do we actually
TALK to the backend?" — this has to exist before any hook or component that needs data,
and it's built in dependency order: `client.ts` (the shared Axios instance + interceptors)
first, since `auth.ts`/`applications.ts`/`ai.ts` all import `apiClient` from it.

**What logic is in it:** `client.ts`'s request interceptor (attach JWT) and response
interceptor (handle 401 globally); typed wrapper functions per feature area; `ai.ts`'s
manual `fetch`-based SSE stream reader (Axios can't do this the way we need).

**Key technologies/terms:** Axios interceptors, the OAuth2 form-encoded login quirk,
`ReadableStream`/`TextDecoder` for manual SSE parsing.

**What it unblocks:** `context/AuthContext.tsx` and every hook in Step 5.

---

### Step 4: `src/context/AuthContext.tsx`
**Why now:** Auth state is needed almost everywhere (protected routes, the navbar), and
it's the textbook Context use case (see the file's own docstring) — build it right after
the API layer it depends on (`api/auth.ts`).

**What logic is in it:** `AuthProvider` (holds `user`/`isLoading`, checks for an existing
token on mount), `useAuth()` (a wrapping hook that throws a clear error if used outside
the provider).

**Key technologies/terms:** React Context API, the "why Context and not Zustand/Redux
for THIS specific state" reasoning, custom-hook-wrapping-useContext pattern.

**What it unblocks:** `ProtectedRoute`, `Navbar`, `LoginForm`, `RegisterForm` — anything
needing to know who's logged in.

---

### Step 5: `src/store/boardStore.ts`
**Why now, right after Context, not mixed in with it:** this is a deliberate contrast —
build the Context-based state (Step 4) and the Zustand-based state (this step) right
next to each other so the DIFFERENCE in what kind of state goes where is fresh in your
mind. Depends on nothing except the `zustand` library itself.

**What logic is in it:** `openCardId`, `draggingCardId`, `isCreateFormOpen` — all
ephemeral, UI-only, ownable-by-no-single-component state.

**Key technologies/terms:** Zustand's `create()` pattern, selector-based subscriptions
(why they avoid unnecessary re-renders).

**What it unblocks:** `Column.tsx`, `KanbanBoard.tsx`, `CardModal.tsx`,
`CreateCardForm.tsx` all read/write this store.

---

### Step 6: `src/hooks/useApplications.ts`, `useWebSocket.ts`, `useDebounce.ts`
**Why hooks come before components that use them:** you always want the LOGIC a
component will lean on to already exist before you write the component itself — writing
them in the other order tends to produce logic awkwardly tangled INSIDE a component,
which then needs a painful later refactor to extract. Build order within this step:
`useApplications.ts` first (it defines `QUERY_KEYS`, which `useWebSocket.ts` imports),
then `useWebSocket.ts`, then the fully independent `useDebounce.ts`.

**What logic is in it:** `useApplications.ts`'s optimistic-update mutation (`onMutate`/
`onError`/`onSettled`) is the single most important piece of frontend logic in the whole
project — re-read its comments carefully. `useWebSocket.ts` connects and invalidates
React Query's cache on any incoming event. `useDebounce.ts` is the generic version of the
advanced interview prep's `useDebouncedSearch`.

**Key technologies/terms:** React Query (`useQuery`/`useMutation`), optimistic updates
and rollback, cache invalidation, the WebSocket-triggers-refetch (not manual patch)
design choice.

**What it unblocks:** Every board-related component from here on.

---

### Step 7: `src/components/common/*.tsx` (`Button`, `Input`, `Spinner`)
**Why generic components before feature components:** these have ZERO dependencies on
anything else in the app (no hooks, no context, no types beyond standard HTML attribute
types) — build the pieces with no dependencies first, so every later component can
immediately reach for them instead of writing raw `<button>`/`<input>` tags and having to
retrofit consistency later.

**Key technologies/terms:** Extending native HTML element prop types
(`ButtonHTMLAttributes`), accessibility basics (`aria-invalid`, label `htmlFor`).

**What it unblocks:** Every form and every clickable element in the rest of the app.

---

### Step 8: `src/components/layout/*.tsx` (`ErrorBoundary`, `ProtectedRoute`, `Navbar`)
**Why layout/shell components before feature components:** these wrap or gate OTHER
components — `ErrorBoundary` will wrap the whole app, `ProtectedRoute` will wrap the
dashboard route — so they need to exist before `App.tsx` (the last file) can reference
them, but they themselves only depend on Steps 4 (AuthContext) and 7 (Button).

**Key technologies/terms:** Class components (the ONLY one in this app —
`ErrorBoundary`, because hooks have no equivalent), `getDerivedStateFromError` vs.
`componentDidCatch`, `<Navigate>` for redirects.

---

### Step 9: `src/components/auth/*.tsx` (`LoginForm`, `RegisterForm`)
**Why now:** with `useAuth()` (Step 4), `Button`/`Input` (Step 7) all available, the auth
forms are now just composition + local form state — the FIRST genuinely "feature-level"
components in the build.

**Key technologies/terms:** Controlled inputs, `useState` per field, async error handling
in an event handler (NOT caught by ErrorBoundary — see that file's own docstring for why).

---

### Step 10: `src/components/board/*.tsx` — build order WITHIN this folder matters a lot:

1. **`ApplicationCard.tsx` first** — the smallest, leaf-most piece (renders one card,
   handles being dragged). Depends on `store/boardStore.ts` and `types/index.ts` only.
2. **`Column.tsx` second** — renders a LIST of `ApplicationCard`s and handles being a
   DROP target. Depends on Step 10.1 plus `hooks/useApplications.ts`'s `useUpdateCard`.
3. **`KanbanBoard.tsx` third** — renders a LIST of `Column`s, fetches the board+cards,
   groups cards by column (the `useMemo`'d grouping). Depends on 10.1, 10.2, plus
   `useWebSocket`.
4. **`CreateCardForm.tsx` and `CardModal.tsx` fourth** — the two modals `KanbanBoard`
   conditionally renders based on Zustand state. `CardModal` is more complex (it hosts
   the AI panel as a tab) so it comes after the simpler create form.
5. **`StatsPanel.tsx` last** — entirely independent of the drag-and-drop mechanics above;
   only needs `useDashboardStats()` and the `recharts` library.

**Why this specific inside-out order (leaf components before container components) is
the general rule, not just a coincidence of THIS project:** you cannot meaningfully write
`KanbanBoard.tsx` (which renders `<Column>`, which renders `<ApplicationCard>`) until the
inner pieces exist to be rendered — React's component composition model means the
dependency direction is always "children exist before the parent that renders them" in
terms of writing order (even though at RUNTIME, the parent renders first).

---

### Step 11: `src/components/ai/*.tsx` (`ChatMessage`, `AIAssistantPanel`)
**Why the AI UI comes after the board UI, mirroring the backend's build order:**
`AIAssistantPanel` is embedded as a TAB inside `CardModal` (Step 10.4) — it couldn't be
finished (or even meaningfully tested in place) until `CardModal` existed to host it.
Build `ChatMessage` (the leaf) before `AIAssistantPanel` (the container), same leaf-first
principle as Step 10.

**What logic is in it:** `AIAssistantPanel`'s local `streamingMessage` state, updated via
the `onToken`/`onToolCall`/`onDone` callbacks from `api/ai.ts`'s `streamChatMessage` —
this is where Step 3's manual SSE-parsing code finally gets CONSUMED by a real UI.

**Key technologies/terms:** Incremental/streaming UI updates, the functional
`setState(prev => ...)` form (essential here — re-read the comment on why), auto-scroll
via `ref` + `scrollIntoView`.

---

### Step 12: `src/pages/*.tsx` (`LoginPage`, `RegisterPage`, `DashboardPage`)
**Why pages are nearly the LAST thing written:** a "page" in this architecture is pure
composition — it has almost no logic of its own, just arranging already-built components
(`LoginForm` inside `LoginPage`, `Navbar`+`StatsPanel`+`KanbanBoard` inside
`DashboardPage`). You literally cannot write a meaningful page until the components it
composes already exist.

---

### Step 13: `src/App.tsx` and `src/main.tsx`
**Why these are the ABSOLUTE last frontend files:** `App.tsx` is the root of the whole
provider tree and routing map — it imports EVERY page, which is why it has to come after
Step 12. `main.tsx` is even later still — its only job is mounting `App` into the real
DOM, so it needs `App.tsx` to exist first.

**Key technologies/terms:** Provider tree ordering (why `QueryClientProvider` wraps
`AuthProvider`, not the reverse), `createRoot`, `StrictMode`'s dev-only double-invocation
behavior.

**What it unblocks:** `npm run dev` now actually shows a working app in the browser —
this is the moment the frontend becomes genuinely runnable end-to-end, the frontend
equivalent of backend Step 16.

---

### Step 14: `src/index.css`, `nginx.conf`, Dockerfiles, `docker-compose.yml`, CI workflow
**Why styling and deployment configs are last:** you cannot meaningfully style or deploy
an app that doesn't have any components rendering real content yet. This is also true at
the PROJECT level, not just the frontend — `docker-compose.yml` (which wires backend +
frontend + MongoDB together) is one of the very last files in the ENTIRE project, because
it has nothing to wire together until both halves independently work.


---

## PART 3 — The Full Dependency Map (Who Imports Whom)

Use this as a quick-reference "what breaks if I change this file" map. An arrow means
"the file on the left imports/depends on the file on the right."

### Backend
```
main.py
  ├─→ core/database.py (connect/close lifecycle)
  └─→ routers/*.py (registers each router)
        auth.py          → services/auth_service.py   → core/security.py, models/user.py
        boards.py        → services/board_service.py  → models/board.py
        applications.py  → services/application_service.py → models/application.py,
                                                               websocket_manager.py,
                                                               services/board_service.py
        ai_assistant.py  → services/ai_service.py      → agents/rag.py
                          → agents/agent.py            → agents/tools.py → agents/rag.py
        websocket.py     → websocket_manager.py, core/security.py

  Every router above also →  dependencies.py → core/security.py, core/database.py,
                                                 models/user.py
  Every router/service above →  schemas/*.py  → models/*.py (schemas import model sub-types)
  Everything →  core/config.py (the one file with NO dependencies, imported everywhere)
```

### Frontend
```
main.tsx
  └─→ App.tsx
        ├─→ context/AuthContext.tsx → api/auth.ts → api/client.ts
        ├─→ components/layout/{ErrorBoundary,ProtectedRoute,Navbar}.tsx
        └─→ pages/*.tsx
              LoginPage/RegisterPage → components/auth/*.tsx → context/AuthContext.tsx
              DashboardPage → components/layout/Navbar.tsx
                            → components/board/StatsPanel.tsx → hooks/useApplications.ts
                            → components/board/KanbanBoard.tsx
                                  → components/board/Column.tsx
                                        → components/board/ApplicationCard.tsx → store/boardStore.ts
                                  → components/board/{CardModal,CreateCardForm}.tsx
                                        → components/ai/AIAssistantPanel.tsx → api/ai.ts
                                        → hooks/{useApplications,useDebounce}.ts
                                  → hooks/useWebSocket.ts → hooks/useApplications.ts (shares QUERY_KEYS)

  hooks/useApplications.ts → api/applications.ts → api/client.ts
  Everything that touches API data →  types/index.ts (the one file with NO dependencies,
                                                        imported nearly everywhere)
```

**How to use this map in practice:** before changing any file, scan for it on the LEFT
side of an arrow above (or search your codebase for its import statements) — that tells
you every file you might break. Before DELETING a file, scan for it on the RIGHT side —
that tells you everything that would stop working.

---

## PART 4 — How to Repeat This Process Yourself (the generalizable checklist)

The single habit this whole guide is trying to teach: **before writing a file, ask "what
does this file need to already exist, and what will exist BECAUSE of this file?"** Below
is the same reasoning turned into a repeatable checklist for YOUR next feature or project.

### When adding a new backend feature (e.g., "email reminders")
1. **Does it need new config?** (an email API key, a "remind X days before" setting) →
   add it to `core/config.py` first.
2. **Does it need a new data shape?** → add a model to `models/` (what's actually
   stored) before deciding the API contract.
3. **What can a client send/see?** → add Create/Update/Out schemas to `schemas/`.
4. **What's the actual business rule?** ("send a reminder if `applied_date` was 7+ days
   ago and no `interview_rounds` exist yet") → write it as a plain-Python service
   function in `services/`, with NO FastAPI imports, so it's unit-testable in isolation.
5. **How does an HTTP client trigger/see this?** → write the thin router in `routers/`,
   translating service exceptions into HTTP responses.
6. **Wire it up** → one `app.include_router(...)` line in `main.py`.
7. **Test it** → a unit test for the service (Step 4's logic, no HTTP), an integration
   test for the router (Step 5, through real HTTP), and — if it touches another user's
   data in any way — an explicit cross-user authorization test, matching
   `test_user_cannot_access_another_users_card` in spirit.

### When adding a new frontend feature (e.g., "CSV export button")
1. **Does the backend need to change first?** (yes, usually) → build that vertical slice
   on the backend using the checklist above BEFORE touching the frontend at all.
2. **Update `types/index.ts`** to match whatever new API shape you just built.
3. **Add the API call** to the relevant file in `api/` (or a new file, if it's a
   genuinely new domain area).
4. **Decide what KIND of state is involved** — is it server data (→ a new React Query
   hook in `hooks/`), or purely local UI state (→ Zustand, or just `useState` if nothing
   else needs to know about it)?
5. **Build the smallest/leaf component first**, then the component that composes it —
   same inside-out order as Part 2, Step 10 above.
6. **Wire it into a page**, and if it needs a new ROUTE, add it to `App.tsx` last.

### The one question to ask before ANY file, on either side of the stack
> "If I opened this file in isolation, with no other context, would I be able to tell
> from its own imports what it depends on — and are all of THOSE dependencies things
> that already exist and already work?"

If the answer is no, you've found the file you actually need to write first.
