# Frontend Architecture — JobTrackr AI

## 1. Folder Structure (feature-and-kind hybrid — small enough for kind-based, documented for growth)

```
frontend/src/
  api/            # All HTTP calls live here — components never call fetch/axios directly
  types/          # Shared TypeScript interfaces/types (the "contract" with the backend)
  context/        # React Context providers (auth — low-frequency, broadly-needed state)
  store/          # Zustand stores (board UI state — higher-frequency, locally-scoped state)
  hooks/          # Custom hooks (useAuth, useApplications, useDebounce, useWebSocket)
  components/
    layout/       # Navbar, ProtectedRoute, ErrorBoundary — app shell concerns
    board/        # KanbanBoard, Column, ApplicationCard, CardModal — the core feature
    auth/         # LoginForm, RegisterForm
    ai/           # AIAssistantPanel, ChatMessage
    common/       # Button, Input, Spinner — generic, reusable, no business logic
  pages/          # Route-level components that compose the above (LoginPage, DashboardPage)
  App.tsx         # Router setup
  main.tsx        # React root + provider tree
```

At this project's size, a single `components/` folder organized by domain (board/auth/ai) rather than a full feature-module structure (`features/board/{components,hooks,api}`) is the right call — see the advanced interview prep's Q1 for exactly when you'd graduate to full feature-based modules (short answer: once 3+ people are contributing and boundaries need enforcing).

## 2. State Management Strategy — Three Kinds of State, Three Tools

This directly implements the advanced interview prep's Q13 pattern:

| Kind of state | Example | Tool | Why |
|---|---|---|---|
| **Server state** | Cards, board, AI conversation history | **React Query** | Handles caching, background refetch, loading/error states, and — critically — cache invalidation after a mutation, without us hand-rolling any of it |
| **Global UI state** | Current logged-in user, auth token | **React Context** | Changes rarely (login/logout), needed broadly across the whole app — the textbook Context use case |
| **Local/ephemeral UI state** | Which card modal is open, current drag target, AI panel open/closed | **Zustand** | Changes frequently and is UI-only (not worth persisting to a server) — using Context here would cause unnecessary re-renders across the whole app on every drag event (see advanced prep Q3/Q5) |

**What we deliberately do NOT do:** put server data (cards) into Zustand or Context. This is the most common beginner mistake this project is designed to correct — server data has its own lifecycle (staleness, refetching, optimistic updates) that a generic state store doesn't handle, which is exactly why React Query exists.

## 3. Component Tree (simplified)

```
App
 └─ AuthProvider (Context)
     └─ Router
         ├─ LoginPage → LoginForm
         ├─ RegisterPage → RegisterForm
         └─ ProtectedRoute
             └─ DashboardPage
                 ├─ Navbar
                 ├─ StatsPanel
                 └─ KanbanBoard
                     └─ Column (×5)
                         └─ ApplicationCard (×N)
                             └─ CardModal (opened on click)
                                 └─ AIAssistantPanel (tab within the modal)
                                     └─ ChatMessage (×N)
```

## 4. Data Flow for a Drag-and-Drop Move (end-to-end, worth being able to narrate in an interview)

1. User drags `ApplicationCard` from Column A to Column B.
2. `KanbanBoard`'s `onDragEnd` handler (native HTML5 Drag and Drop API — see component comments for why we didn't add a library dependency for this) computes the new `column_id`/`order`.
3. **Optimistic update:** React Query's `useMutation` immediately updates the local cache to show the card in its new position — the UI feels instant, before the network request even resolves.
4. The `PATCH /cards/{id}` request fires in the background.
5. On success: React Query reconciles the cache with the server's authoritative response (usually a no-op visually, since we predicted correctly).
6. On failure: React Query's `onError` rolls back the optimistic update, and a toast explains what happened — the UI never lies about state for long.
7. Separately, the backend's WebSocket broadcast tells any OTHER open tab to refetch — so a second monitor with the board open also updates, without polling.

This flow is a direct, hands-on implementation of "optimistic updates" and "cache invalidation," two concepts that come up constantly in frontend system-design interviews.

## 5. Routing Map

| Path | Component | Protected? |
|---|---|---|
| `/login` | LoginPage | No |
| `/register` | RegisterPage | No |
| `/` | DashboardPage (board + stats) | Yes |

`ProtectedRoute` (see `components/layout/ProtectedRoute.tsx`) wraps protected routes and redirects to `/login` if there's no valid auth context — implementing exactly the pattern from the advanced interview prep's Q57 (client-side route guarding as a UX layer, with the REAL enforcement happening server-side on every API call).

## 6. Design Tokens (lightweight, intentional — not a generic AI-template palette)

Since this is a productivity/focus tool (not a marketing site), the design leans calm and legible rather than flashy:

- **Palette:** deep slate (`#1E293B`) for text, a muted teal (`#0D9488`) as the single accent (used sparingly — for the primary action button and active column indicator only), warm off-white (`#FAFAF7`) background, soft amber (`#F59E0B`) reserved specifically for "Interviewing" status (the stage that needs attention) so status has a color language, not just a label.
- **Type:** `Inter` for UI text (legible at small sizes, which matters for a dense kanban board), `JetBrains Mono` for anything code/data-like (salary figures, dates) — a small, deliberate detail that reinforces this is a tool for engineers.
- **Layout:** columns are fixed-width with horizontal scroll on overflow (rather than shrinking), so card content never gets cramped — a real Kanban-tool decision, not a default.
