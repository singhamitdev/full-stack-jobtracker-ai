# Backend Concepts Glossary — Every Important Term, File by File

**How to use this doc:** Open it side-by-side with the actual code file it's describing
(same pattern as `docs/08_Development_Workflow_Guide.md`). Each entry follows: **What it
is → Why it's used here → A tiny standalone example** you can run in isolation to see the
concept work, separate from the project's real code.

Terms are explained THOROUGHLY the first time they appear (in build order — see
`docs/08_Development_Workflow_Guide.md` for that order), and given a brief reminder +
pointer back on later appearances, rather than being re-explained at full length every
single time — otherwise this document would be mostly repetition.

---

## `app/core/config.py`

### `class Settings(BaseSettings):` — classes and inheritance
**What it is:** A class is a blueprint bundling data (attributes) and behavior (methods)
together. `(BaseSettings)` means `Settings` **inherits** from Pydantic's `BaseSettings` —
it automatically gets all of `BaseSettings`'s existing behavior (reading env vars,
validating types) for free, and only needs to declare WHICH fields exist.
**Why here:** Without inheritance, you'd hand-write the "read this from an environment
variable, validate its type, fall back to a default" logic yourself, for every setting.
**Example:**
```python
class Animal:
    def speak(self): return "some generic sound"

class Dog(Animal):        # Dog INHERITS from Animal
    pass                    # gets .speak() for free, no need to rewrite it

Dog().speak()  # "some generic sound" — inherited, not redefined
```

### Type hints (`str`, `int`, `list[str]`, `bool`)
**What it is:** Annotations telling Python (and your editor) what TYPE a variable/
parameter/field is expected to hold. Python doesn't enforce these at runtime by itself —
but Pydantic (used throughout this backend) DOES enforce them, raising a validation error
if the actual value doesn't match.
**Why here:** `ACCESS_TOKEN_EXPIRE_MINUTES: int = 30` means "this must be an integer, and
defaults to 30 if not set in the environment" — if someone sets it to `"abc"` in `.env`,
Pydantic raises a clear startup error instead of a confusing crash later.
**Example:**
```python
def greet(name: str) -> str:      # says: takes a str, returns a str
    return f"Hello, {name}"
```

### Default values on class attributes (`SECRET_KEY: str = "dev-only-secret..."`)
**What it is:** If no value is provided (here: no environment variable set), Python uses
the value after `=` instead.
**Why here:** Lets the app run locally with sane defaults without requiring every
developer to configure every single variable before they can even start the server.

### `model_config = SettingsConfigDict(env_file=".env", ...)`
**What it is:** A special class attribute telling Pydantic-Settings HOW to find its
values — here, "also check a file named `.env`," in addition to real environment
variables.
**Why here:** Lets you keep local secrets in a git-ignored `.env` file during
development, while production (which has no `.env` file at all) still works via real
environment variables injected by the hosting platform.

### The module-level singleton instance (`settings = Settings()`)
**What it is:** Creating exactly ONE instance of the class, at the bottom of the file, so
every other file that does `from app.core.config import settings` gets the SAME object
— read from the environment exactly once.
**Why here:** Avoids re-reading/re-validating environment variables every time
configuration is needed — a "compute once, reuse everywhere" pattern.

---

## `app/core/security.py`

### `CryptContext(schemes=["bcrypt"], ...)`
**What it is:** An object from the `passlib` library configured to use the bcrypt
hashing algorithm specifically (rather than a faster, less secure one).
**Why here:** bcrypt is DELIBERATELY slow — this makes brute-force password guessing
attacks computationally expensive even if a password database leaks. This is a security
requirement, not a performance oversight.

### `try` / `except` (exception handling)
**What it is:** `try` wraps code that MIGHT fail; `except SomeError:` catches that
specific failure and lets you respond gracefully instead of crashing.
**Why here (in `decode_access_token`):** `jwt.decode(...)` raises `JWTError` for an
invalid, tampered, or expired token — we catch it and return `None` instead of letting
the exception crash the whole request.
**Example:**
```python
try:
    result = 10 / 0
except ZeroDivisionError:
    result = None   # handled gracefully instead of crashing the program
```

### `str | None` (the modern union-type syntax)
**What it is:** Means "this value is EITHER a `str`, OR `None`." The `|` here is Python
3.10+'s shorthand for what used to require `Optional[str]` (an older, more verbose
syntax from the `typing` module).
**Why here:** `decode_access_token`'s return type is `str | None` because a bad token
genuinely has no valid user ID to return — `None` represents that "nothing valid" case
honestly in the type signature itself, rather than the function pretending it always
succeeds.

### `datetime.now(timezone.utc) + timedelta(minutes=...)`
**What it is:** `timedelta` represents a SPAN of time (e.g., "30 minutes"), which can be
added to a specific point in time (`datetime.now(...)`) to compute a future timestamp.
**Why here:** Computes exactly when a JWT should expire — always in UTC, never a local
timezone, so token expiry behaves identically regardless of which timezone the server
happens to be running in.

---

## `app/core/database.py`

### `class MongoDB:` with class-level attributes (no `__init__`)
**What it is:** A class used purely as a namespaced container for two values (`client`,
`db`) that start as `None` and get filled in later — notice this class is never
"instantiated" with arguments; it's used more like an organized global variable holder.
**Why here:** Groups `client`/`db` together under one name (`mongodb.client`,
`mongodb.db`) rather than having two disconnected global variables floating in the
module — and starting both as `None` makes it explicit that they're not valid until
`connect_to_mongo()` actually runs.

### `async def` and `await` (asynchronous functions)
**What it is:** `async def` declares a function that can be PAUSED while waiting on slow
I/O (a network call, a database query) so the program can do OTHER work during that
wait, instead of sitting frozen. `await` is used INSIDE an async function to actually
pause at a specific slow operation.
**Why here:** `await mongodb.db["users"].create_index(...)` doesn't block the whole
server while waiting for MongoDB's response — other incoming requests can be handled
during that wait. This is the single most important concept in the entire backend; if
you only deeply understand one term from this glossary, make it this one.
**Example:**
```python
import asyncio

async def slow_task():
    await asyncio.sleep(2)   # pauses THIS task for 2 seconds, but doesn't freeze the whole program
    return "done"

async def main():
    result = await slow_task()
    print(result)

asyncio.run(main())
```

### `assert` statements
**What it is:** `assert condition, "message"` — if `condition` is `False`, immediately
raises an `AssertionError` with that message; if `True`, does nothing at all.
**Why here (in `get_database()`):** `assert mongodb.db is not None, "..."` is a defensive
check catching a specific programmer mistake (forgetting to call `connect_to_mongo()`
before using the database) with an immediately understandable error, rather than a
confusing `NoneType has no attribute...` error much later.

---

## `app/models/user.py`, `board.py`, `application.py`

### `class UserInDB(BaseModel):` — Pydantic models
**What it is:** `BaseModel` (from Pydantic) is a class that, once inherited from, turns
plain type-hinted class attributes into a fully validating data structure — assigning
the wrong type to a field raises a clear error immediately.
**Why here:** Guarantees that any `UserInDB` object anywhere in the codebase genuinely
has the shape it claims to, rather than trusting a plain dictionary to "probably" have
the right keys.

### `Field(alias="_id")`
**What it is:** Lets a Pydantic field be POPULATED from a differently-named key in the
source data — here, MongoDB's `_id` key maps to Python's more conventional `id`
attribute name.
**Why here:** MongoDB always names its primary key `_id`; Python style strongly prefers
`id` without the underscore — `alias` lets you satisfy both conventions at once, without
manually renaming the key every time you read from or write to the database.

### `Field(default_factory=lambda: datetime.now(timezone.utc))`
**What it is:** `default_factory` provides a FUNCTION that computes a default value each
time a new instance is created — as opposed to a plain default like `= 30`, which
computes the value only ONCE, when the class is first defined. A `lambda` is a small,
unnamed, inline function.
**Why here:** If we wrote `created_at: datetime = datetime.now(timezone.utc)` (no
factory), EVERY user would get the exact same timestamp — whatever moment the CLASS was
first loaded, not when each individual user was actually created. `default_factory`
computes it fresh, per instance, at the moment each one is made.
**Example:**
```python
from datetime import datetime
import time

class Bad:
    created = datetime.now()   # computed ONCE, when the class is defined

a = Bad(); time.sleep(2); b = Bad()
a.created == b.created   # True — same instant, which is WRONG for "created_at"
```

### `list[Column]` and `list[InterviewRound]` (generic container types)
**What it is:** Says "this is a list, and specifically a list OF `Column` objects" (not
just any list of anything).
**Why here:** Gives you autocomplete and type-checking on items you pull out of the
list — `board.columns[0].title` is known to be valid because the type system knows every
item is a `Column`, which has a `.title` field.

### Module-level constants (`DEFAULT_COLUMNS = [...]`)
**What it is:** A value defined once, at the top level of a module (not inside any
function or class), acting as a single shared source of truth.
**Why here:** Both the board-creation logic AND any future test referencing "the default
columns" import this SAME list, rather than each writing out the five column names
separately and risking the two copies silently drifting apart over time.

---

## `app/schemas/*.py`

### `EmailStr` (a Pydantic-provided specialized type)
**What it is:** A string type that Pydantic validates IS a well-formed email address
(has an `@`, a domain, etc.) — rejecting malformed input automatically, before your own
code ever has to think about it.
**Why here:** `UserCreate.email: EmailStr` means a registration request with
`email: "not-an-email"` is rejected by FastAPI automatically with a clear 422 error,
before `register_user()`'s own logic ever runs.

### `Field(min_length=8, description="...")`
**What it is:** Adds VALIDATION CONSTRAINTS directly onto a field, beyond just its type.
**Why here:** `password: str = Field(min_length=8)` rejects a too-short password at the
API boundary automatically — no manual `if len(password) < 8:` check needed in your own
service code.

### Why THIS folder is separate from `models/` (a design pattern, not a syntax term)
**What it is:** `schemas/` defines what a CLIENT sends/receives; `models/` defines what's
ACTUALLY STORED internally. `UserOut` (schema) deliberately has no `hashed_password`
field at all — not "hidden," genuinely absent from the type — while `UserInDB` (model)
does.
**Why here:** Makes it a compile-time/validation-time impossibility to accidentally leak
an internal-only field in an API response — see `models/user.py`'s own docstring for the
full security reasoning.

---

## `app/dependencies.py`

### `Depends(...)` (FastAPI's dependency injection)
**What it is:** Tells FastAPI "before running this route, first call THIS other function,
and pass its return value in as this parameter." Dependencies can depend on OTHER
dependencies — FastAPI resolves the whole chain automatically.
**Why here:** `current_user: UserInDB = Depends(get_current_user)` means EVERY protected
route gets a verified, real `UserInDB` object injected automatically — the auth-checking
logic is written exactly once, here, instead of copy-pasted into every route.
**Example (a simplified, non-FastAPI illustration of the same IDEA):**
```python
def get_greeting():
    return "Hello"

def get_message(greeting=get_greeting()):   # "depends on" get_greeting's result
    return f"{greeting}, world!"

get_message()  # "Hello, world!" — FastAPI's Depends() does this MUCH more powerfully,
                # re-running the dependency fresh on every request, with error handling
```

### `OAuth2PasswordBearer(tokenUrl="auth/login")`
**What it is:** A FastAPI helper that (a) extracts the `Bearer <token>` string from the
`Authorization` header of an incoming request automatically, and (b) tells the
interactive `/docs` page which endpoint to use for its built-in "Authorize" login button.

### Raising `HTTPException`
**What it is:** A special exception class that, when raised inside a route or
dependency, FastAPI automatically converts into a real HTTP error response with the
given status code and detail message.
**Why here:** `raise HTTPException(status_code=401, detail="...")` stops the request
immediately and sends the client a proper 401 response — the client never reaches the
actual route body at all.

---

## `app/services/auth_service.py`, `board_service.py`, `application_service.py`

### Custom exception classes (`class EmailAlreadyRegisteredError(Exception):`)
**What it is:** Defining your OWN exception type by inheriting from Python's built-in
`Exception` class — `pass` means "no extra behavior needed beyond what `Exception`
already provides."
**Why here:** Lets the SERVICE layer signal a specific business-rule failure ("this
email is taken") using plain Python, with ZERO knowledge of HTTP — the ROUTER layer
catches this specific exception type and decides how to translate it into an HTTP
response. This is what keeps business logic and HTTP concerns cleanly separated.
**Example:**
```python
class InsufficientFundsError(Exception):
    pass

def withdraw(balance, amount):
    if amount > balance:
        raise InsufficientFundsError(f"Cannot withdraw {amount}, only {balance} available")
    return balance - amount
```

### `uuid.uuid4()`
**What it is:** Generates a random, effectively-unique 128-bit identifier (looks like
`a1b2c3d4-...`).
**Why here:** Used as MongoDB's `_id` for new users/boards/cards instead of letting
MongoDB auto-generate its own `ObjectId` — a plain string UUID is simpler to embed in
JWTs, URLs, and JSON without needing special conversion helpers everywhere.

### `async for ... in ...:` (async iteration over a database cursor)
**What it is:** Like a normal `for` loop, but for iterating over an ASYNC source (here,
a MongoDB query cursor) — each iteration may pause to wait for the next batch of results
from the database.
**Why here (`list_cards`):**
```python
cursor = db["cards"].find({"owner_id": owner_id}).sort("order", 1)
return [CardInDB(**doc) async for doc in cursor]
```
This is an **async list comprehension** — builds a list by looping over the cursor,
converting each raw MongoDB document (`doc`, a plain dict) into a validated `CardInDB`
object via `**doc` (see next entry).

### `**doc` (dictionary unpacking into keyword arguments)
**What it is:** `**` before a dictionary "unpacks" its key-value pairs as individual
keyword arguments to a function/constructor call.
**Why here:** `CardInDB(**doc)` where `doc = {"company": "Acme", "role": "Engineer"}` is
exactly equivalent to writing `CardInDB(company="Acme", role="Engineer")` by hand — but
works automatically regardless of how many keys the dict has.
**Example:**
```python
def greet(name, age):
    return f"{name} is {age}"

info = {"name": "Alice", "age": 30}
greet(**info)   # "Alice is 30" — same as greet(name="Alice", age=30)
```

### `data.model_dump(exclude_unset=True, mode="json")`
**What it is:** Converts a Pydantic model back into a plain dictionary. `exclude_unset
=True` includes ONLY the fields the caller explicitly set (omitting anything that just
defaulted to `None` because it wasn't mentioned at all). `mode="json"` ensures nested
values like `datetime` are converted into JSON-safe strings.
**Why here (the single most important line in `update_card`):** This is what makes a
PATCH request like `{"column_id": "offer"}` update ONLY that field — without
`exclude_unset=True`, every other field (which defaults to `None` on the `CardUpdate`
schema) would be included and would OVERWRITE existing data with `None`, silently
wiping it. Re-read the full comment in `application_service.py` — this is a genuinely
common real-world bug this one flag prevents.

### MongoDB aggregation pipelines (`get_dashboard_stats`)
**What it is:** A list of stages (`$match`, `$group`, etc.) that MongoDB itself executes,
server-side, to transform/summarize data — as opposed to fetching raw documents and
looping over them in Python.
**Why here:** `{"$group": {"_id": "$column_id", "count": {"$sum": 1}}}` counts cards per
column INSIDE the database, which scales far better than pulling every card over the
network and counting in a Python loop once a user has thousands of cards.

---

## `app/websocket_manager.py`

### A class holding genuinely mutable state (`self.active_connections: dict[...]`)
**What it is:** Unlike `config.py`'s `Settings` (which is essentially read-only after
creation) or the service files (mostly plain functions), `ConnectionManager` is a class
specifically because it needs to REMEMBER and MUTATE a growing/shrinking collection
(which users are connected right now) across many separate method calls over time — the
textbook reason to reach for a class instead of plain functions (see the earlier
explanation of `class Settings` for the general decision framework).

### `dict[str, list[WebSocket]]` (nested generic types)
**What it is:** A dictionary where each KEY is a `str` (a user ID) and each VALUE is a
`list` of `WebSocket` objects (that user's open connections/tabs).
**Why here:** One user can have the board open in multiple browser tabs simultaneously —
the value needs to be a LIST, not a single connection, to support that.

---

## `app/routers/*.py`

### `APIRouter()` and route decorators (`@router.get("/me")`)
**What it is:** `APIRouter` groups related endpoints together (e.g., everything
`/auth/*`) before being included into the main `app` in `main.py`. `@router.get(...)` is
a DECORATOR — a function that wraps another function to add behavior. Here, it
registers the function right below it as the handler for GET requests to that exact path.
**Why here:** Keeps auth endpoints, card endpoints, and AI endpoints in separate,
independently readable files, all merged together in one place (`main.py`) at the end.
**Example (a simplified illustration of what a decorator does, conceptually):**
```python
def shout(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs).upper()
    return wrapper

@shout
def greet():
    return "hello"

greet()   # "HELLO" — @shout wrapped greet() with extra behavior, transparently
```

### `response_model=UserOut`
**What it is:** Tells FastAPI to validate AND FILTER the route's return value against
this schema before sending it — even if the underlying object had extra fields, only
`UserOut`'s declared fields make it into the actual HTTP response.
**Why here:** A genuine safety net, not just documentation — if a service function
accidentally returned a `hashed_password` field, `response_model=UserOut` would strip it
before it ever reached the client, since `UserOut` has no such field defined at all.

### `status.HTTP_201_CREATED`, etc.
**What it is:** Named constants for HTTP status codes, from FastAPI's `status` module —
more readable than remembering `201` means "created" from memory.

---

## `app/agents/rag.py`

### `math.sqrt(...)` and hand-written vector math
**What it is:** Plain Python arithmetic implementing cosine similarity by hand (dot
product divided by the product of magnitudes) instead of importing `numpy` for one
function.
**Why here:** Makes the actual MATH of "how similar are two pieces of text" fully
visible and explainable — a common "can you explain this" interview moment, and a good
example of NOT reaching for a heavy dependency when a few lines of plain Python do the
job clearly.

### List comprehensions with a condition, and generator-style sorting
**What it is:** `[(cosine_similarity(...), chunk["text"]) for chunk in chunks]` builds a
list of `(score, text)` tuples in one line; `.sort(key=lambda pair: pair[0], reverse=True)`
sorts that list by the FIRST element of each tuple (the score), highest first.
**Why here:** Concise, readable way to score every candidate chunk and rank them by
relevance — see the basic-level interview prep's Q32 for the general list-comprehension
concept this builds on.

---

## `app/agents/tools.py`

### Plain dictionaries as data (`TOOL_SCHEMAS`, `_QUESTION_BANK`)
**What it is:** No classes here at all — just nested dictionaries and lists, because
there's no behavior to bundle, only static reference data.
**Why here:** Reinforces the "when NOT to use a class" side of the earlier decision
framework — this data doesn't change over time and doesn't need methods, so a class
would add ceremony with zero benefit.

### `str.format(role=role)`
**What it is:** A string method that substitutes `{role}` placeholders inside a string
with an actual value.
**Why here:** `"How would you design {role} systems...".format(role=role)` inserts the
specific job role into a template question without manual string concatenation.

---

## `app/agents/agent.py`

### `AsyncGenerator` and `yield` (async generator functions)
**What it is:** A function containing `yield` doesn't run all at once and return a
single value — it PAUSES at each `yield`, hands a value back to whoever is consuming it,
and RESUMES from that exact point when asked for the next value. `AsyncGenerator` is the
type-hint for a generator that ALSO supports `await` inside it.
**Why here:** `run_agent_stream()` yields `{"type": "token", ...}` events one at a time,
AS Claude generates them — the caller (the router) can forward each one to the browser
immediately, rather than waiting for the entire response to finish first.
**Example (a plain, non-async illustration of the core `yield` idea):**
```python
def countdown(n):
    while n > 0:
        yield n     # pauses here, hands back n, resumes on the NEXT call
        n -= 1

for number in countdown(3):
    print(number)   # prints 3, 2, 1 — one at a time, not all at once
```

### `async with ... as ...:` (async context managers)
**What it is:** Like a normal `with` statement (which guarantees cleanup code runs even
if an error occurs), but for a resource that needs `await` to properly set up or tear
down — here, an open streaming connection to Claude's API.
**Why here:** `async with _client.messages.stream(...) as stream:` guarantees the
streaming connection is properly closed once the block finishes, even if something goes
wrong partway through.

### List/dict comprehensions filtering by attribute (`[block for block in ... if block.type == "tool_use"]`)
**What it is:** Same list comprehension concept as `rag.py`, applied here to filter
Claude's response content down to just the blocks representing a tool-call request.

---

## `app/main.py`

### `@asynccontextmanager` and the `lifespan` pattern
**What it is:** A decorator (from Python's `contextlib` module) that turns a generator
function into a proper async context manager. Everything before `yield` runs at
STARTUP; everything after `yield` runs at SHUTDOWN.
**Why here:** `connect_to_mongo()` runs once when the app starts; `close_mongo_connection
()` runs once when it stops — FastAPI calls this automatically at the right moments.
**Example (the shape, simplified):**
```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app):
    print("starting up")
    yield                     # the app runs HERE, in between
    print("shutting down")
```

### Middleware (`app.add_middleware(CORSMiddleware, ...)`)
**What it is:** Code that runs on EVERY request/response passing through the app, before
it reaches your actual route handlers (or after, on the way back out).
**Why here:** CORS middleware checks the `Origin` header of every incoming request and
decides whether to allow it, based on the `CORS_ORIGINS` allow-list from `config.py` —
applied uniformly, without needing to repeat that check in every single route.

### `@app.exception_handler(HTTPException)`
**What it is:** Registers a function to intercept every `HTTPException` raised ANYWHERE
in the app and customize how it's turned into a response.
**Why here:** Adds a consistent `error_code` field to every error response, project-wide,
from one single place, rather than every router remembering to add it individually.

---

## `backend/tests/*.py`

### Pytest fixtures (`@pytest_asyncio.fixture`, function parameters matching fixture names)
**What it is:** A fixture is reusable setup code that pytest automatically runs and
"injects" into any test function that declares a parameter with the SAME NAME as the
fixture.
**Why here:** `async def test_x(client, auth_headers):` automatically gets a fresh test
HTTP client AND a logged-in user's auth headers, without that test having to set either
one up manually — the setup logic is written once, in `conftest.py`, and reused
everywhere.

### `app.dependency_overrides[get_db] = lambda: test_db`
**What it is:** FastAPI-specific: replaces a real dependency with a fake one, for the
duration of a test, using the SAME `Depends()` mechanism from `dependencies.py`.
**Why here:** Lets tests use an in-memory fake database (`mongomock_motor`) instead of a
real MongoDB server, with ZERO changes to the actual route/service code being tested —
the substitution happens entirely at the dependency-injection layer.

### `pytest.raises(SomeException)`
**What it is:** A context manager asserting that the code inside its `with` block DOES
raise the given exception type — the test FAILS if the exception is NOT raised.
**Example:**
```python
import pytest

def divide(a, b):
    if b == 0:
        raise ValueError("cannot divide by zero")
    return a / b

def test_divide_by_zero_raises():
    with pytest.raises(ValueError):
        divide(10, 0)   # test passes BECAUSE this raises ValueError as expected
```
