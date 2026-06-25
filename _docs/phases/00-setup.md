# Phase 00 — Setup (barebones, runnable, not yet "usable")

## Goal
Create a **running full-stack skeleton** that matches `project-rules.md`, plus a tiny end-to-end "smoke slice" proving: **web ↔ api ↔ socket ↔ (db/redis)** are wired correctly.

## Scope (what "done" looks like)
- Web app loads and renders a basic app shell.
- API responds to `/health` and returns a clear status object.
- Socket.IO connects from web and can round-trip a test event.
- Postgres + Redis run locally via Docker Compose.
- You can run a scripted "demo" in < 3 minutes from a clean machine.

## Deliverables
- **Repo layout**: `apps/web`, `apps/api`, `packages/shared`, `infra/docker`, `_docs/`
- **Local runtime**: `docker compose up` for Postgres/Redis; one command to run web+api
- **Type safety baseline**: TS strict across apps + shared
- **Backend baseline**: three-layer boundaries established (routes → controllers → services → repositories)
- **Ops baseline**: env validation, request IDs, structured logs, sanitized errors

## Learning goals (senior fundamentals, early)
- **Make the "happy path" easy**: one command to run, one place to configure env.
- **Fail fast**: missing env → app refuses to boot with a helpful error.
- **Observable by default**: request IDs + JSON logs from day one.
- **Small experiments**: when something is new (Socket.IO / Redis / Docker), prove it in a tiny lab before integrating.

---

## Features

### 1) Tooling & repo conventions baseline

- [ ] Pick a Node version and pin it (use Volta or `.nvmrc`).
- [ ] Add root scripts: `dev`, `lint`, `typecheck`, `test` (even if `test` is empty for now).
- [ ] Add formatter + linter config shared by both apps (ESLint + Prettier).
- [ ] Add `.env.example` at repo root as the single source of truth for required env vars.
- [ ] Document "how to run locally" in `README.md` in 10–20 lines.

**Examples**

```bash
# Pin Node with Volta
volta pin node@20

# Or with nvm — add a .nvmrc file at root
echo "20" > .nvmrc && nvm use
```

```json
// root package.json — keep scripts boring and predictable
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
# .env.example — shapes only, no real secrets
WEB_ORIGIN=http://localhost:5173
API_PORT=4000

DATABASE_URL=postgresql://postgres:postgres@localhost:5432/kiwi_go?schema=public
REDIS_URL=redis://localhost:6379
```

**Done checks**
- [ ] Fresh clone → install → `dev` runs without manual detective work.

---

### 2) Monorepo scaffold (web + api + shared)

- [ ] Create folder structure per `project-rules.md` (see example below).
- [ ] Add `packages/shared` with a tiny export (a `SOCKET_EVENTS` map + one DTO type).
- [ ] Configure TypeScript strict mode and path aliases so web/api can import shared types.
- [ ] Enforce boundaries: apps can import from `packages/shared`, never from each other.
- [ ] Add `_docs/decisions/` folder (empty now — used for ADR notes later).

**Examples**

```text
Kiwi-Go/
  apps/
    api/           ← Node + Express + Socket.IO
    web/           ← React + TS + Tailwind
  packages/
    shared/        ← types + event maps (no Node/DOM-only code here)
  infra/
    docker/        ← compose files + Dockerfiles
  _docs/
    phases/
    decisions/     ← lightweight decision records
```

```ts
// packages/shared/src/socket-events.ts
// Use a map instead of an enum — avoids TypeScript enum pitfalls
export const SOCKET_EVENTS = {
  ping: "ping",
  pong: "pong",
  joinRoom: "join_room",
  leaveRoom: "leave_room",
} as const;

export type SocketEventName = (typeof SOCKET_EVENTS)[keyof typeof SOCKET_EVENTS];
```

```ts
// packages/shared/src/index.ts
export * from "./socket-events";
```

**Done checks**
- [ ] `typecheck` passes with no errors across all workspaces.
- [ ] Both apps can import a type from `packages/shared`.

---

### 3) Local infra (Docker Compose) for Postgres + Redis

- [ ] Add `infra/docker/compose.yml` with Postgres + Redis services and named volumes.
- [ ] Configure all ports/credentials via env vars (pull from `.env` file, not hardcoded).
- [ ] Add container healthchecks so you know when each service is actually ready.
- [ ] Add a clearly-labeled "reset local db" runbook command (destructive — local only).
- [ ] Prove connectivity from the API via a `/health` endpoint that pings both DB + Redis.

**Examples**

```yaml
# infra/docker/compose.yml
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
# Start infra
docker compose -f infra/docker/compose.yml up -d

# ⚠️ Destructive reset — ONLY run locally
docker compose -f infra/docker/compose.yml down -v
```

**Done checks**
- [ ] `docker compose up` → both services show as healthy.
- [ ] `GET /health` returns `{ "checks": { "db": "ok", "redis": "ok" } }`.

---

### 4) Mini-lab: Socket.IO "echo" (1 server, 2 clients)

> **Why this lab exists**: Socket.IO is completely new. Before wiring it into the real app, spend 1–2 days here. Build the smallest possible thing so you understand how connections, events, and rooms actually work — without the noise of a full project around it.

- [ ] Create a bare-bones Socket.IO server (separate file, not the main API yet) that listens for a `ping` event and emits back a `pong`.
- [ ] Connect to it from two separate browser tabs and confirm both receive the `pong`.
- [ ] Add a disconnect log on the server to observe the connection lifecycle.
- [ ] Try joining a named room and broadcasting only to that room (not all clients).
- [ ] Write down 3 things that confused you or failed (CORS? transport? reconnection?) in `_docs/decisions/socket-io-notes.md`.

**Examples**

```ts
// Mini-lab server (standalone file, not part of the real API yet)
// io.on("connection", (socket) => {
//   console.log("connected:", socket.id);
//
//   socket.on("ping", (payload) => {
//     socket.emit("pong", { ...payload, serverTime: Date.now() });
//   });
//
//   socket.on("disconnect", (reason) => {
//     console.log("disconnected:", socket.id, reason);
//   });
// });
```

```ts
// Mini-lab client (run this in the browser console or a tiny HTML file)
// const socket = io("http://localhost:4000", { withCredentials: true });
// socket.emit("ping", { clientId: crypto.randomUUID(), sentAt: Date.now() });
// socket.on("pong", (data) => console.log("pong received:", data));
```

**Done checks**
- [ ] Two browser tabs both receive `pong` from the same server.
- [ ] You can explain in your own words: what is a room, what is an event, what happens on disconnect.
- [ ] Notes written in `_docs/decisions/socket-io-notes.md`.

---

### 5) API skeleton (Express) with senior defaults

- [ ] Add request logging middleware that attaches a unique `requestId` to every request.
- [ ] Add a centralized error handler as the last middleware — always returns `{ code, message, requestId }`.
- [ ] Add env validation at startup: if a required env var is missing, log the missing key and crash (fail fast).
- [ ] Add `/health` route and a versioned base path (`/api/v1`) for future routes.
- [ ] Keep any business logic out of route handlers — even the `/health` logic should live in a service function.

**Examples**

```ts
// Env validation at startup — crash early with a helpful message
// const requiredEnv = ["DATABASE_URL", "REDIS_URL", "WEB_ORIGIN"];
// for (const key of requiredEnv) {
//   if (!process.env[key]) throw new Error(`Missing required env var: ${key}`);
// }
```

```json
// /health response shape
{
  "status": "ok",
  "requestId": "req_7f3a",
  "checks": { "db": "ok", "redis": "ok" }
}
```

```ts
// Error response shape — never leak stack traces to the client
// res.status(400).json({ code: "bad_request", message: "Invalid input.", requestId })
```

**Done checks**
- [ ] Delete a required env var, restart the server → it crashes with a clear error message.
- [ ] Trigger an unhandled error → client receives `{ code, message, requestId }`, no stack trace.
- [ ] Logs include `requestId` on every line for that request.

---

### 6) Socket.IO integration into the real API

- [ ] Add Socket.IO server wiring under `apps/api/src/realtime/` (attach to the same HTTP server as Express).
- [ ] Add a handshake auth placeholder middleware (accept all connections for now; real auth comes in Phase 01).
- [ ] Implement `join_room` and `leave_room` handlers that call `socket.join("room_${conversationId}")`.
- [ ] Wire up `ping`/`pong` using the event name constants from `packages/shared`.
- [ ] On `disconnect`, remove all per-socket state and log the reason (prevents listener leaks).

**Examples**

```ts
// Room naming — must match the convention in project-rules.md
// const roomName = `room_${conversationId}`;
// socket.join(roomName);

// Broadcast to everyone in the room (including the sender)
// io.to(roomName).emit(SOCKET_EVENTS.message, payload);

// Broadcast to everyone EXCEPT the sender
// socket.to(roomName).emit(SOCKET_EVENTS.typing, payload);
```

```ts
// Disconnect cleanup — always do this to prevent ghost listeners
// socket.on("disconnect", (reason) => {
//   console.log(`[socket] ${socket.id} disconnected — reason: ${reason}`);
//   // remove any per-socket state, timers, or subscriptions here
// });
```

**Done checks**
- [ ] Web can connect to the API socket and join a room.
- [ ] A message emitted to `room_${id}` is only received by sockets in that room, not all connected clients.

---

### 7) Web skeleton (React + TS + Tailwind + shadcn-ready)

- [ ] Create the base layout using semantic HTML landmarks (`<header>`, `<aside>`, `<main>`).
- [ ] Add a status panel component that calls `/health` and shows the socket connection state.
- [ ] Add a `<button>` that emits `ping` and renders the latest `pong` response on screen.
- [ ] Add `focus-visible` styles and verify you can tab through all interactive elements.
- [ ] Keep all state local for now (`useState`) — Redux/RTK Query gets introduced in Phase 01 when it earns its place.

**Examples**

```html
<!-- Semantic layout landmarks — per project-rules.md accessibility requirements -->
<header><!-- app bar / nav --></header>
<div class="flex">
  <aside><!-- sidebar / drawer --></aside>
  <main><!-- primary content --></main>
</div>
```

```tsx
// Accessible button with visible focus ring (Tailwind)
// <button
//   type="button"
//   onClick={handlePing}
//   className="rounded-md border px-3 py-2 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-offset-2"
// >
//   Send ping
// </button>
```

**Done checks**
- [ ] Web loads with no console errors.
- [ ] Status panel shows "API ok" and "Socket connected" when both are running.
- [ ] Clicking "Send ping" renders the pong response on screen.
- [ ] All buttons are reachable and visibly focused via keyboard only.

---

## Exit criteria (demo script)

- [ ] Start infra: `docker compose up` → Postgres + Redis healthy.
- [ ] Start apps: API + web both running.
- [ ] Visit web: see **API ok** and **Socket connected**.
- [ ] Click "Send ping": see pong payload rendered in the UI.
- [ ] Check server logs: see `requestId`, socket ID, and room join logged.
