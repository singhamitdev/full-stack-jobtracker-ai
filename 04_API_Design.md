# API Design — JobTrackr AI

Base URL (local dev): `http://localhost:8000/api/v1`
FastAPI auto-generates interactive docs at `/docs` (Swagger) and `/redoc` — this is a genuine FastAPI feature worth mentioning in interviews: your API contract documentation is never out of sync with your code because it's generated FROM the code (Pydantic models + type hints).

## Auth

| Method | Path | Auth? | Description |
|---|---|---|---|
| POST | `/auth/register` | No | Create a new user account |
| POST | `/auth/login` | No | Exchange email+password for a JWT access token |
| GET | `/auth/me` | Yes | Get the currently authenticated user's profile |

**POST `/auth/register`**
```json
// Request
{ "email": "jane@example.com", "password": "correct-horse-battery-staple", "full_name": "Jane Doe" }
// Response 201
{ "id": "665f...", "email": "jane@example.com", "full_name": "Jane Doe" }
```

**POST `/auth/login`**
```json
// Request (form-encoded, OAuth2 password flow convention)
{ "username": "jane@example.com", "password": "correct-horse-battery-staple" }
// Response 200
{ "access_token": "eyJhbGciOi...", "token_type": "bearer" }
```

## Boards

| Method | Path | Auth? | Description |
|---|---|---|---|
| GET | `/boards/me` | Yes | Get (or auto-create) the current user's board, with columns |

## Cards (Applications)

| Method | Path | Auth? | Description |
|---|---|---|---|
| GET | `/cards` | Yes | List all cards for the current user's board |
| POST | `/cards` | Yes | Create a new card |
| GET | `/cards/{card_id}` | Yes | Get one card's full detail |
| PATCH | `/cards/{card_id}` | Yes | Partially update a card (edit fields, or move column/order) |
| DELETE | `/cards/{card_id}` | Yes | Delete a card |

**POST `/cards`**
```json
// Request
{ "company": "Anthropic", "role": "Senior Frontend Engineer", "column_id": "wishlist", "job_url": "https://..." }
// Response 201
{ "id": "665f...", "company": "Anthropic", "role": "Senior Frontend Engineer", "column_id": "wishlist",
  "order": 0, "notes": "", "interview_rounds": [], "created_at": "2026-08-06T10:00:00Z" }
```

**PATCH `/cards/{card_id}`** (this single endpoint handles both "edit details" AND "drag to a new column" — the request body just contains whichever fields changed; see `schemas/application.py` for why PATCH + all-optional-fields is the right REST pattern here)
```json
// Request (moving a card via drag-and-drop)
{ "column_id": "interviewing", "order": 1 }
// Response 200 — the updated card
```

## AI Coach

| Method | Path | Auth? | Description |
|---|---|---|---|
| POST | `/ai/resume` | Yes | Upload/replace the user's resume text (triggers chunking + embedding for RAG) |
| POST | `/ai/chat/{card_id}` | Yes | Send a message to the AI coach, scoped to one application; **streams back the response** (Server-Sent Events) |
| GET | `/ai/chat/{card_id}/history` | Yes | Get the past conversation for this application |

**POST `/ai/chat/{card_id}`**
```json
// Request
{ "message": "What should I expect in the system design round, and how does my resume line up?" }
// Response: `text/event-stream` — a sequence of Server-Sent Events, e.g.
// event: token
// data: {"text": "Based on "}
// event: token
// data: {"text": "typical system "}
// ...
// event: done
// data: {"tool_calls": [{"tool": "search_resume", "input": {"query": "system design experience"}}]}
```

## WebSocket

| Path | Description |
|---|---|
| `WS /ws/board` | Client connects after login (JWT passed as a query param, since browsers can't set custom headers on the initial WS handshake). Server pushes `{"event": "card_updated", "card": {...}}` style messages whenever ANY device/tab mutates a card, so all open tabs stay in sync. |

## Standard Error Shape (every error response, across all endpoints)

```json
{ "detail": "Human-readable message", "error_code": "CARD_NOT_FOUND" }
```
FastAPI's `HTTPException` produces the `detail` field automatically; `error_code` is our own addition (a custom exception handler — see `backend/app/main.py`) so the frontend can branch on a stable machine-readable code instead of parsing an English sentence, which is the correct pattern for any real production API.

## Pagination (documented convention, applied to `GET /cards`)

We use **cursor-based pagination** (not offset) even though a user's card count is small in v1 — this is a deliberate consistency choice so the pattern is correct from day one and doesn't need retrofitting, directly matching the advanced interview prep's Q37 reasoning (offset pagination "drifts" under concurrent writes; cursor pagination doesn't).
```
GET /cards?limit=20&cursor=<opaque-cursor>
```
