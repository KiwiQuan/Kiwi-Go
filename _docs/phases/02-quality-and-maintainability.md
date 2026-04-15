# Phase 02 — Quality & Maintainability (senior defaults)

## Goal
Turn the MVP into something you can **safely change**: consistent contracts, observability, testing, and predictable architecture boundaries—without adding lots of new product surface area.

## Scope
- Keep MVP features working while upgrading quality.
- Add tests for core business logic and critical API routes.
- Add consistent validation + typed DTOs across web/api/shared.
- Improve accessibility, error states, and performance guardrails.

## Deliverables
- **Contracts**: shared request/response + socket event schemas in `packages/shared`
- **Validation**: runtime validation at boundaries (API input, env, socket payloads)
- **Testing**: unit tests for services + integration tests for key routes
- **Observability**: structured logs, request IDs everywhere, basic metrics hooks
- **DX**: consistent lint/format, pre-commit checks, predictable module boundaries

## Learning goals
- **Boundary-first thinking**: validate external inputs; keep internal types trusted.
- **Service-centric design**: business rules in services, not routes or UI.
- **Testing strategy**: pyramid (unit > integration > e2e) with fast feedback loops.
- **Accessibility mindset**: keyboard-first flows, semantic landmarks, focus management.

## Features

### 1) Shared contracts (types + schemas)
- Define DTOs for `User`, `Conversation`, `Message`, and paginated responses.
- Define Socket.IO event payload types for `join_room`, `message`, `typing`, `presence`.
- Add runtime schemas (e.g., zod) for inbound API + socket payload validation.
- Ensure web uses the same DTO types (no duplicate “frontend-only” shapes).
- Add a versioning note in `_docs/` for how you’ll evolve contracts safely.

### 2) Error handling & UI resilience
- Standardize API error shape (code, message, requestId) and enforce it.
- Add a single error translation layer in web (toast + inline states).
- Add guarded empty/loading/error states to conversations + messages views.
- Add optimistic UI failure handling for message send (retry + “failed” badge).
- Confirm no stack traces leak to clients in any environment.

### 3) Testing foundation (api)
- Unit test message service: membership checks, persistence call, emit contract.
- Integration test: auth-protected route + message history pagination.
- Add deterministic test DB setup (migrate, seed, cleanup).
- Add a Redis test strategy (real container or in-memory substitute) for presence logic.
- Add CI-friendly test scripts (no “works on my machine” assumptions).

### 4) Performance guardrails (web)
- Add message list virtualization with stable keys and memoized rows.
- Prevent “store everything” Redux usage (keep large history in component state/cache).
- Add basic profiling checklist for re-renders (documented in `_docs/`).
- Ensure RTK Query caching + invalidation is deliberate and minimal.
- Add a smoke performance budget (e.g., max render time for 1k messages).

### 5) Senior-grade project hygiene
- Add env parity rules enforcement (fail startup if `.env` keys missing).
- Add consistent naming conventions enforcement via lint rules where possible.
- Add a small ADR template under `_docs/decisions/` and write 1 ADR (contracts).
- Add “Definition of Done” checklist to PR/issue template (optional, but recommended).
- Confirm folder structure matches `project-rules.md` and no drift exists.

## Exit criteria (demo script)
- MVP still works end-to-end.
- Break a contract intentionally → you get a **validated** error, not undefined behavior.
- Run tests locally → green; run in CI → green.
- Accessibility spot-check: keyboard navigation works across the main flow.

