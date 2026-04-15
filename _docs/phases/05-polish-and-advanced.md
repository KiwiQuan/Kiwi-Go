# Phase 05 — Polish + Advanced Features (feature-rich, portfolio-ready)

## Goal
Add the “nice to have” product features and deeper system design topics (search, receipts, notifications, encryption) while keeping performance and reliability strong.

## Scope
- UX polish + advanced messaging capabilities.
- Optional “stretch” items that teach modern senior-level trade-offs.
- Choose features based on interest; keep the app stable at every step.

## Deliverables
- **Enhancements**: read receipts, reactions, message editing/deleting
- **Discovery**: message search + filtering
- **Notifications**: push notifications (web) or email digests (optional)
- **Security depth**: optional end-to-end encryption exploration
- **Observability**: better metrics + tracing mindset (even if minimal tooling)

## Learning goals
- **Product thinking**: prioritize user value, reduce friction, handle edge cases.
- **Scalability trade-offs**: indexes, search strategies, caching, background work.
- **Senior UX**: accessibility, latency hiding, reliable optimistic updates.

## Features

### 1) Read receipts (room-level correctness)
- Add message “delivered/read” state model (per user + message).
- Emit receipt events via sockets when messages become visible.
- Persist receipts in Postgres with sensible indexing.
- Render receipts minimally (e.g., last message read indicator).
- Validate correctness with multi-tab tests and reconnection scenarios.

### 2) Reactions + message actions
- Add reactions model and endpoints/events.
- Add optimistic UI for reactions with rollback on failure.
- Add edit/delete with authorization rules (owner-only or role-based).
- Ensure audit-friendly behavior (soft delete + “message removed” placeholder).
- Add basic moderation hooks (report/flag pipeline stub).

### 3) Search (start simple, evolve)
- Implement Postgres full-text search for messages in a conversation.
- Add UI search box with debounced queries and highlighted matches.
- Add pagination + filters (sender, date range).
- Add indexes to keep search responsive.
- Document when to graduate to Elasticsearch/OpenSearch (decision record).

### 4) Notifications (choose one learning path)
- Implement web push notifications for mentions/DMs (service worker path), OR
- Implement email digests for unread messages (background job path).
- Store notification preferences per user.
- Add “mute conversation” and “mentions only” preference.
- Add an unsubscribe/disable flow and test it end-to-end.

### 5) Optional: End-to-end encryption (learning-focused)
- Define threat model + constraints in an ADR (what E2EE protects/doesn’t).
- Prototype client-side encryption for 1:1 messages (key exchange choice documented).
- Store only ciphertext on server; keep metadata minimal.
- Handle multi-device key management at a basic level (documented limitations).
- Add a “verification” UX step (safety number / fingerprint concept).

## Exit criteria (demo script)
- MVP+deployment still works.
- Search returns correct results quickly.
- Reactions and receipts are consistent across tabs/devices.
- At least one “advanced” path (notifications or E2EE) is implemented and documented.

