# Project Rules (Kiwi-Go)

This document is the **single source of truth** for how we structure, name, and build this codebase so it stays **modular, scalable, and easy to navigate** as features grow.

It complements:
- `Kiwi-Go/_docs/project-overview.md`
- `Kiwi-Go/_docs/user-flow.md`
- `Kiwi-Go/_docs/tech-stack.md`
- `Kiwi-Go/_docs/ui-rules.md`
- `Kiwi-Go/_docs/theme-rules.md`

---

## Goals (non-negotiable)

- **Navigability**: predictable places for files; no “misc” dumping grounds.
- **Modularity**: features are isolated; cross-feature coupling is explicit.
- **Separation of concerns**: UI ≠ state ≠ side effects; routes ≠ business logic ≠ data access.
- **Type safety end-to-end**: TypeScript strict mode; Prisma-generated types; shared types where appropriate.
- **Production readiness**: structured logging, error handling, env validation, security defaults.

---

## Repository layout (target structure)

This project is intended to scale cleanly as a **monorepo** (recommended), while still allowing each app to run independently.

```
Kiwi-Go/
  _docs/
  apps/
    web/                      # React + TS + Tailwind + shadcn
    api/                      # Node + Express + TS + Prisma + Socket.IO
  packages/
    shared/                   # shared types + schema utilities (no Node/DOM-only code)
    config/                   # shared eslint/tsconfig/prettier configs (optional)
  infra/
    docker/                   # Dockerfiles + compose fragments
    aws/                      # ECS task defs, IaC, notes (optional)
  scripts/                    # one-off dev scripts (idempotent; documented)
  .github/
    workflows/
  README.md
```

### Rules for what goes where

- **apps/**: runnable applications only.
- **packages/**: reusable, versionable code with clean boundaries (shared types, utilities).
- **infra/**: deployment and runtime configuration (Docker, AWS), not app code.
- **_docs/**: specs, UX rules, flows, decisions.

---

## Naming conventions

### Files and folders

- **Folders**: `kebab-case` (e.g. `conversation-list/`, `auth-callback/`).
- **TypeScript files**: `kebab-case.ts` / `kebab-case.tsx`.
- **React components**: `PascalCase.tsx` for component files when the file is the component (e.g. `MessageBubble.tsx`).
- **Hooks**: `use-something.ts` (e.g. `use-socket.ts`, `use-presence.ts`).
- **Tests**: `*.test.ts` / `*.test.tsx`.

### Symbols

- **Components**: `PascalCase`
- **Functions/variables**: `camelCase` using auxiliary verbs for booleans (`isLoading`, `hasMoreMessages`)
- **Types**: `PascalCase` (`User`, `Conversation`, `MessageDTO`)
- **Constants**: `SCREAMING_SNAKE_CASE` only for true constants (event names, env keys)

### Exports

- Prefer **named exports** for everything.
- Avoid “god barrels”. `index.ts` is allowed **only** to export a small, coherent public surface for a folder.

---

## Frontend architecture rules (`apps/web`)

### Feature-first organization

Organize by feature, not by file type.

```
apps/web/src/
  app/                        # routing + app composition
  features/
    auth/
    conversations/
    messages/
    presence/
    groups/
  components/                 # shared UI primitives & composition components
  hooks/                      # cross-feature hooks
  lib/                        # small focused utilities (fetch client, socket client, cn(), etc.)
  state/                      # Redux store setup; RTK Query APIs; UI slices
  styles/
  types/
```

### State management (from `tech-stack.md`)

- **RTK Query**: server state (conversation list, chat history pagination, user lists).
- **Slices**: local UI state (modals, active tabs, panel visibility).
- **Do not** store massive message history in Redux; prefer pagination + virtualization.

### UI rules (from `ui-rules.md` + `theme-rules.md`)

- **Mobile-first** layout with bottom navigation on mobile.
- **Sidebar becomes a drawer** on mobile; persistent column on desktop.
- **Presence** uses Emerald consistently; do not invent new “online” colors.
- **Optimistic UI** for sending messages with explicit `sending`/`failed` states.
- **Virtualize** long message lists (only render what’s on screen).
- **Theme**: minimalist; prefer borders over heavy shadows; use consistent radii (`rounded-lg`, `rounded-full`).

### Accessibility and semantics

- Use semantic HTML landmarks (`header`, `nav`, `main`, `aside`).
- Use `<button>` for actions, `<a>` for navigation, and always include accessible names.
- Keyboard navigation must work (focus styles must remain visible).

---

## Backend architecture rules (`apps/api`)

### Three-layer architecture (from `tech-stack.md`)

**Routes → Controllers → Services → Data access (repositories)**

- **Routes**: define endpoints and middleware only.
- **Controllers**: request/response translation; validation; call services.
- **Services**: business rules; orchestration; transactions; emit domain events.
- **Repositories**: Prisma queries only; no HTTP/websocket concerns.

Target layout:

```
apps/api/src/
  server.ts                   # server bootstrap only
  app.ts                      # express app wiring
  config/                     # env + app configuration
  modules/                    # feature modules (vertical slices)
    auth/
    users/
    conversations/
    messages/
    presence/
  db/                         # prisma client, repository base helpers
  realtime/                   # socket.io namespaces/rooms, redis adapter wiring
  middleware/                 # auth, error handling, request logging
  lib/                        # logger, ids, time, small shared helpers
  types/
```

### Prisma conventions (Context7-backed)

- **Schema path**: `prisma/schema.prisma`
- **Migrations**: `prisma/migrations/`
- **Env**: `DATABASE_URL` is required at startup; do not boot if missing.
- **Prisma client**: create a **single shared instance** and reuse it to avoid excess connections; do not instantiate Prisma in request handlers.

### Error handling & logging (from `tech-stack.md`)

- Always return **sanitized** client errors (no stack traces).
- Use structured JSON logging (e.g. `pino` or `winston`) with request correlation IDs.
- Centralize errors in a single error middleware.

### Auth rules (from `tech-stack.md` + `user-flow.md`)

- JWTs should be stored in **HttpOnly cookies**.
- Access tokens should be short-lived; refresh tokens must be revocable (blocklist in Redis if needed).
- OAuth login (Google/GitHub): provider callback → validate/create user in Postgres → issue session → redirect.

---

## Realtime rules (Socket.IO + Redis)

### Rooms and event naming (from `tech-stack.md` + `project-overview.md`)

- Rooms must be namespaced and predictable: `room_${conversationId}`.
- Events are lowercase snake: `join_room`, `leave_room`, `typing`, `message`, `disconnect`.
- Always clean up listeners on disconnect to prevent leaks.

### Scaling (from `tech-stack.md`)

- Use Redis Pub/Sub so multiple API instances can broadcast messages consistently.
- Presence tracking lives in Redis (sets); treat Redis as volatile unless persistence is explicitly configured.

---

## Data and performance rules (Postgres + Redis)

- Index for message retrieval: `conversation_id`, `created_at`.
- Avoid N+1s when fetching messages + senders; use explicit includes/selects.
- Prefer pagination by time/cursor for history (infinite scroll).

---

## Environment variables & secrets

- Maintain strict parity between **`.env.example`** and required env vars.
- Validate env vars at startup (fail fast).
- Never commit secrets; never bake `.env` into Docker images.

---

## Styling rules (Tailwind + shadcn)

- Prefer Tailwind tokens from `tailwind.config.*` for colors/spacing.
- If custom CSS is necessary, keep it minimal and use **BEM naming** for class selectors.
- Avoid `!important`.
- For conditional classes, use `clsx` + `tailwind-merge`.

---

## Documentation rules

- Any non-trivial decision must be recorded in `_docs/` (short and practical).
- Keep `_docs/user-flow.md` updated when flows change (auth, presence, messaging, uploads).

---

## Definition of Done (per feature)

- **Modular**: code lives under the correct feature/module folder.
- **Typed**: no `any`; validate external inputs.
- **Accessible**: keyboard + focus + semantic structure.
- **Secure**: cookies, CORS, rate limiting (when implemented), sanitized errors.
- **Realtime safe**: rooms/events consistent, disconnect cleanup present.
- **Testable**: logic is in services/utilities rather than UI/handlers.

