# Phase 01 — MVP (minimal usable messaging app)

## Goal
Ship a **usable** real-time chat experience: login, see conversations, send and receive messages live, and see who is online. Every feature maps directly to `_docs/user-flow.md`.

## Scope (what "usable" means)
- Users can log in via Google or GitHub OAuth and stay signed in.
- Users can see their conversations, open one, and load message history.
- Users can send and receive messages in real-time.
- Users can see who is online and typing.
- Messages persist in Postgres. Single API instance for now (scaling comes in Phase 04).

## Deliverables
- **Auth**: OAuth (Google + GitHub) → JWT issued in an HttpOnly cookie
- **Data model**: Users, Conversations, ConversationMembers, Messages (Prisma + migrations)
- **Messaging**: `join_room` → load history → send → persist → broadcast → receive
- **Presence**: Redis Set tracks online users; `lastSeenAt` written to Postgres on disconnect
- **State**: RTK Query for server state (conversations, history); Redux Slices for UI state (active conversation, modals)

## New technologies introduced this phase
| Technology | What you'll use it for | Your current level |
|---|---|---|
| **Prisma** | DB schema, migrations, typed queries | New |
| **JWT in HttpOnly cookie** | Stateless session after OAuth | New |
| **Passport.js + OAuth** | Google / GitHub login flow | New |
| **RTK Query** | Fetch + cache conversations and history | Familiar but new patterns |
| **Redis (Sets)** | Track online users; pub/sub later (Phase 04) | New |

## Learning approach
For each new technology: **mini-lab first (1–2 days) → integrate into the app**.
You don't master it. You learn just enough to solve the specific problem in front of you.

---

## Section A — Data Layer (Prisma)

### A1) Mini-lab: Prisma basics (before touching the real schema)

> **Why**: Prisma is new to you. Before writing the real schema, spend 1 day running through these steps on a throwaway table so that migrations, the Prisma client, and typed queries feel familiar.

- [ ] Install Prisma in `apps/api`: `npm install prisma @prisma/client` then `npx prisma init`.
- [ ] Write a simple `Post` model (title, body, createdAt) in `schema.prisma` and run your first migration.
- [ ] Create a `db.ts` file that exports a single shared `PrismaClient` instance — never instantiate it in a request handler.
- [ ] Write a function that creates a Post, fetches all Posts, and fetches a single Post by ID using the generated client.
- [ ] Run `npx prisma studio` and view your data in the browser UI.

**Examples**

```ts
// apps/api/src/db.ts — single shared instance (never create new PrismaClient() in a handler)
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const prisma =
  globalForPrisma.prisma ?? new PrismaClient({ log: ["query"] });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

```bash
# Run your first migration
npx prisma migrate dev --name add_post_table

# Open Prisma Studio (visual DB browser)
npx prisma studio
```

**Done checks**
- [ ] Migration ran without errors and you can see the table in Prisma Studio.
- [ ] You can explain: what is a migration file, why you only create one PrismaClient instance, what `npx prisma generate` does.

---

### A2) Real schema: Users, Conversations, Messages

- [ ] Add `User`, `Conversation`, `ConversationMember`, and `Message` models to `schema.prisma`.
- [ ] Add database indexes on `Message(conversationId, createdAt)` for fast history retrieval.
- [ ] Run `npx prisma migrate dev --name initial_schema` and verify the tables exist.
- [ ] Create repository files under `apps/api/src/modules/messages/message.repository.ts` and `conversations/conversation.repository.ts` — Prisma queries live here only, never in routes or services.
- [ ] Seed a couple of conversations and messages for local dev (`prisma/seed.ts`).

**Examples**

```prisma
// prisma/schema.prisma
model User {
  id             String    @id @default(cuid())
  email          String    @unique
  username       String
  avatarUrl      String?
  lastSeenAt     DateTime?
  createdAt      DateTime  @default(now())
  memberships    ConversationMember[]
  messages       Message[]
}

model Conversation {
  id        String    @id @default(cuid())
  type      String    // "private" | "group"
  createdAt DateTime  @default(now())
  members   ConversationMember[]
  messages  Message[]
}

model ConversationMember {
  userId         String
  conversationId String
  user           User         @relation(fields: [userId], references: [id])
  conversation   Conversation @relation(fields: [conversationId], references: [id])

  @@id([userId, conversationId])
}

model Message {
  id             String       @id @default(cuid())
  content        String
  senderId       String
  conversationId String
  createdAt      DateTime     @default(now())
  sender         User         @relation(fields: [senderId], references: [id])
  conversation   Conversation @relation(fields: [conversationId], references: [id])

  // Index for fast history queries (most important index in the whole schema)
  @@index([conversationId, createdAt])
}
```

```ts
// (example) message.repository.ts — Prisma queries only, no HTTP/socket concerns
// export async function getMessagesByConversation(conversationId: string, take = 25, cursor?: string) {
//   return prisma.message.findMany({
//     where: { conversationId },
//     include: { sender: { select: { id: true, username: true, avatarUrl: true } } },
//     orderBy: { createdAt: "desc" },
//     take,
//     ...(cursor ? { skip: 1, cursor: { id: cursor } } : {}),
//   });
// }
```

**Done checks**
- [ ] `npx prisma migrate dev` runs clean — no drift warnings.
- [ ] You can query messages for a conversation and the sender data is included in **one query** (no N+1).

---

## Section B — Auth (JWT + OAuth)

### B1) Mini-lab: JWT in an HttpOnly cookie (no OAuth yet)

> **Why**: Before adding OAuth, understand how JWT sessions work in a cookie vs. localStorage. This is a critical security concept that's easy to get wrong. Spend half a day here.

- [ ] Install `jsonwebtoken` and `cookie-parser` in `apps/api`.
- [ ] Write a `/lab/login` POST endpoint that accepts a fake `userId`, signs a JWT, and sets it as an HttpOnly, Secure, SameSite=Strict cookie.
- [ ] Write a `/lab/me` GET endpoint that reads the cookie, verifies the JWT, and returns the decoded payload.
- [ ] Open your browser DevTools → Application → Cookies — verify you **cannot** access the cookie via `document.cookie` in the console.
- [ ] Write down: why HttpOnly matters (XSS can't steal it), why SameSite matters (CSRF protection), in `_docs/decisions/auth-notes.md`.

**Examples**

```ts
// (example) signing a JWT and setting it as an HttpOnly cookie
// const token = jwt.sign({ userId }, process.env.JWT_SECRET!, { expiresIn: "15m" });
// res.cookie("access_token", token, {
//   httpOnly: true,     // JS cannot read this cookie — prevents XSS token theft
//   secure: process.env.NODE_ENV === "production",  // HTTPS only in prod
//   sameSite: "strict", // blocks CSRF: cookie won't be sent on cross-site requests
//   maxAge: 15 * 60 * 1000, // 15 minutes in ms
// });
```

```ts
// (example) reading + verifying the cookie on a protected route
// const token = req.cookies["access_token"];
// if (!token) return res.status(401).json({ code: "unauthorized", message: "No session." });
// const payload = jwt.verify(token, process.env.JWT_SECRET!) as { userId: string };
// req.user = { id: payload.userId };
```

**Done checks**
- [ ] `/lab/me` returns user data when the cookie is present and valid.
- [ ] Deleting the cookie → `/lab/me` returns 401.
- [ ] `document.cookie` in the browser console does **not** show the token.

---

### B2) Mini-lab: OAuth flow with Google only (before adding GitHub)

> **Why**: OAuth has a confusing multi-step redirect flow. Build it for Google alone first so the concept is clear before duplicating it for GitHub.

- [ ] Create a Google OAuth app in [Google Cloud Console](https://console.cloud.google.com/) and get `CLIENT_ID` + `CLIENT_SECRET`. Add them to `.env`.
- [ ] Install `passport` and `passport-google-oauth20` in `apps/api`.
- [ ] Register the `GoogleStrategy` with the verify callback: look up or create the user in Postgres via the repository, then call `cb(null, user)`.
- [ ] Add two routes: `GET /auth/google` (starts the flow) and `GET /auth/google/callback` (handles the redirect).
- [ ] In the callback success handler, issue a JWT cookie (from B1) and redirect the user to the frontend.

**Examples**

```ts
// (example) Google strategy registration
// passport.use(new GoogleStrategy({
//   clientID: process.env.GOOGLE_CLIENT_ID!,
//   clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
//   callbackURL: `${process.env.API_URL}/auth/google/callback`,
//   scope: ["profile", "email"],
// }, async (accessToken, refreshToken, profile, cb) => {
//   try {
//     // upsert: find user by Google ID or create them
//     const user = await userRepository.upsertFromOAuth({
//       provider: "google",
//       providerId: profile.id,
//       email: profile.emails?.[0].value ?? "",
//       username: profile.displayName,
//       avatarUrl: profile.photos?.[0].value,
//     });
//     return cb(null, user);
//   } catch (err) {
//     return cb(err as Error);
//   }
// }));
```

```ts
// (example) callback route — issue JWT cookie then redirect
// app.get("/auth/google/callback",
//   passport.authenticate("google", { session: false, failureRedirect: "/login" }),
//   (req, res) => {
//     const user = req.user as User;
//     const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET!, { expiresIn: "15m" });
//     res.cookie("access_token", token, { httpOnly: true, secure: isProduction, sameSite: "strict" });
//     res.redirect(`${process.env.WEB_ORIGIN}/`);
//   }
// );
```

**Done checks**
- [ ] Clicking "Login with Google" in the browser redirects to Google, authenticates, and lands back on the app with a cookie set.
- [ ] A new user record is created in Postgres on first login; subsequent logins reuse the existing record.

---

### B3) Auth integration: protect routes + socket handshake

- [ ] Extract JWT verification into a reusable `authMiddleware` under `apps/api/src/middleware/auth.middleware.ts`.
- [ ] Add GitHub OAuth strategy following the same pattern as Google (second provider, same flow).
- [ ] Protect all `/api/v1` routes with `authMiddleware` — unauthenticated requests get a 401.
- [ ] In the Socket.IO handshake middleware, read the `access_token` cookie and verify the JWT — reject connections with no valid token.
- [ ] Add a frontend login page with "Login with Google" and "Login with GitHub" buttons.

**Examples**

```ts
// (example) authMiddleware — reuse this on every protected route
// export function authMiddleware(req: Request, res: Response, next: NextFunction) {
//   const token = req.cookies["access_token"];
//   if (!token) return res.status(401).json({ code: "unauthorized", message: "No session." });
//   try {
//     const payload = jwt.verify(token, process.env.JWT_SECRET!) as { userId: string };
//     req.user = { id: payload.userId };
//     next();
//   } catch {
//     res.status(401).json({ code: "unauthorized", message: "Session expired." });
//   }
// }
```

```ts
// (example) Socket.IO handshake auth — runs before "connection" event fires
// io.use((socket, next) => {
//   const token = socket.handshake.headers.cookie
//     ? parseCookies(socket.handshake.headers.cookie)["access_token"]
//     : null;
//   if (!token) return next(new Error("unauthorized"));
//   try {
//     const payload = jwt.verify(token, process.env.JWT_SECRET!) as { userId: string };
//     socket.data.userId = payload.userId;
//     next();
//   } catch {
//     next(new Error("unauthorized"));
//   }
// });
```

**Done checks**
- [ ] Calling a protected REST endpoint without a cookie → `401 unauthorized`.
- [ ] Connecting to the socket without a cookie → connection rejected.
- [ ] Both Google and GitHub login flows work end-to-end.

---

## Section C — State Management (RTK Query)

### C1) Mini-lab: RTK Query basics (one tiny API slice)

> **Why**: RTK Query handles server state differently than regular Redux. Before wiring it to real endpoints, spend half a day on a tiny experiment so the `createApi → store → hook` pattern clicks.

- [ ] Install `@reduxjs/toolkit` and `react-redux` in `apps/web`.
- [ ] Create a single `api.ts` file using `createApi` with one GET endpoint (e.g., fetch the health check).
- [ ] Add the API's `reducer` and `middleware` to the Redux `store`.
- [ ] Call `useGetHealthQuery()` in a component and render `isLoading`, `isError`, and `data` states.
- [ ] Invalidate the cache manually and watch RTK Query re-fetch automatically.

**Examples**

```ts
// apps/web/src/state/api.ts — minimal RTK Query slice
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";

export const api = createApi({
  reducerPath: "api",
  baseQuery: fetchBaseQuery({ baseUrl: "/api/v1", credentials: "include" }), // credentials needed for cookies
  tagTypes: ["Conversation", "Message"],
  endpoints: () => ({}), // endpoints injected per feature
});
```

```ts
// apps/web/src/state/store.ts
import { configureStore } from "@reduxjs/toolkit";
import { setupListeners } from "@reduxjs/toolkit/query";
import { api } from "./api";

export const store = configureStore({
  reducer: { [api.reducerPath]: api.reducer },
  middleware: (getDefault) => getDefault().concat(api.middleware),
});

setupListeners(store.dispatch); // enables automatic re-fetch on focus/reconnect
```

**Done checks**
- [ ] Component renders loading → data states correctly from a real API call.
- [ ] You can explain: `providesTags`, `invalidatesTags`, why `credentials: "include"` is needed.

---

### C2) Conversation list (RTK Query + sidebar)

- [ ] Add a `GET /api/v1/conversations` endpoint that returns conversations for the authenticated user (with last message preview).
- [ ] Create a `conversationsApi` endpoint (injected into the base `api`) and call `useGetConversationsQuery()` in the sidebar.
- [ ] Render conversation list items with participant names, last message preview, and a presence indicator placeholder.
- [ ] Add empty state ("No conversations yet") and loading skeleton.
- [ ] Implement "select conversation" — store the active `conversationId` in a Redux UI slice (not in RTK Query cache).

**Examples**

```ts
// (example) inject conversations endpoint into the base api
// export const conversationsApi = api.injectEndpoints({
//   endpoints: (build) => ({
//     getConversations: build.query<Conversation[], void>({
//       query: () => "conversations",
//       providesTags: ["Conversation"],
//     }),
//   }),
// });
// export const { useGetConversationsQuery } = conversationsApi;
```

```ts
// (example) UI slice for active conversation — local state, not server state
// const uiSlice = createSlice({
//   name: "ui",
//   initialState: { activeConversationId: null as string | null },
//   reducers: {
//     setActiveConversation: (state, action) => {
//       state.activeConversationId = action.payload;
//     },
//   },
// });
```

**Done checks**
- [ ] Conversations load from the API and render in the sidebar.
- [ ] Selecting a conversation updates the Redux UI state without triggering a refetch.

---

## Section D — Realtime Messaging (Socket.IO)

> You already understand Socket.IO rooms from the Phase 00 mini-lab. This section integrates messaging into the real app.

### D1) Room join + message history (cursor pagination)

- [ ] On conversation open, emit `join_room` with the `conversationId`; server calls `socket.join("room_${conversationId}")`.
- [ ] Add a `GET /api/v1/conversations/:id/messages` endpoint with cursor-based pagination (`?cursor=<messageId>&take=25`).
- [ ] Wire the endpoint to an RTK Query endpoint and call it on conversation open to load the latest 25 messages.
- [ ] On scroll to the top of the message list, fetch the next page using the oldest visible message's `id` as the cursor.
- [ ] Render messages in chronological order; group consecutive messages from the same sender.

**Examples**

```ts
// (example) cursor pagination query in the repository
// return prisma.message.findMany({
//   where: { conversationId },
//   include: { sender: { select: { id: true, username: true, avatarUrl: true } } },
//   orderBy: { createdAt: "desc" },
//   take: 25,
//   ...(cursor ? { skip: 1, cursor: { id: cursor } } : {}),
// });
```

```ts
// (example) RTK Query endpoint for paginated history
// getMessages: build.query<Message[], { conversationId: string; cursor?: string }>({
//   query: ({ conversationId, cursor }) =>
//     `conversations/${conversationId}/messages${cursor ? `?cursor=${cursor}` : ""}`,
//   providesTags: (_result, _error, { conversationId }) => [{ type: "Message", id: conversationId }],
// }),
```

**Done checks**
- [ ] Opening a conversation loads the last 25 messages.
- [ ] Scrolling to the top loads the next 25 — without losing scroll position.
- [ ] Message sender data is loaded in **one query** per batch (no N+1).

---

### D2) Send message (optimistic UI + persistence + broadcast)

- [ ] Add a message composer form with an accessible `<textarea>` + submit `<button>`.
- [ ] On submit, **optimistically** add the message to the UI with a `sending` status before waiting for the server.
- [ ] Emit a `message` socket event to the server; server validates membership → saves to Postgres → broadcasts to `room_${conversationId}`.
- [ ] On receiving a `message` event from the socket, append it to the conversation view (skip if it's your own optimistic message).
- [ ] If the server emits a `message_error` event back, update the optimistic message's status to `failed` and show a "Retry" button.

**Examples**

```ts
// (example) send message event shape — type this in packages/shared
// export type SendMessagePayload = { conversationId: string; content: string; tempId: string };
// export type MessagePayload = { id: string; conversationId: string; content: string; sender: UserPreview; createdAt: string };
```

```ts
// (example) server handler — validate → persist → broadcast
// socket.on(SOCKET_EVENTS.message, async (payload: SendMessagePayload) => {
//   const isMember = await conversationService.isMember(payload.conversationId, socket.data.userId);
//   if (!isMember) return socket.emit("message_error", { tempId: payload.tempId, code: "forbidden" });
//
//   const message = await messageService.create({ ...payload, senderId: socket.data.userId });
//   io.to(`room_${payload.conversationId}`).emit(SOCKET_EVENTS.message, message);
// });
```

**Done checks**
- [ ] Sending a message shows it instantly (optimistic), then confirms once saved.
- [ ] Opening the same conversation in a second browser tab — the message arrives in real-time.
- [ ] Deliberately causing a server error → message shows "failed" state with a Retry button.

---

## Section E — Presence (Redis Sets)

### E1) Mini-lab: Redis Sets for presence (before integrating)

> **Why**: Redis is completely new. Spend half a day running these commands directly so SADD/SREM/SMEMBERS feel familiar before you wire them into the socket handlers.

- [ ] Install `ioredis` in `apps/api` and connect to your local Redis (`redis://localhost:6379`).
- [ ] In a throwaway script, run `SADD online_users user1`, `SADD online_users user2`, then `SMEMBERS online_users`.
- [ ] Run `SREM online_users user1` and confirm it's removed with `SMEMBERS`.
- [ ] Try `SET session:user1 "active" EX 30` and watch it expire after 30 seconds with `GET`.
- [ ] Write down: why Redis is used here instead of Postgres, what "volatile" means, in `_docs/decisions/redis-notes.md`.

**Examples**

```ts
// (example) ioredis connection — reuse one instance per purpose
// import { Redis } from "ioredis";
// export const redis = new Redis(process.env.REDIS_URL!);
```

```bash
# Run these directly in redis-cli to understand the commands before coding
redis-cli
> SADD online_users "user_abc"     # add to set → returns 1 (added)
> SADD online_users "user_xyz"     # returns 1
> SMEMBERS online_users            # ["user_abc", "user_xyz"]
> SREM online_users "user_abc"     # remove from set → returns 1
> SMEMBERS online_users            # ["user_xyz"]
> SET session:test "hello" EX 10   # set with 10 second TTL
> TTL session:test                 # shows remaining seconds
```

**Done checks**
- [ ] You can add/remove/list members of a Redis set from Node.js code.
- [ ] You can explain: why Redis and not Postgres for online status, what happens to Redis data if it restarts.

---

### E2) Presence integration: online status + last seen

- [ ] On socket `connection`: `SADD online_users ${userId}` and broadcast a `presence_update` event to all connected clients.
- [ ] On socket `disconnect`: `SREM online_users ${userId}`, write `lastSeenAt` to Postgres, and broadcast a final `presence_update`.
- [ ] Add a `GET /api/v1/users/online` endpoint that returns the current online set via `SMEMBERS online_users`.
- [ ] Render a green presence dot next to online users in the conversation list and a "last seen" timestamp for offline users.
- [ ] Handle reconnection: if the same user connects twice (e.g., two tabs), `SADD` is idempotent — it won't double-add.

**Examples**

```ts
// (example) presence handlers in the socket connection event
// io.on("connection", async (socket) => {
//   const { userId } = socket.data;
//   await redis.sadd("online_users", userId);
//   io.emit(SOCKET_EVENTS.presenceUpdate, { userId, status: "online" });
//
//   socket.on("disconnect", async () => {
//     await redis.srem("online_users", userId);
//     await userRepository.updateLastSeen(userId); // write to Postgres
//     io.emit(SOCKET_EVENTS.presenceUpdate, { userId, status: "offline" });
//   });
// });
```

**Done checks**
- [ ] Opening the app shows your own status as online in another tab.
- [ ] Closing the tab → presence indicator updates to offline within a few seconds.
- [ ] `SMEMBERS online_users` in `redis-cli` matches who is actually connected.

---

## Section F — Typing Indicators

### F1) Typing indicators (debounced events)

- [ ] In the message composer, emit a `typing` event when the user types — but **debounce it** so it only fires once every 500ms, not on every keystroke.
- [ ] Server broadcasts `typing` to the room, **excluding** the sender (`socket.to(room).emit(...)`).
- [ ] Render "X is typing…" below the message list when a typing event is received.
- [ ] Clear the typing indicator after 2 seconds of inactivity (set a timeout that resets on each typing event).
- [ ] Ensure typing state resets when the user sends a message or switches conversations.

**Examples**

```ts
// (example) typing event payload type — define in packages/shared
// export type TypingPayload = { conversationId: string; userId: string; username: string };
```

```ts
// (example) debounce in the composer — fire the event at most once per 500ms
// const emitTyping = useMemo(
//   () => debounce(() => socket.emit(SOCKET_EVENTS.typing, { conversationId, userId, username }), 500),
//   [conversationId]
// );
// <textarea onChange={(e) => { setValue(e.target.value); emitTyping(); }} />
```

```ts
// (example) server: broadcast to room excluding sender
// socket.on(SOCKET_EVENTS.typing, (payload: TypingPayload) => {
//   socket.to(`room_${payload.conversationId}`).emit(SOCKET_EVENTS.typing, payload);
// });
```

**Done checks**
- [ ] Typing in one tab shows "X is typing…" in a second tab — not on your own screen.
- [ ] The indicator disappears 2 seconds after you stop typing.
- [ ] Sending a message immediately clears the typing indicator.

---

## Exit criteria (demo script)

- [ ] Start infra + apps.
- [ ] Log in via Google → redirected to dashboard with a cookie set.
- [ ] See online presence indicator for your own account (open a second tab).
- [ ] Open a conversation → last 25 messages load → scroll up to load older ones.
- [ ] Send a message → optimistic bubble appears → arrives in the second tab in real-time.
- [ ] Type in one tab → "X is typing…" appears in the other tab and disappears after 2 seconds.
- [ ] Close one tab → presence updates to offline within a few seconds.
- [ ] Check `redis-cli SMEMBERS online_users` → only the remaining connected user is listed.
