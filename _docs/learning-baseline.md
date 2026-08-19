# Kiwi_Go Learning Baseline (Universal)

## Purpose

This document is a reusable context handoff for new chats across all phases of the Kiwi_Go project.  
It captures my current understanding, learning preferences, and agreed engineering mindset so each new phase can start fast without re-explaining everything.

---

## How To Use This In New Chats

At the top of each new phase chat, include this file and fill in:

- Current phase:
- Completed scope:
- Current goals:
- Open blockers:

Everything else below should remain mostly stable and only be updated when my understanding changes.

---

## Project Context (Stable)

- Project: real-time messaging platform (learning-focused, production-minded)
- Architecture direction: monorepo
- Stack commitments:
  - React + TypeScript
  - Node + Express
  - Socket.IO
  - Prisma + PostgreSQL
  - Redis
  - Redux Toolkit / RTK Query
  - OAuth (Google + GitHub)
  - Docker + AWS deployment path
- Coding maturity goal: develop senior-level engineering thinking while building

---

## Learner Profile

- Background: primarily PERN stack
- New tech baseline: started from zero in Prisma, Redis, Docker, AWS, Socket.IO implementation
- TypeScript: recently learned, needs real project repetition
- Redux Toolkit: learned previously, currently less fresh than TypeScript
- Preferred learning style: **small concept -> mini exercise -> reflection -> integration**

---

## Tutor Contract (How I Want To Be Guided)

- Don’t give code unless I explicitly ask for code.
- Teach with leading questions and hints, not just final answers.
- Break work into small, testable steps.
- Push me to reason about:
  - trade-offs
  - edge cases
  - failure modes
  - naming clarity
  - testability
- Reinforce senior principles consistently:
  - DRY
  - SRP
  - KISS
  - modular design
- Prefer modular JS/TS patterns (pure functions, closures, ES modules), avoid class-heavy patterns by default.

---

## Confirmed Understanding So Far

### 1) HTTP vs WebSocket

- HTTP is request/response and typically short-lived per request.
- WebSocket keeps a persistent connection for bidirectional real-time communication.

### 2) Docker mental model

- Docker helps make runtime behavior consistent across machines by packaging app/runtime/dependencies into an image and running it as a container.
- Remaining growth: practical Docker workflow and debugging container issues.

### 3) Prisma mental model

- Prisma is an ORM for typed database access.
- Key concept learned: migrations are versioned schema changes and must be reproducible.
- Remaining growth: migration lifecycle in real feature work.

### 4) OAuth vs JWT

- OAuth: identity/login delegation through provider (Google/GitHub).
- JWT: token-based session/auth mechanism after login.
- Remaining growth: secure refresh/revocation lifecycle in production patterns.

### 5) Redis role

- Redis is strong for fast ephemeral state (presence/session/cache/pub-sub coordination).
- PostgreSQL remains source of truth for durable business data.

---

## Architectural Truths Agreed For This Project

- Durable records (users, conversations, messages, membership, last-seen) belong in PostgreSQL.
- Ephemeral/high-speed coordination (online presence set, pub/sub fan-out, short-lived cache/session) belongs in Redis.
- Safer message flow is: **persist in DB first -> then broadcast** to prevent phantom messages.
- In multi-instance deployments, Redis pub/sub (and Socket.IO Redis adapter) is required for cross-instance real-time consistency.
- Presence systems must handle disconnect and multi-tab edge cases to avoid false online/offline states.

---

## Current Strengths

- Honest about unknowns (asks instead of guessing)
- Good foundational reasoning on realtime concepts
- Good early debugging instincts via data-flow tracing
- Motivated to connect concept -> implementation directly

---

## Current Growth Areas (Ongoing)

- Docker fundamentals and day-to-day commands
- Prisma schema + migration confidence
- Redis command fluency and mental model depth
- Socket lifecycle cleanup and reliability patterns
- RTK Query practical data-flow patterns
- TypeScript strict-mode confidence through repeated use

---

## Senior Mindset Commitments

- Define success criteria before implementation.
- Favor predictable architecture over clever shortcuts.
- Design for maintainability, not just immediate functionality.
- Think beyond happy path (network failures, invalid payloads, reconnects, race conditions).
- Name code by intent (`isConnected`, `hasValidSession`, etc.).
- Ask “How would this be tested?” before coding.
- Evaluate explicit trade-offs (speed vs readability vs scalability).

---

## Debugging Protocol (Default)

1. Reproduce consistently.
2. Narrow scope (where data diverges from expectation).
3. Form one hypothesis at a time.
4. Instrument and validate (logs/devtools/network/event flow).
5. Confirm fix and regression-check adjacent flows.
6. Document root cause + prevention note.

---

## Reusable Phase Start Template

Use this at the beginning of any new phase chat:

- **Current phase:**
- **Phase objective:**
- **Definition of done:**
- **New technologies in this phase:**
- **Known risks:**
- **Mini-labs required first:**
- **Architecture constraints to preserve:**
- **What I want to practice most in this phase (TS/Redux/Debugging/System design):**

---

## Reusable Phase End Reflection Template

- What did I build?
- What broke and why?
- Which assumptions were wrong?
- What did I learn technically?
- What did I learn about engineering judgment?
- What should be improved before the next phase?

---

## Update Log

Keep this short and append-only.

- Initial universal baseline created.
- (Add dated entries as understanding evolves.)
