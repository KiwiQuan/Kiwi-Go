# Phase 00 — Setup (barebones, runnable, not yet “usable”)

## Goal
Create a **running full-stack skeleton** that matches `project-rules.md`, plus a tiny end-to-end “smoke slice” proving: **web ↔ api ↔ socket ↔ (db/redis)** are wired correctly.

## Scope (what “done” looks like)
- Web app loads and renders a basic app shell.
- API responds to `/health` and returns a clear status object.
- Socket.IO connects from web and can round-trip a test event.
- Postgres + Redis run locally via Docker Compose.
- You can run a scripted “demo” in < 3 minutes from a clean machine.

## Deliverables
- **Repo layout**: `apps/web`, `apps/api`, `packages/shared`, `infra/docker`, `_docs/`
- **Local runtime**: `docker compose up` for Postgres/Redis; one command to run web+api
- **Type safety baseline**: TS strict across apps + shared
- **Backend baseline**: three-layer boundaries established (routes → controllers → services → repositories)
- **Ops baseline**: env validation, request IDs, structured logs, sanitized errors

## Learning goals (senior fundamentals, early)
- **Make the “happy path” easy**: one command to run, one place to configure env.
- **Fail fast**: missing env → app refuses to boot with a helpful error.
- **Observable by default**: request IDs + JSON logs from day one.
- **Small experiments**: when something is new (Socket.IO / Redis / Docker), prove it in a tiny lab before integrating.

## Features (≤5 steps each)

### 1) Tooling & repo conventions baseline
- Pick a Node version (pin it) and a package manager (pin it).
- Add root scripts: `dev`, `lint`, `typecheck`, `test` (even if `test` is empty).
- Add formatter + linter config shared by both apps (or documented plan if deferred).
- Add `.env.example` at repo root and make it the “source of truth” for required env.
- Document “how to run locally” in `README.md` in 10–20 lines.

**Examples**

```bash
# pin Node (choose one approach)
node -v

# (example) if you use Volta:
volta pin node@20

# (example) if you use nvm:
nvm install 20 && nvm use 20
```

```json
// (example) root package.json scripts (keep them boring and predictable)
{
  "scripts": {
    "dev": "concurrently \"npm:dev:*\"",
    "dev:web": "npm --prefix apps/web run dev",
    "dev:api": "npm --prefix apps/api run dev",
    "lint": "npm -ws run lint",
    "typecheck": "npm -ws run typecheck",
    "test": "npm -ws run test"
  }
}
```

```env
# (example) .env.example (no secrets, just shape + defaults)
WEB_ORIGIN=http://localhost:5173
API_PORT=4000

DATABASE_URL=postgresql://postgres:postgres@localhost:5432/kiwi_go?schema=public
REDIS_URL=redis://localhost:6379
```

**Done checks**
- Fresh clone → install → `dev` runs without manual detective work.

### 2) Monorepo scaffold (web + api + shared)
- Create folders and minimal `package.json`/TS configs per `project-rules.md`.
- Add `packages/shared` with a tiny export (e.g., `SOCKET_EVENTS` map + DTO type).
- Configure TypeScript strict mode and path aliases (web/api import shared types).
- Enforce boundaries: apps can import from `packages/shared`, not from each other.
- Add `_docs/decisions/` folder (empty now; used later for ADRs).

**Examples**

```text
Kiwi-Go/
  apps/
    api/
    web/
  packages/
    shared/
  infra/
    docker/
  _docs/
    phases/
    decisions/
```

```ts
// packages/shared/src/socket-events.ts (example: prefer maps over enums)
export const SOCKET_EVENTS = {
  ping: "ping",
  pong: "pong",
  joinRoom: "join_room",
  leaveRoom: "leave_room",
} as const;

export type SocketEventName = (typeof SOCKET_EVENTS)[keyof typeof SOCKET_EVENTS];
```

```ts
// packages/shared/src/index.ts (example)
export * from "./socket-events";
```

**Done checks**
- `typecheck` succeeds and both apps can import a shared type.

### 3) Local infra (Docker Compose) for Postgres + Redis
- Add `infra/docker/compose.yml` with named volumes for Postgres persistence.
- Configure ports + credentials via env vars in `.env.example`.
- Add container healthchecks (Postgres ready + Redis ping).
- Add a “reset local db” command/runbook (clearly labeled as destructive).
- Prove connectivity from API using a `/health` check (DB + Redis ping).

**Examples**

```yaml
# infra/docker/compose.yml (example)
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: kiwi_go
    ports:
      - "5432:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d kiwi_go"]
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  pg_data:
```

```bash
# run infra
docker compose -f infra/docker/compose.yml up

# optional: destructive reset (ONLY for local dev)
docker compose -f infra/docker/compose.yml down -v
```

**Done checks**
- `docker compose up` → both services healthy; `/health` reports `db: ok`, `redis: ok`.

### 4) Mini-lab: Socket.IO “echo” (2 clients, 1 server)
- Create a tiny Socket.IO server that listens for `ping` and responds with `pong`.
- Create two tiny clients (browser tabs is fine) that connect and emit `ping`.
- Confirm both clients receive the server response (timestamp + clientId).
- Add a disconnect log to prove you understand connection lifecycle.
- Write down 3 pitfalls you hit (CORS, transport, reconnection) in `_docs/decisions/`.

**Examples**

```ts
// (example) server: listen for ping and reply with pong
// io.on("connection", (socket) => {
//   socket.on(SOCKET_EVENTS.ping, (payload) => {
//     socket.emit(SOCKET_EVENTS.pong, { ...payload, serverTime: Date.now() });
//   });
// });
```

```ts
// (example) client: connect and emit ping
// const socket = io("http://localhost:4000", { withCredentials: true });
// socket.emit(SOCKET_EVENTS.ping, { clientId: crypto.randomUUID(), sentAt: Date.now() });
// socket.on(SOCKET_EVENTS.pong, console.log);
```

**Done checks**
- You can explain: “what is a room”, “what is an event”, “what happens on disconnect”.

### 5) API skeleton (Express) with senior defaults
- Add request logging with a correlation/request ID per request.
- Add centralized error handling returning `{ code, message, requestId }`.
- Add env validation at startup (missing required env stops the server).
- Add `/health` and a versioned base path for future routes (e.g., `/api/v1`).
- Keep business logic out of routes (even if it’s just one service function).

**Examples**

```json
// (example) /health response shape
{
  "status": "ok",
  "requestId": "req_123",
  "checks": { "db": "ok", "redis": "ok" }
}
```

```ts
// (example) error response shape (never include stack traces)
// res.status(400).json({ code: "bad_request", message: "Invalid input", requestId })
```

**Done checks**
- Force an error → client receives sanitized error + requestId; logs include requestId.

### 6) Socket.IO integration inside the API (real app wiring)
- Add Socket.IO server wiring under `apps/api/src/realtime/` (separate from HTTP).
- Add handshake auth placeholder (accept all for now; real auth comes later).
- Implement `join_room` and `leave_room` events that join `room_${conversationId}`.
- Implement `ping`/`pong` as a baseline event contract using shared constants.
- Ensure disconnect cleanup doesn’t leak listeners (one connection = one set of handlers).

**Examples**

```ts
// (example) room naming convention from project rules
// const roomName = `room_${conversationId}`;
// socket.join(roomName);
// io.to(roomName).emit("message", messagePayload);
```

**Done checks**
- From web, you can join a “room” and get a room-scoped broadcast.

### 7) Web skeleton (React + TS + Tailwind + shadcn-ready)
- Create semantic layout landmarks (`header`, `nav/aside`, `main`) per UI rules.
- Add a “status panel” showing API health and socket connection status.
- Add a “Send ping” button (`<button>`) and render the latest `pong`.
- Add focus-visible styles and verify keyboard navigation works.
- Keep state minimal and local (Redux comes later when it’s useful).

**Examples**

```html
<!-- (example) semantic landmarks -->
<header></header>
<main></main>
```

```tsx
// (example) accessible button
// <button type="button" onClick={handlePing} className="rounded-md border px-3 py-2 focus-visible:outline-none focus-visible:ring-2">
//   Send ping
// </button>
```

**Done checks**
- Web loads, shows green “API ok / socket connected”, and `ping` returns `pong`.

## Exit criteria (demo script)
- Start infra: Postgres + Redis.
- Start apps: web + api.
- Visit web: see **API ok** and **Socket connected**.
- Click “Send ping”: see `pong` payload and a matching server log entry.

