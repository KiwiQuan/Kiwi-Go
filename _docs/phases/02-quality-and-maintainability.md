# Phase 02 — Quality & Maintainability (senior defaults)

## Goal
Turn the MVP into something you can **safely change** and confidently hand to another developer. No new product features — just the quality layer that separates junior code from senior code: contracts, validation, tests, error resilience, and accessibility.

## Scope
- MVP still works end-to-end throughout this whole phase.
- Every external input (API body, socket payload, env vars) is validated at the boundary.
- Core business logic has unit tests. Key routes have integration tests.
- Errors are consistent, observable, and never leak internals to clients.
- The codebase matches `project-rules.md` with zero structural drift.

## Deliverables
- **Zod** validation on all inbound API and socket payloads + env vars
- **Shared contracts** (DTOs + socket event types) in `packages/shared`
- **Vitest** unit tests for service functions + integration tests for key routes
- **Error resilience** in the UI: loading/empty/error states everywhere
- **Accessibility** audit: keyboard navigation + focus management working end-to-end

## New technologies introduced this phase
| Technology | What you'll use it for | Your current level |
|---|---|---|
| **Zod** | Runtime schema validation at API/socket boundaries | New |
| **Vitest** | Unit + integration testing for Node.js services | New |
| **Supertest** | HTTP integration tests against your Express app | New |

## Learning approach
Mini-lab each new tool (Zod, Vitest) for half a day before wiring it into the app.

---

## Section A — Validation with Zod

### A1) Mini-lab: Zod basics (before adding it to the API)

> **Why**: Zod is new. Spend half a day on these basics so `z.object`, `safeParse`, and `z.infer` feel natural before you use it in real request handlers.

- [ ] Install Zod: `npm install zod`.
- [ ] Write a schema for a `Message` object with `content` (non-empty string), `conversationId` (cuid string), and `createdAt` (optional date).
- [ ] Use `.parse()` to validate a valid object — see what the typed return looks like.
- [ ] Use `.safeParse()` on an invalid object — inspect the `error.issues` array to understand what the error output looks like.
- [ ] Use `z.infer<typeof MessageSchema>` to extract a TypeScript type from the schema — print it and confirm it matches what you'd write by hand.

**Examples**

```ts
import { z } from "zod";

// Define a schema
const MessageSchema = z.object({
  content: z.string().min(1, "Message cannot be empty"),
  conversationId: z.string().cuid(),
  createdAt: z.date().optional(),
});

// Extract a TypeScript type from the schema — free type safety
type Message = z.infer<typeof MessageSchema>;

// .parse() throws on failure
MessageSchema.parse({ content: "hello", conversationId: "cjld2cyuq0000..." });

// .safeParse() returns { success, data } or { success: false, error }
const result = MessageSchema.safeParse({ content: "", conversationId: "bad" });
if (!result.success) {
  console.log(result.error.issues); // array of { path, message } — very readable
}
```

**Done checks**
- [ ] You can explain: `parse` vs `safeParse`, what `z.infer` gives you, and why you put schemas in `packages/shared`.

---

### A2) Validate all API request bodies

- [ ] Move `MessageSchema`, `SendMessageSchema`, and `CreateConversationSchema` into `packages/shared/src/schemas/`.
- [ ] Write a reusable `validate(schema)` middleware factory for Express that calls `safeParse` and returns `400` with structured errors on failure.
- [ ] Apply the validate middleware to the message send route, the conversation creation route, and any other mutating routes.
- [ ] Confirm that sending a bad request body returns a predictable `{ code: "validation_error", issues: [...] }` response shape.
- [ ] Confirm that valid requests still work end-to-end.

**Examples**

```ts
// apps/api/src/middleware/validate.middleware.ts
// import { ZodSchema } from "zod";
// export function validate(schema: ZodSchema) {
//   return (req: Request, res: Response, next: NextFunction) => {
//     const result = schema.safeParse(req.body);
//     if (!result.success) {
//       return res.status(400).json({
//         code: "validation_error",
//         issues: result.error.issues.map(i => ({ path: i.path.join("."), message: i.message })),
//         requestId: req.requestId,
//       });
//     }
//     req.body = result.data; // replace body with the safe, typed, parsed value
//     next();
//   };
// }
```

**Done checks**
- [ ] POST to a message endpoint with `content: ""` → gets `400 validation_error`.
- [ ] POST with valid data → passes through to the handler unchanged.

---

### A3) Validate socket event payloads

- [ ] Define Zod schemas for `join_room`, `message`, and `typing` payloads in `packages/shared`.
- [ ] In each socket event handler, run `safeParse` on the incoming payload before using it.
- [ ] If validation fails, emit a scoped error event back to the sender (e.g., `socket_error`) and return early.
- [ ] Never trust `socket.data.userId` without confirming it was set by the auth middleware first.

**Examples**

```ts
// (example) validate inside a socket event handler
// socket.on(SOCKET_EVENTS.message, (rawPayload) => {
//   const result = SendMessageSchema.safeParse(rawPayload);
//   if (!result.success) {
//     socket.emit("socket_error", { code: "validation_error", issues: result.error.issues });
//     return;
//   }
//   const payload = result.data; // now typed and safe
//   messageService.handleSend(socket.data.userId, payload);
// });
```

**Done checks**
- [ ] Emitting a malformed socket event → server logs a warning and emits `socket_error` back, does not crash.

---

### A4) Validate env vars at startup with Zod

- [ ] Replace the manual env check loop from Phase 00 with a Zod schema for all required env vars.
- [ ] Use `z.object({ DATABASE_URL: z.string().url(), REDIS_URL: z.string(), ... })` and call `.parse(process.env)`.
- [ ] Export the typed `env` object and import it everywhere — no more `process.env.WHATEVER` scattered around.
- [ ] Confirm that missing or malformed env vars crash the app with a Zod error listing exactly which keys are wrong.

**Examples**

```ts
// apps/api/src/config/env.ts
import { z } from "zod";

const EnvSchema = z.object({
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().min(1),
  JWT_SECRET: z.string().min(32),
  WEB_ORIGIN: z.string().url(),
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().default(4000),
});

export const env = EnvSchema.parse(process.env);
// If any key is missing or wrong, this throws a clear Zod error at startup.
// Import `env` instead of using process.env anywhere else in the app.
```

**Done checks**
- [ ] Remove `JWT_SECRET` from `.env` and restart — server crashes with a clear message naming the missing key.
- [ ] All uses of `process.env` in the API are replaced with the typed `env` import.

---

## Section B — Testing

### B1) Mini-lab: Vitest basics (before testing the real app)

> **Why**: Vitest is new. Spend half a day writing tests for a trivial pure function so the `describe/it/expect/vi.mock` pattern is familiar before testing services with real dependencies.

- [ ] Install Vitest: `npm install -D vitest`.
- [ ] Add a `vitest.config.ts` and a test script (`"test": "vitest run"`) to `apps/api`.
- [ ] Write a pure function (e.g., `formatLastSeen(date: Date): string`) and write 3 tests: valid date, null input, future date.
- [ ] Write a test that uses `vi.mock` to mock a dependency — mock the repository and test the service independently.
- [ ] Run `vitest --watch` and make a test fail on purpose — observe the output.

**Examples**

```ts
// (example) testing a pure service function
import { describe, it, expect, vi } from "vitest";
import { messageService } from "../modules/messages/message.service";
import * as messageRepo from "../modules/messages/message.repository";

// Mock the repository so tests don't need a real database
vi.mock("../modules/messages/message.repository");

describe("messageService.create", () => {
  it("throws if user is not a conversation member", async () => {
    vi.mocked(messageRepo.isMember).mockResolvedValue(false);

    await expect(
      messageService.create({ conversationId: "c1", senderId: "u1", content: "hi" })
    ).rejects.toThrow("forbidden");
  });
});
```

**Done checks**
- [ ] `npm test` runs and at least 1 test passes, 1 fails on purpose — you fix it.
- [ ] You can explain: what `vi.mock` does and why you mock the repository (not the service).

---

### B2) Unit tests for service functions

- [ ] Write unit tests for `messageService.create`: membership check fails → throws; valid → calls repo + returns message.
- [ ] Write unit tests for `conversationService.getForUser`: returns only conversations the user belongs to.
- [ ] Write unit tests for the `validate` middleware: valid body passes through; invalid body returns 400 with issues.
- [ ] Each test file lives next to the file it tests (e.g., `message.service.test.ts` beside `message.service.ts`).
- [ ] All tests run with `npm test` and are deterministic (no random or time-dependent assertions without mocks).

**Done checks**
- [ ] `npm test` is green across all service test files.
- [ ] Tests run in < 5 seconds (they're mocked — no real DB calls).

---

### B3) Integration tests for key routes

- [ ] Install Supertest: `npm install -D supertest @types/supertest`.
- [ ] Write an integration test for `POST /api/v1/messages` (auth required): no cookie → 401; bad body → 400; valid → 201 with message shape.
- [ ] Write an integration test for `GET /api/v1/conversations`: no cookie → 401; valid session → 200 with array.
- [ ] Set up a test Postgres database (use a separate `TEST_DATABASE_URL` and run `prisma migrate deploy` in test setup).
- [ ] Add a `beforeEach` that clears relevant tables and seeds minimal data so tests are isolated.

**Examples**

```ts
// (example) integration test with Supertest
// import request from "supertest";
// import { app } from "../app";
//
// describe("POST /api/v1/messages", () => {
//   it("returns 401 without auth cookie", async () => {
//     const res = await request(app).post("/api/v1/messages").send({ content: "hi" });
//     expect(res.status).toBe(401);
//   });
//
//   it("returns 400 with empty content", async () => {
//     const res = await request(app)
//       .post("/api/v1/messages")
//       .set("Cookie", `access_token=${validTestToken}`)
//       .send({ content: "", conversationId: testConversationId });
//     expect(res.status).toBe(400);
//     expect(res.body.code).toBe("validation_error");
//   });
// });
```

**Done checks**
- [ ] Integration tests run against a real test database and pass.
- [ ] Breaking a route (e.g., removing the auth middleware) causes the relevant test to fail.

---

## Section C — Error Resilience (UI)

### C1) Standardize API error shape and UI error states

- [ ] Confirm every API error response uses the same shape: `{ code, message, requestId }` — fix any endpoints that don't.
- [ ] Add a single `handleApiError` utility in `apps/web/src/lib/` that maps error codes to user-facing messages.
- [ ] Add loading skeletons to the conversation list and message history (render placeholders while `isLoading` is true).
- [ ] Add error banners for `isError` states in both the conversation list and message history.
- [ ] Add an "optimistic message failed" state: if the server rejects a message, show a red "Failed — Retry" badge on that message.

**Examples**

```ts
// (example) handleApiError utility
// export function handleApiError(error: unknown): string {
//   if (isFetchBaseQueryError(error)) {
//     const data = error.data as { code: string; message: string };
//     const messages: Record<string, string> = {
//       unauthorized: "Your session expired. Please log in again.",
//       forbidden: "You don't have permission to do that.",
//       validation_error: "Something was wrong with your input.",
//     };
//     return messages[data?.code] ?? "Something went wrong. Please try again.";
//   }
//   return "An unexpected error occurred.";
// }
```

**Done checks**
- [ ] Turn off the API and reload the app — every data-dependent section shows a non-crashing error state.
- [ ] Sending a message that fails shows a red badge, not a silent disappearance.

---

## Section D — Accessibility Audit

### D1) Keyboard navigation + focus management

- [ ] Tab through the entire app with a mouse unplugged — every interactive element must be reachable and visibly focused.
- [ ] After opening a conversation, focus should move to the message composer automatically (`useEffect` + `ref.current.focus()`).
- [ ] Modals/drawers must trap focus while open and restore focus to the trigger element when closed.
- [ ] All icon-only buttons must have an `aria-label` (e.g., the send button, close button).
- [ ] Run Lighthouse Accessibility audit — fix any issues scoring below 90.

**Examples**

```tsx
// (example) auto-focus composer when conversation opens
// const composerRef = useRef<HTMLTextAreaElement>(null);
// useEffect(() => {
//   composerRef.current?.focus();
// }, [activeConversationId]); // re-focus each time conversation changes
```

```html
<!-- (example) icon-only button with accessible name -->
<!-- <button type="button" aria-label="Send message">
  <SendIcon aria-hidden="true" />
</button> -->
```

**Done checks**
- [ ] Entire main flow (login → select conversation → read messages → send message) is completable via keyboard only.
- [ ] Lighthouse Accessibility score ≥ 90.

---

## Exit criteria (demo script)

- [ ] MVP still works end-to-end.
- [ ] Send a POST with a bad body → receive `{ code: "validation_error", issues: [...] }`.
- [ ] Send a socket event with a missing field → server rejects it and emits `socket_error` back.
- [ ] Remove a required env var → server refuses to start with a clear Zod error.
- [ ] `npm test` → all green.
- [ ] Tab through the whole app → all interactive elements reachable and focused.
