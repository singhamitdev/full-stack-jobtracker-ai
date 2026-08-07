# Database Schema — MongoDB

MongoDB has no fixed schema, but a *production* app still needs one — enforced in application code (via Pydantic models) rather than by the database itself. This is itself an interview talking point: "How do you ensure data integrity in a schemaless database?" → answer: **application-level validation + a documented schema + indexes that assume a shape.**

## Collection: `users`

```json
{
  "_id": ObjectId,
  "email": "jane@example.com",       // unique index
  "hashed_password": "$2b$12$...",    // NEVER store plaintext — see backend/app/core/security.py
  "full_name": "Jane Doe",
  "created_at": ISODate,
  "updated_at": ISODate
}
```
**Index:** `{ email: 1 }` unique — enforces no duplicate accounts and makes login lookups O(log n) instead of a full collection scan.

---

## Collection: `boards`

Each user gets exactly one board in v1 (kept simple on purpose — the schema still supports multiple boards per user for future growth).

```json
{
  "_id": ObjectId,
  "owner_id": ObjectId,               // references users._id
  "name": "My Job Search",
  "columns": [                         // EMBEDDED, not referenced — see rationale below
    { "id": "wishlist",    "title": "Wishlist",    "order": 0 },
    { "id": "applied",     "title": "Applied",     "order": 1 },
    { "id": "interviewing","title": "Interviewing","order": 2 },
    { "id": "offer",       "title": "Offer",       "order": 3 },
    { "id": "rejected",    "title": "Rejected",    "order": 4 }
  ],
  "created_at": ISODate
}
```
**Index:** `{ owner_id: 1 }` — every board query filters by owner; this index makes that filter fast and, combined with application-level filtering, is our BOLA (Broken Object-Level Authorization) defense (see `02_Architecture.md` §5).

**Why columns are embedded, not a separate collection:** columns are always fetched WITH their board, never independently, rarely change, and there are only ever a handful — the textbook case for embedding (see the "embedding vs referencing" decision framework below).

---

## Collection: `cards` (the job applications)

```json
{
  "_id": ObjectId,
  "board_id": ObjectId,               // references boards._id
  "owner_id": ObjectId,               // DENORMALIZED copy of the board's owner — see note below
  "column_id": "applied",              // matches a columns[].id in the parent board
  "order": 2,                          // position within the column, for drag-and-drop ordering
  "company": "Anthropic",
  "role": "Senior Frontend Engineer",
  "job_url": "https://...",
  "salary_range": { "min": 150000, "max": 190000, "currency": "USD" },
  "notes": "Referred by a friend. Recruiter call went well.",
  "applied_date": ISODate,
  "interview_rounds": [                 // EMBEDDED array — see rationale below
    { "round": "Recruiter screen", "date": ISODate, "outcome": "passed", "notes": "..." },
    { "round": "System design", "date": ISODate, "outcome": "pending", "notes": "..." }
  ],
  "created_at": ISODate,
  "updated_at": ISODate
}
```
**Indexes:**
- `{ owner_id: 1, column_id: 1 }` — the exact shape of our most common query ("give me this user's cards in column X").
- `{ board_id: 1, order: 1 }` — supports efficient ordered retrieval for rendering a column top-to-bottom.

**Why `owner_id` is denormalized onto the card (not just looked up via `board_id → boards.owner_id`):**
This is a deliberate, explainable trade-off. Without it, checking "does this user own this card" would require a lookup into `boards` first (or a `$lookup`/join). By duplicating `owner_id` directly onto the card, every authorization check and every "get my cards" query is a single-collection query with no join — at the cost of needing to keep `owner_id` in sync if a card ever changed boards (which, in this product, it structurally can't — cards don't move between boards, only between columns). **This is the classic NoSQL trade-off: duplicate data deliberately to optimize your most frequent read pattern, accepting the write-side cost of keeping the duplicate in sync.** Being able to articulate this trade-off is a strong signal in a backend interview.

**Why `interview_rounds` is embedded:** always read/written together with the card, never queried independently ("find all interview rounds across all cards" is not a real product query) — another clean embedding case.

---

## Collection: `resumes`

```json
{
  "_id": ObjectId,
  "owner_id": ObjectId,
  "raw_text": "Jane Doe\nSenior Frontend Engineer\n...",   // extracted plain text
  "chunks": [                                                // pre-split for RAG — see rag.py
    { "chunk_id": 0, "text": "...", "embedding": [0.012, -0.034, ...] },
    { "chunk_id": 1, "text": "...", "embedding": [0.045, 0.007, ...] }
  ],
  "uploaded_at": ISODate
}
```
**Why embeddings are stored alongside the resume document, not in a separate vector DB:** at this scale (one resume per user, a few dozen chunks), an external vector database (Pinecone/Chroma/Weaviate) is unnecessary infrastructure — we do in-memory cosine similarity over a small array (see `rag.py`). **Knowing when NOT to reach for a vector DB is exactly the kind of judgment call the advanced interview prep's Q54 decision framework is about.** The code comments in `rag.py` document precisely at what scale (~10,000+ chunks, or multi-user shared search) you WOULD need a real vector DB, and why.

---

## Collection: `ai_conversations`

```json
{
  "_id": ObjectId,
  "owner_id": ObjectId,
  "card_id": ObjectId,                  // which application this conversation is scoped to
  "messages": [
    { "role": "user", "content": "What should I expect in the system design round?", "timestamp": ISODate },
    { "role": "assistant", "content": "Based on typical rounds at similar companies...", "timestamp": ISODate,
      "tool_calls": [ { "tool": "get_interview_questions", "input": {...}, "output": {...} } ] }
  ],
  "created_at": ISODate,
  "updated_at": ISODate
}
```
**Index:** `{ owner_id: 1, card_id: 1 }` — conversations are always fetched scoped to a specific application.

**Why `tool_calls` is logged inside each message:** this is our agent observability — directly reusing the advanced interview prep's Q53 principle ("log every step so you can debug which stage introduced an error"), applied at data-model level.

---

## Embedding vs Referencing — the Decision Framework Used Above

| Question | Embed | Reference |
|---|---|---|
| Is the child data always read together with the parent? | ✅ | ❌ |
| Does the child data need to be queried independently? | ❌ | ✅ |
| Is the child collection unbounded in size (could grow to thousands per parent)? | ❌ | ✅ |
| Does the child change far more often than the parent? | ❌ (embedding still fine unless VERY frequent) | ✅ if extremely frequent |

Applying this: `columns` (few, always read with board) → embed. `interview_rounds` (few per card, always read with card) → embed. `cards` themselves (queried independently — "show me all cards in this column" — and unbounded per board) → separate collection, referenced by `board_id`.
