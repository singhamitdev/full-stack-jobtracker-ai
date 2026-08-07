# Frontend Concepts Glossary — Every Important Term, File by File

**Companion to `docs/09_Backend_Concepts_Glossary.md`** — same format: What it is → Why
it's used here → a tiny standalone example. Organized in the build order from
`docs/08_Development_Workflow_Guide.md`, Part 2.

---

## `src/types/index.ts`

### `interface` declarations
**What it is:** Defines the SHAPE an object must have — which keys exist, and what type
each one's value must be. TypeScript checks this at compile time; it doesn't exist at
runtime at all (see the file's own docstring for why that's a real limitation, not just
a technicality).
**Why here:** `interface Card { id: string; company: string; ... }` means anywhere a
`Card` is used, your editor knows EXACTLY what fields are available, and using a
misspelled field name (`card.compnay`) is a compile error, not a silent `undefined` bug
discovered by a user.
**Example:**
```ts
interface Point { x: number; y: number }
const p: Point = { x: 1, y: 2 };   // ✅ matches the shape
const bad: Point = { x: 1 };        // ❌ compile error — missing `y`
```

### Union types (`'pending' | 'passed' | 'failed'`)
**What it is:** Says a value must be ONE of these exact literal strings — nothing else
is allowed.
**Why here:** `outcome: 'pending' | 'passed' | 'failed'` prevents a typo like
`outcome: 'passd'` from ever compiling, unlike a plain `outcome: string`, which would
accept absolutely any string with no protection at all.

### Discriminated unions (`BoardWebSocketEvent`)
**What it is:** A union of several object shapes that all share one common field (here,
`event`) with a different literal value per shape — TypeScript automatically narrows
which shape you're dealing with once you check that field.
**Why here:** After checking `if (data.event === 'card_deleted')`, TypeScript KNOWS
`data.card_id` exists and `data.card` does NOT — making it a compile error to access
`.card` on that branch, catching a whole category of "wrong field for this event type"
bugs before the code ever runs. See the advanced interview prep's Q31 for the full
pattern this implements.
**Example:**
```ts
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'square'; side: number };

function area(s: Shape) {
  if (s.kind === 'circle') return Math.PI * s.radius ** 2;  // s.radius is valid HERE
  return s.side ** 2;                                        // s.side is valid HERE
}
```

---

## `src/api/client.ts`, `auth.ts`, `applications.ts`, `ai.ts`

### `axios.create({...})`
**What it is:** Creates a configured, reusable HTTP client instance (as opposed to
calling the plain `axios.get(...)` directly every time), with shared settings like a
base URL.
**Why here:** Every API call in the app goes through this ONE instance, so a change to
how requests are authenticated (see interceptors below) applies everywhere automatically.

### Interceptors (`apiClient.interceptors.request.use(...)`)
**What it is:** A function that runs automatically on EVERY request (or response) passing
through this Axios instance, able to modify it before it's sent (or handle it after it
comes back).
**Why here:** The request interceptor attaches the JWT to every outgoing call; the
response interceptor catches every 401 in one place and redirects to `/login` — without
either piece of logic needing to be repeated at each individual API call site.
**Example:**
```ts
const client = axios.create();
client.interceptors.request.use((config) => {
  config.headers.Authorization = 'Bearer some-token';   // added to EVERY request automatically
  return config;
});
```

### `async`/`await` in TypeScript (same concept as Python, different syntax)
**What it is:** Identical concept to the backend's `async def`/`await` — a function
marked `async` can `await` a Promise (JS's version of "a value that will exist later"),
pausing execution until it resolves, without blocking the rest of the browser.
**Why here:** `async function loginUser(...) { const { data } = await apiClient.post(...); return data; }`
waits for the network request to finish before using its result, without freezing the UI
during that wait.

### Generics in a function call (`apiClient.get<User>('/auth/me')`)
**What it is:** Tells TypeScript "the data this returns will be shaped like `User`" —
same generics concept as the advanced interview prep's Q28/Q29, applied to an HTTP call.
**Why here:** Without it, `data` would be typed `any`, and every field access afterward
would have zero type safety at all.

### `URLSearchParams` and manual form-encoding
**What it is:** A browser API for building a `key=value&key2=value2`-style string —
used here instead of JSON specifically because the backend's login endpoint expects
form-encoded data (see `api/auth.ts`'s comment on the OAuth2 convention).

### `ReadableStream`, `getReader()`, `TextDecoder` (manual streaming — `api/ai.ts`)
**What it is:** Low-level browser APIs for reading a response body incrementally, chunk
by chunk, AS it arrives over the network — rather than waiting for the whole response to
finish downloading first.
**Why here:** This is what makes the AI chat feel like it's "typing" in real time — see
the advanced interview prep's Q50 for the full streaming-UI concept this implements.

---

## `src/context/AuthContext.tsx`

### `createContext<T>(defaultValue)`
**What it is:** Creates a Context object — a channel for passing a value down through
the component tree WITHOUT manually passing it as a prop through every intermediate
component ("prop drilling").
**Why here:** `createContext<AuthContextValue | undefined>(undefined)` — starting the
default as `undefined` (not `null` or a fake user) lets `useAuth()` DETECT if it's ever
called outside an `<AuthProvider>` at all, and throw a clear error instead of silently
behaving wrongly.

### `useContext(AuthContext)`
**What it is:** The Hook that actually READS the current value provided by the nearest
matching `<AuthContext.Provider>` above it in the tree.

### `useState<User | null>(null)`
**What it is:** Same `useState` Hook from the basic-level interview prep's Q5, here
explicitly typed — `User | null` means "either a real logged-in user, or genuinely no
one," matching reality more precisely than pretending a user always exists.

### `useEffect(() => {...}, [])`
**What it is:** Same Hook from the basic-level prep's Q6 — an empty dependency array
`[]` means "run this once, when the component first mounts."
**Why here:** Checks for an existing token in `localStorage` exactly once, when the app
first loads — not on every re-render, which would be wasteful and pointless since a
token in storage doesn't change on its own.

### Custom hooks wrapping built-in ones (`function useAuth() { ... }`)
**What it is:** A function starting with `use` that itself calls other Hooks inside it —
same concept as the basic-level prep's Q9, here wrapping `useContext` specifically so
callers write the shorter `useAuth()` and get a clear error message if misused.

### `throw new Error(...)`
**What it is:** JavaScript/TypeScript's exception-raising mechanism — same underlying
idea as Python's `raise`, different keyword.
**Why here:** Fails LOUDLY and immediately if `useAuth()` is called outside an
`AuthProvider`, rather than silently returning broken/undefined behavior that's much
harder to trace back to its actual cause.

---

## `src/store/boardStore.ts`

### Zustand's `create<T>()(...)`
**What it is:** Defines a global state store — a set of values plus functions that
update them — accessible from any component via a Hook-like call, WITHOUT wrapping the
app in a Context provider at all.
**Why here (vs. Context, used in AuthContext.tsx just above):** This state changes
FREQUENTLY (opening/closing modals, drag state) — Zustand's per-value subscriptions mean
a component reading only `openCardId` doesn't re-render when `draggingCardId` changes,
unlike Context, where ANY value change re-renders every consumer. See the advanced
interview prep's Q5 for the full decision framework.
**Example:**
```ts
import { create } from 'zustand';
const useCounter = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
}));
// In a component: const count = useCounter((state) => state.count);
```

### Selector functions (`useBoardStore((state) => state.openCardId)`)
**What it is:** Passing a function to the store Hook that picks out JUST the piece of
state you need — this is what enables the "only re-render when THIS specific value
changes" behavior mentioned above.

---

## `src/hooks/useApplications.ts`, `useWebSocket.ts`, `useDebounce.ts`

### `useQuery({ queryKey, queryFn })`
**What it is:** React Query's Hook for fetching and caching SERVER data — `queryKey`
uniquely identifies this piece of cached data; `queryFn` is the async function that
actually fetches it.
**Why here:** Handles caching, loading/error states, and background refetching
automatically — see `docs/05_Frontend_Architecture.md` for why server data specifically
belongs here, not in Zustand or Context.

### `useMutation({ mutationFn, onMutate, onError, onSettled })`
**What it is:** React Query's Hook for CHANGING server data (create/update/delete),
with optional callbacks at each stage of the change.
**Why here — the optimistic update pattern (the most important pattern in this whole
file):** `onMutate` runs IMMEDIATELY, before the network request even starts, updating
the local cache so the UI feels instant; `onError` rolls that back if the request fails;
`onSettled` refetches to reconcile with the server's authoritative state either way.
Re-read the extensive comments in `useUpdateCard` — this is worth understanding deeply.

### `queryClient.setQueryData(...)` / `getQueryData(...)` / `invalidateQueries(...)`
**What it is:** Direct, manual manipulation of React Query's cache — reading its current
value, overwriting it, or marking it stale so the next read triggers a refetch.
**Why here:** `setQueryData` implements the optimistic update itself; `invalidateQueries`
is how `useWebSocket.ts` tells React Query "the data might be stale, refetch it" when a
server-pushed event arrives from another tab.

### `useRef<T>(initialValue)`
**What it is:** Holds a mutable value that PERSISTS across re-renders WITHOUT causing a
re-render itself when changed — unlike `useState`, which always triggers a re-render.
**Why here (`useWebSocket.ts`):** The WebSocket connection object itself doesn't need to
trigger a re-render when it changes — it's an implementation detail, not something
rendered in the UI.
**Example:**
```ts
const renderCount = useRef(0);
renderCount.current += 1;   // does NOT cause a re-render, unlike setState would
```

### The generic debounce hook (`function useDebounce<T>(value: T, delayMs: number): T`)
**What it is:** A REUSABLE version of the pattern from the advanced interview prep's Q4
(`useDebouncedSearch`) — generalized here to work with ANY value type `T`, not just
search strings, via a generic type parameter.

---

## `src/components/common/Button.tsx`, `Input.tsx`, `Spinner.tsx`

### Extending native HTML prop types (`ButtonHTMLAttributes<HTMLButtonElement>`)
**What it is:** TypeScript types, provided by React's own type definitions, describing
every prop a native `<button>` element accepts (`onClick`, `disabled`, `type`, etc.).
**Why here:** `interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement>`
means our custom `<Button>` automatically accepts every normal button prop PLUS our own
`variant` prop, without manually re-declaring `onClick`/`disabled`/etc. one by one.

### The rest (`...rest`) spread pattern in props
**What it is:** `{ variant, children, ...rest }` destructures out the props we handle
specially, and collects EVERYTHING else into `rest`, which then gets spread onto the
actual `<button>` element (`<button {...rest}>`).
**Why here:** Lets any caller pass standard button props (like `disabled` or `onClick`)
straight through to the real DOM element, without `Button.tsx` needing to know about
every possible prop in advance.

---

## `src/components/layout/ErrorBoundary.tsx`

### Class components (the only one in this codebase)
**What it is:** The pre-Hooks way of writing a React component, using `class X extends
Component`. React error boundaries can ONLY be written this way — there is no Hook
equivalent, a genuine current gap in the Hooks API (see the advanced interview prep's Q16).

### `static getDerivedStateFromError()` and `componentDidCatch()`
**What it is:** Two lifecycle methods specific to class components — the first computes
new STATE in response to an error (no side effects allowed here); the second is where
side effects (like logging to an error-tracking service) belong. This separation mirrors
the render-vs-commit separation from the advanced interview prep's Q2.

---

## `src/components/auth/LoginForm.tsx`, `RegisterForm.tsx`

### Controlled inputs (`value={email} onChange={(e) => setEmail(e.target.value)}`)
**What it is:** The input's displayed value is driven entirely by React state — every
keystroke updates state, which then re-renders the input with the new value. Same
concept as the basic-level interview prep's Q7.

### `FormEvent` and `e.preventDefault()`
**What it is:** `FormEvent` types the browser's native form-submission event;
`preventDefault()` stops the browser's DEFAULT behavior for that event — here, stopping
a full-page reload that a plain HTML form submission would otherwise trigger.
**Why here:** We want to handle the submission with our OWN JavaScript (calling the API,
managing loading state) instead of letting the browser do a traditional page navigation.

### `try` / `catch` / `finally` around an async call
**What it is:** Same concept as Python's `try`/`except`/`finally`, JS/TS syntax.
**Why here:** Catches a failed login attempt and shows a friendly error message instead
of an unhandled promise rejection; `finally` ensures the "loading" state is turned off
regardless of success or failure.

---

## `src/components/board/ApplicationCard.tsx`, `Column.tsx`, `KanbanBoard.tsx`

### Native HTML5 Drag and Drop events (`onDragStart`, `onDragOver`, `onDrop`, `e.dataTransfer`)
**What it is:** Browser-native events for drag-and-drop interactions — `dataTransfer` is
the mechanism for passing data FROM the drag source TO whatever eventually receives the
drop.
**Why here:** `e.dataTransfer.setData('cardId', card.id)` (in `ApplicationCard`) and
`e.dataTransfer.getData('cardId')` (in `Column`) are how the dropped-ON column learns
WHICH card was just dropped onto it — this is a real, low-level browser API, not a
React-specific concept.

### `e.preventDefault()` inside `onDragOver`
**What it is:** Browsers block dropping onto an element BY DEFAULT unless its
`dragover` handler explicitly calls `preventDefault()`.
**Why here:** Without this exact line, `Column`'s `onDrop` handler would never fire at
all — a genuinely easy-to-miss, non-obvious browser requirement.

### `key={card.id}` on a mapped list — revisited with real stakes
**What it is:** Same concept as the basic-level interview prep's Q8, but here it's not
theoretical — `Column.tsx`'s comment explains the EXACT bug that would occur (a card's
local edit-mode state sticking to the wrong visual position) if index were used instead.

### `useMemo(() => {...}, [cards])`
**What it is:** Same Hook from the advanced interview prep's Q6 — recomputes its result
ONLY when something in the dependency array (`[cards]`) actually changes, caching the
result otherwise.
**Why here (`KanbanBoard.tsx`):** Grouping cards by column is an O(n) loop — memoizing it
avoids re-running that loop on every render caused by unrelated state changes elsewhere
in the app, a genuinely measurable saving once card count grows.

---

## `src/components/ai/AIAssistantPanel.tsx`

### The functional `setState` form (`setStreamingMessage((prev) => ({...prev, ...}))`)
**What it is:** Passing a FUNCTION to a state setter (rather than a plain new value)
receives the LATEST state as its argument, guaranteeing correctness even when several
updates happen in rapid succession (as tokens stream in).
**Why here:** Directly implements the closures concept from the basic-level interview
prep's Q13 — without the functional form, rapid-fire token updates could each capture a
STALE snapshot of `streamingMessage` from when the callback was originally created,
silently dropping some tokens.
**Example:**
```ts
// RISKY — `count` here is whatever it was when this callback was CREATED, not necessarily current
setCount(count + 1);

// SAFE — `prev` is always the ACTUAL latest value, regardless of timing
setCount((prev) => prev + 1);
```

### `useRef` for DOM access + `scrollIntoView`
**What it is:** `useRef` here holds a reference to an actual DOM element (an empty
`<div>` at the bottom of the messages list); `.scrollIntoView()` is a native browser DOM
method, not a React concept — this is a case of "reaching outside React" to directly
manipulate the DOM, which `useEffect` + `useRef` together is the correct, sanctioned way
to do (see the basic-level interview prep's Q6).

---

## `src/App.tsx`, `main.tsx`

### `QueryClient` and `QueryClientProvider`
**What it is:** `QueryClient` is React Query's central cache/config object; the
`Provider` makes it available to every `useQuery`/`useMutation` call anywhere in the tree.
**Why here:** Created ONCE, as a module-level constant OUTSIDE the `App` component —
creating it INSIDE the component (even with `useState`) risks accidentally recreating it
on certain re-renders, which would wipe the entire cache.

### `<Routes>` / `<Route path="..." element={...} />` (React Router)
**What it is:** Declarative routing — maps a URL path to the component that should
render for it.

### `createRoot(...).render(...)`
**What it is:** React 18's entry point for mounting the app into a real DOM node — the
modern replacement for the older `ReactDOM.render(...)`.
**Why here:** `createRoot` specifically (not the legacy API) is required for React 18's
concurrent features (referenced throughout the advanced interview prep's Q2) to be
available at all.

### `<StrictMode>`
**What it is:** A wrapper component that intentionally double-invokes certain functions
in DEVELOPMENT ONLY, to surface accidental impurities (a component doing something it
shouldn't during render).
**Why here:** Zero effect on the production build's actual behavior — purely a
development-time safety net, as covered in the advanced interview prep's Q2.
