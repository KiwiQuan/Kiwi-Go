# Phase 00 — Setup (barebones, runnable, not yet “usable”)

## Goal
Create a **running full-stack skeleton** that matches the repo + architecture rules, with a tiny end-to-end slice (web ↔ api ↔ db/redis) to prove the wiring works.

## Scope (what “done” looks like)
- Monorepo structure exists and runs locally.
- Web app loads, API responds, Socket.IO connects.
- Postgres + Redis run via Docker Compose.
- One “smoke” flow works: open web → connect socket → send test event → server logs → optional DB write.

## Deliverables
- **Repo layout**: `apps/web`, `apps/api`, `packages/shared`, `infra/docker`, `_docs/`
- **Local runtime**: `docker compose up` runs Postgres + Redis; `pnpm dev` (or npm) runs web + api
- **Type safety baseline**: TS strict on both apps + shared package compiling cleanly
- **Quality baseline**: ESLint + Prettier + import boundaries (no cross-feature dumping)
- **Ops baseline**: env validation at startup; structured logging; global error handler

## Learning goals (new tech + senior fundamentals)
- **Monorepo boundaries**: keep “shared” truly shareable (no Node/DOM-only code).
- **Three-layer backend**: routes/controllers/services/repositories separation from day 1.
- **Runtime hygiene**: env validation, request IDs, structured logs, consistent error shapes.
- **Realtime basics**: Socket.IO handshake + auth placeholder + clean disconnect handling.

## Features (each ends in a runnable increment)

### 1) Monorepo scaffold (web + api + shared)
- Create folders: `apps/web`, `apps/api`, `packages/shared`, `infra/docker`.
- Configure TypeScript strict mode for both apps and shared.
- Add shared type export surface (`packages/shared/src/index.ts`).
- Add workspace scripts to run web + api together.
- Add `_docs/decisions/` folder for future ADR-style notes (empty is fine).

### 2) Local infrastructure via Docker Compose (Postgres + Redis)
- Add `infra/docker/compose.yml` with Postgres + Redis services.
- Add `.env.example` with `DATABASE_URL`, `REDIS_URL`, `WEB_ORIGIN`, `PORT`.
- Add a “reset/dev” workflow (documented commands; no destructive production assumptions).
- Add healthchecks for Postgres/Redis containers.
- Confirm web + api can read env vars without crashing.

### 3) API skeleton (Express) with senior defaults
- Add `apps/api/src/app.ts` with middleware wiring (CORS, JSON, request ID).
- Add global error middleware returning a sanitized error shape.
- Add structured logger (JSON) and log correlation ID per request.
- Add `/health` route with DB + Redis connectivity checks (simple ping).
- Add env validation at startup (fail fast if required keys missing).

### 4) Realtime skeleton (Socket.IO) with safe cleanup
- Add Socket.IO server wiring separated under `apps/api/src/realtime/`.
- Implement `connection` and `disconnect` handlers with logging.
- Implement a `ping` event that echoes back to prove the channel works.
- Ensure listeners are removed / per-socket state cleared on disconnect.
- Confirm web can connect and receive the echo.

### 5) Web skeleton (React + TS + Tailwind + shadcn-ready)
- Create base layout following UI rules (mobile-first, sidebar placeholder).
- Add an API health check call and render status.
- Add a socket connect indicator + “Send test event” button.
- Add accessible focus styles and semantic landmarks (`header/nav/main`).
- Confirm hot reload works and no console errors on load.

## Exit criteria (demo script)
- Start infra: Postgres + Redis running.
- Start apps: web + api.
- Visit web: see **API healthy** and **Socket connected**.
- Click “Send test event”: receive server response; see log with request/socket ID.

