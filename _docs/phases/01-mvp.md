# Phase 01 — MVP (minimal usable messaging app)

## Goal
Ship a **usable** real-time chat experience with OAuth login, presence, conversations, and persisted messages—aligned to `_docs/user-flow.md`.

## Scope (what “usable” means)
- Users can **log in** (Google/GitHub), land on dashboard, and stay signed in.
- Users can **see conversations**, open one, load history, and **send messages** in real-time.
- Users can see **online/offline presence** and typing indicators.
- Messages persist in Postgres; realtime fanout works in a single API instance (scaling comes later).

## Deliverables
- **Auth**: OAuth + JWT session in **HttpOnly cookie** (access token)
- **Core data model**: Users, Conversations, Messages (Prisma migrations)
- **Messaging**: join room, send message, persist + broadcast
- **Presence**: Redis set tracks online users + last seen in Postgres
- **UX baseline**: responsive layout + accessible keyboard navigation

## Learning goals (focus areas)
- **Prisma**: schema design, migrations, typed queries, avoiding N+1.
- **RTK Query**: server-state patterns (conversations + message pagination).
- **Socket.IO**: rooms, event contracts, optimistic UI, reconnect edge cases.
- **Auth**: OAuth callbacks, cookie sessions, protecting routes/websocket connections.

## Features

### 1) OAuth login + session (Google/GitHub)
- Implement OAuth start + callback endpoints (provider → backend → user upsert).
- Issue JWT access token and set it in an HttpOnly cookie.
- Add auth middleware for REST routes (validate cookie JWT).
- Authenticate Socket.IO handshake using the same cookie JWT.
- Add frontend login page + callback handling + protected app shell.

### 2) Database modeling + repositories (Prisma)
- Create Prisma models: `User`, `Conversation`, `ConversationMember`, `Message`.
- Add indexes for message history retrieval (`conversationId`, `createdAt`).
- Implement repository functions for conversations and messages (Prisma-only).
- Implement service layer orchestration (validation + authorization + persistence).
- Seed minimal dev data (optional) to unblock UI development.

### 3) Conversation list + routing (web)
- Fetch conversation list via RTK Query and render in sidebar/drawer.
- Implement “select conversation” state and deep-linkable route (if using routing).
- Show participants + last message preview (minimal).
- Add empty states (no conversations, not found).
- Ensure mobile-first: sidebar becomes a drawer on small screens.

### 4) Room join + message history (infinite scroll)
- On conversation open, emit `join_room` with `conversationId`.
- Load the latest N messages via REST (RTK Query) and render list.
- Implement pagination on scroll-up using cursor/time-based queries.
- Virtualize the message list (baseline windowing) per project rules.
- Ensure message author data loads without N+1 query patterns.

### 5) Send message (optimistic UI + persistence + broadcast)
- Implement message composer with accessible form semantics.
- Optimistically add message with `sending/failed` state in UI.
- On send: service validates membership → persist to Postgres → emit to room.
- On receive: append message in UI without storing huge history in Redux.
- Add basic retry for failed sends (manual “retry” action).

### 6) Presence (Redis) + last seen
- On socket connect: add user ID to Redis online set; broadcast presence update.
- On disconnect: remove from set; write `lastSeenAt` to Postgres; broadcast update.
- Render presence indicators in conversation list / header.
- Add “last seen” text for offline users (minimal formatting).
- Ensure reconnect doesn’t leave “ghost online” state (best-effort cleanup).

### 7) Typing indicators
- Emit `typing` events with debounce/throttle from composer.
- Broadcast typing to the room excluding sender.
- Render “X is typing…” UI with timeout expiration.
- Ensure typing state clears on send and on conversation change.
- Keep event contracts in `packages/shared` types.

## Exit criteria (demo script)
- Login via Google/GitHub → redirected into dashboard.
- See online indicator for users currently connected.
- Open a conversation → history loads → scroll up loads older messages.
- Send a message → optimistic bubble appears → persists and arrives in another tab.
- See typing indicator from another tab/user and presence updates on disconnect.

