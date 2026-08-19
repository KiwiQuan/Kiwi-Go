# Kiwi_Go Learning Baseline (Pre-Phase 00)

## Context Snapshot

- Starting point: **Phase 00**
- Current project state: **no code yet**
- Target architecture: **exact monorepo layout from `project-rules.md`**
- Confirmed stack choices:
  - Prisma ORM: **Yes**
  - Socket.IO: **Yes**
  - RTK / RTK Query: **Yes**
  - OAuth providers in Phase 01: **Google + GitHub**
  - Deployment target: **AWS**
- Stated baseline for new tech: **0 experience**
- Additional goal: solidify **TypeScript** and improve **Redux Toolkit** confidence

---

## Q&A Session 1 (Initial Baseline)

### Q1) Difference between HTTP and WebSocket?

**Your answer:**  
HTTP is one-time request/response and closes; WebSocket stays open and allows two-way ongoing updates without re-requesting each time.

**Tutor feedback:**  
Correct and practical understanding.

---

### Q2) Comfort with TypeScript strict typing?

**Your answer:**  
Recently completed TypeScript bootcamp; fairly comfortable conceptually, but little real project practice. Redux knowledge is older and less fresh.

**Tutor feedback:**  
Good conceptual base; needs repetition and real usage.

---

### Q3) Docker experience?

**Your answer:**  
No real Docker experience yet.

**Tutor feedback:**  
Noted as beginner area for guided learning.

---

### Q4) What is ORM migration?

**Your answer:**  
Know ORM acronym, not sure what migrations are.

**Tutor explanation:**  
A migration is a versioned, reproducible database schema change tracked in source control.

---

### Q5) OAuth vs JWT?

**Your answer:**  
OAuth is login via provider (Google/GitHub/etc); JWT is session/auth token after login.

**Tutor feedback:**  
Correct mental model.

---

### Q6) Current debugging process?

**Your answer:**  
Use console logs and trace data flow with hardcoded examples.

**Tutor feedback:**  
Good start; will level up with hypothesis-based debugging and observability.

---

### Q7) Preferred learning style?

**Your answer:**  
A) small concept -> mini exercise -> reflect.

**Tutor feedback:**  
Excellent fit for this project approach.

---

## Q&A Session 2 (Concept Reinforcement)

### Q1) Define Docker / Prisma / Socket.IO

**Your answer (summary):**

- Docker helps app run consistently across machines by packaging environment into image/container.
- Prisma is a way to interact with DB without hand-writing all SQL.
- Socket.IO adds reliability features on top of WebSockets (reconnect/fallback abstractions).

**Tutor upgrades:**

- Docker image is built from a recipe (`Dockerfile`), container is a running instance.
- Node pinning is usually `.nvmrc` / Volta / engines (not only package.json).
- Socket cleanup includes app-level cleanup (timers, presence state, listeners), not just socket disconnect.

---

### Q2) Why strict typing helps?

**Your answer:**  
Catches type errors early and reduces long-term bugs.

**Tutor feedback:**  
Correct.

---

### Q3) Why “works on my machine” happens and how Docker helps?

**Your answer:**  
Version/tool/environment mismatches across machines; Docker packages dependencies/runtime environment for consistency.

**Tutor feedback:**  
Correct.

---

### Q4) Room scoping and cleanup?

**Your answer:**  
Room scoping determines who receives events; cleanup prevents disconnected clients/state from accumulating.

**Tutor feedback:**  
Correct, with note that app-level state cleanup is critical.

---

### Q5) Why single Prisma client?

**Your answer:**  
Not fully sure yet.

**Tutor explanation:**  
Main reason is connection management/stability; many instances can exhaust DB connections, especially during dev reloads/scaling.

---

## Q&A Session 3 (Architecture Drill)

### Prompt area (you initially said unsure)

You were unsure about:

- Postgres vs Redis responsibilities
- Message source of truth order
- Multi-instance behavior without Redis pub/sub
- Presence edge cases

### Tutor target model:

- **Postgres**: durable truth (users, conversations, messages, memberships, last-seen)
- **Redis**: ephemeral fast state (online set, pub/sub fan-out, cache/session)
- **Safer message flow**: persist to DB first, then broadcast
- **Without Redis pub/sub at scale**: cross-instance real-time delivery breaks
- **Presence edge cases**: ghost-online on abrupt disconnect; multi-tab correctness issues

---

## Quick Validation Answers

### Q1) Why Redis for presence but not message history?

**Your answer:**  
Redis is good for short-term/session/cache; Postgres should store message history.

**Tutor verdict:**  
Correct.

### Q2) Why DB first then broadcast?

**Your answer:**  
If DB write fails, users may see a message that does not exist in DB.

**Tutor verdict:**  
Correct.

---

## Current Understanding Summary

### Strong areas

- HTTP vs WebSocket core concept
- Basic OAuth vs JWT distinction
- Good intuitive debugging habit
- Good reasoning honesty (“I don’t know” when needed)

### Growth areas (Phase 00 focus)

- Docker fundamentals and workflow
- Prisma migrations + schema lifecycle
- Redis mental model (ephemeral vs durable)
- Socket.IO lifecycle and cleanup patterns
- RTK/RTK Query practical usage
- TypeScript strict-mode real project practice

---

## Agreed Phase 00 Execution Order

1. Monorepo folder skeleton
2. TS/workspace strict baseline
3. Docker Compose infra (Postgres + Redis)
4. API foundation (`/health`, env validation, request IDs, sanitized errors)
5. Socket.IO smoke lab (`ping/pong`, room join/leave, disconnect)
6. Web shell + socket/API status UI
7. Decision notes in docs

---

## Senior Mindset Commitments

- Define success criteria before implementation
- Prefer predictable architecture over clever shortcuts
- Think in edge cases, not just happy path
- Name for intent (`isConnected`, `hasValidSession`, etc.)
- Ask “How would I test this?” before coding logic
