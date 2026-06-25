# Phase 05 — Polish + Advanced Features (portfolio-ready)

## Goal
Add the features that make the app feel **complete and professional**: read receipts, reactions, message search, group management, and web push notifications. This phase is also where you develop a **senior product mindset** — every feature decision has a trade-off, and you document them.

## Scope
- All prior phases still work throughout.
- Pick features based on your current interests — this phase is intentionally flexible.
- Every non-trivial design decision gets a short note in `_docs/decisions/`.

## Deliverables
- **Messaging enhancements**: read receipts, message reactions, edit/delete
- **Search**: full-text message search within a conversation (Postgres first, Elasticsearch noted)
- **Groups**: create group conversations with a name + member invites
- **Notifications**: web push for new messages when the tab is in the background
- **Observability**: structured logs flowing to CloudWatch; basic alerting concept

## New concepts introduced this phase
| Concept | What you'll learn |
|---|---|
| **Read receipts** | Correctness under concurrent events (optimistic vs. authoritative) |
| **Soft deletes** | Why you rarely hard-delete user-generated content |
| **Full-text search** | Postgres `tsvector`, GIN indexes, trade-offs vs. Elasticsearch |
| **Web Push API** | Service workers, VAPID keys, push subscription lifecycle |
| **Observability** | Logs → metrics → alerts mental model |

---

## Section A — Messaging Enhancements

### A1) Read receipts

- [ ] Add a `MessageReceipt` model to the Prisma schema: `messageId`, `userId`, `readAt` — one row per user per message.
- [ ] When a user opens a conversation, emit a `mark_read` event with the latest visible `messageId`; server writes `readAt` to Postgres.
- [ ] Broadcast a `receipt_update` event to the conversation room so other clients update their UI.
- [ ] Render a minimal read receipt indicator: a check mark or avatar stack on the last message the other user has read.
- [ ] Write a test validating that receipts are idempotent (marking the same message read twice doesn't create duplicates).

**Examples**

```prisma
// (example) receipt model — one row per user per message
model MessageReceipt {
  messageId String
  userId    String
  readAt    DateTime @default(now())
  message   Message @relation(fields: [messageId], references: [id])
  user      User    @relation(fields: [userId], references: [id])

  @@id([messageId, userId])
  @@index([messageId])  // fast lookup of "who read this message"
}
```

```ts
// (example) mark_read event contract in packages/shared
// export type MarkReadPayload = { conversationId: string; lastReadMessageId: string };
// export type ReceiptUpdatePayload = { conversationId: string; userId: string; lastReadMessageId: string };
```

**Done checks**
- [ ] Open a conversation in tab A while tab B is watching — tab B sees the receipt indicator update.
- [ ] Calling `mark_read` twice for the same message doesn't create duplicate DB rows.

---

### A2) Message reactions

- [ ] Add a `MessageReaction` model: `messageId`, `userId`, `emoji` (e.g., "👍") — unique per user per message per emoji.
- [ ] Add REST endpoints (or socket events) to add/remove a reaction; service enforces membership.
- [ ] Implement optimistic UI: reaction count updates instantly; rolls back on server error.
- [ ] Render a reaction pill row under each message (emoji + count); clicking your own reaction removes it.
- [ ] Add a soft limit: max 10 distinct emoji types per message to keep the UI clean.

**Examples**

```ts
// (example) optimistic reaction toggle in the UI
// const handleReact = (messageId: string, emoji: string) => {
//   // 1. Optimistically update local state
//   dispatch(toggleReactionOptimistic({ messageId, emoji, userId: currentUser.id }));
//   // 2. Send to server
//   addReaction({ messageId, emoji })
//     .unwrap()
//     .catch(() => {
//       // 3. Rollback if server rejects
//       dispatch(rollbackReaction({ messageId, emoji, userId: currentUser.id }));
//     });
// };
```

**Done checks**
- [ ] Reacting in tab A is immediately reflected in tab B in real-time.
- [ ] Removing a reaction works; reaction count hits 0 → pill disappears.

---

### A3) Edit + delete messages

- [ ] Add an `editedAt` field and `deletedAt` field (soft delete) to the `Message` model. Never hard-delete user messages.
- [ ] Add a `PATCH /api/v1/messages/:id` endpoint (owner only — enforce in service layer).
- [ ] Add a `DELETE /api/v1/messages/:id` endpoint: sets `deletedAt`, does not remove the row.
- [ ] Broadcast `message_updated` and `message_deleted` socket events to the room.
- [ ] Render edited messages with an "(edited)" label; render deleted messages as "This message was deleted." (not blank).

**Examples**

```ts
// (example) soft delete — never hard delete user-generated content
// await prisma.message.update({
//   where: { id: messageId },
//   data: { deletedAt: new Date() },  // row stays, content effectively hidden
// });
// When querying messages: add `where: { deletedAt: null }` to exclude deleted ones
// OR return them with content replaced: if (msg.deletedAt) return { ...msg, content: null }
```

**Done checks**
- [ ] Only the message author sees edit/delete options.
- [ ] Deleted messages show placeholder text, not blank space.
- [ ] `deletedAt` row still exists in Postgres — data is preserved for audit purposes.

---

## Section B — Group Conversations

### B1) Create + manage group conversations

- [ ] Add a "New Group" button to the sidebar that opens a modal: group name + select members from your contact list.
- [ ] Add `POST /api/v1/conversations` endpoint (already exists for 1:1, extend it for `type: "group"`): create conversation + bulk-insert `ConversationMember` rows.
- [ ] Emit a `group_created` socket event to all invited members so their sidebar updates without a page refresh.
- [ ] Add a group detail panel (name, members list, leave group button).
- [ ] Add a `leave_conversation` endpoint: removes the member row; broadcast to the room.

**Examples**

```ts
// (example) create group conversation request body
// POST /api/v1/conversations
// {
//   type: "group",
//   name: "Design Team",
//   memberIds: ["user_a", "user_b", "user_c"]
// }
```

```ts
// (example) notify invited members in real-time
// After creating the conversation:
// for (const memberId of memberIds) {
//   const socketIds = await io.in(`user_${memberId}`).fetchSockets();
//   // If they're online, their sidebar will update immediately
//   io.to(`user_${memberId}`).emit(SOCKET_EVENTS.groupCreated, newConversation);
// }
// Note: each user also joins a personal room `user_${userId}` on connect for targeted notifications
```

**Done checks**
- [ ] Create a group → all invited members see it in their sidebar in real-time (if online).
- [ ] Leaving a group → member is removed from the room and can no longer receive messages.

---

## Section C — Message Search

### C1) Mini-lab: Postgres full-text search

> **Why**: Full-text search is a new concept. Spend half a day understanding how Postgres's `tsvector` and `GIN` index work before adding it to the messages table.

- [ ] In a throwaway Prisma migration, add a `tsvector` column to a test table. Populate it with `to_tsvector('english', content)`.
- [ ] Run a `plainto_tsquery` search and observe the results. Try a partial word — see that it doesn't match (full-text search works on whole stems, not substrings).
- [ ] Add a GIN index on the `tsvector` column and compare query speed with `EXPLAIN ANALYZE`.
- [ ] Write down in `_docs/decisions/search-notes.md`: how `tsvector` works, when you'd graduate to Elasticsearch.

**Examples**

```sql
-- (example) Postgres full-text search query
-- SELECT id, content, ts_rank(search_vector, query) AS rank
-- FROM messages
-- WHERE search_vector @@ plainto_tsquery('english', 'hello world')
--   AND conversation_id = $1
-- ORDER BY rank DESC, created_at DESC
-- LIMIT 20;
```

```prisma
// (example) adding a search vector field via Prisma raw migration
// (Prisma doesn't natively support tsvector — use a raw SQL migration)
// -- Add column
// ALTER TABLE "Message" ADD COLUMN search_vector tsvector;
// -- Create GIN index for fast search
// CREATE INDEX message_search_idx ON "Message" USING GIN(search_vector);
// -- Populate from content
// UPDATE "Message" SET search_vector = to_tsvector('english', content);
```

**Done checks**
- [ ] Search query returns correct messages; order is by relevance rank.
- [ ] Search with a typo returns no results (expected — document this limitation).

---

### C2) Search integration in the app

- [ ] Add a `GET /api/v1/conversations/:id/search?q=<query>` endpoint using Postgres full-text search.
- [ ] Scope results to conversations the requesting user is a member of (never return messages from other conversations).
- [ ] Add a search input to the conversation header that triggers the query with a debounce (300ms).
- [ ] Render search results as a separate overlay list with the matched text highlighted.
- [ ] Add "no results" and "enter at least 3 characters" empty states.

**Done checks**
- [ ] Searching for a word that exists in messages returns those messages.
- [ ] Results are scoped: searching in conversation A cannot return messages from conversation B.

---

## Section D — Web Push Notifications

### D1) Mini-lab: Service workers + Web Push API

> **Why**: Service workers and the Push API are new and have a confusing setup. Spend 1 day on a standalone demo before integrating it.

- [ ] Generate VAPID keys: `npx web-push generate-vapid-keys`. Add them to `.env`.
- [ ] Register a bare-bones service worker in the web app (`public/sw.js`) that listens for `push` events and calls `self.registration.showNotification(...)`.
- [ ] In the app, call `Notification.requestPermission()` and `PushManager.subscribe()` to get a push subscription object.
- [ ] Send the subscription object to a test endpoint on the API. Use `web-push` to send a test notification.
- [ ] Confirm the notification appears even when the browser tab is in the background.

**Examples**

```ts
// (example) subscribe the user to push notifications (browser-side)
// const registration = await navigator.serviceWorker.register("/sw.js");
// const subscription = await registration.pushManager.subscribe({
//   userVisibleOnly: true,
//   applicationServerKey: urlBase64ToUint8Array(VAPID_PUBLIC_KEY),
// });
// // Send subscription to API to store per user
// await api.savePushSubscription(subscription.toJSON());
```

```js
// public/sw.js — minimal push event handler
self.addEventListener("push", (event) => {
  const data = event.data?.json() ?? {};
  event.waitUntil(
    self.registration.showNotification(data.title ?? "New message", {
      body: data.body,
      icon: "/icon.png",
    })
  );
});
```

**Done checks**
- [ ] Send a push notification from the API → it appears in the OS notification tray with the tab in the background.
- [ ] You can explain: what VAPID keys are for, what a service worker is, why `userVisibleOnly: true` is required.

---

### D2) Push notifications integration

- [ ] Add a `PushSubscription` model to store the subscription object per user.
- [ ] On new message in a conversation: check if any members are offline (not in the Redis online set) → send a push notification to their subscription endpoints.
- [ ] Include the conversation name and a truncated message preview in the notification payload.
- [ ] Add a "disable notifications" toggle per conversation stored in user preferences.
- [ ] Handle subscription expiry: catch `410 Gone` from the push provider and delete the stale subscription from Postgres.

**Done checks**
- [ ] Close the browser tab → receive a new message → OS notification appears.
- [ ] Mute a conversation → no notification for that conversation.

---

## Section E — Observability (Ops mindset)

### E1) Structured logs + CloudWatch

- [ ] Confirm all API logs are JSON-structured (pino or winston) with at minimum: `level`, `requestId`, `userId` (when authenticated), `message`, `timestamp`.
- [ ] Add log groups in CloudWatch (auto-created by ECS) — search for a specific `requestId` end-to-end across log entries.
- [ ] Add a `duration_ms` field to all request logs so you can spot slow endpoints.
- [ ] Set up a basic CloudWatch alarm: alert if API 5xx errors exceed 5 in 5 minutes.
- [ ] Write a runbook in `_docs/decisions/observability-notes.md`: how to investigate a bug in production using logs.

**Examples**

```ts
// (example) structured log entry for a handled request
// logger.info({
//   requestId: req.requestId,
//   userId: req.user?.id,
//   method: req.method,
//   path: req.path,
//   statusCode: res.statusCode,
//   duration_ms: Date.now() - req.startTime,
// });
```

**Done checks**
- [ ] Pick a request from the UI, copy its `requestId` from the network tab → find the exact request + all sub-logs in CloudWatch.
- [ ] CloudWatch alarm exists and would fire if errors spiked.

---

## Exit criteria (demo script)

- [ ] All prior phases still work end-to-end in production.
- [ ] Send a message → sender sees delivery check; receiver reads it → receipt indicator updates in sender's tab.
- [ ] React to a message in tab A → reaction appears in tab B in real-time.
- [ ] Delete a message → it shows "This message was deleted." in both tabs.
- [ ] Create a group → all invited online members see it appear in their sidebar.
- [ ] Search for a word → relevant messages appear; results are scoped to the current conversation.
- [ ] Close the browser tab → receive a message → OS push notification appears.
- [ ] Find a request end-to-end in CloudWatch logs using its `requestId`.
