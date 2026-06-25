# Phase 03 — Media + Security Hardening

## Goal
Add **file and image sharing** while leveling up the security of the whole app: hardened auth (refresh tokens + revocation), S3 uploads behind signed URLs, rate limiting, and locked-down defaults. This is where you start thinking like a security-conscious senior developer.

## Scope
- Users can attach and share images/files in chat.
- Files are never public — only conversation members can access them via short-lived signed URLs.
- JWT sessions are hardened: short-lived access tokens + revocable refresh tokens.
- Abuse controls exist: rate limiting on auth and message endpoints.

## Deliverables
- **AWS S3**: presigned PUT for upload + presigned GET for access (private bucket)
- **Refresh token flow**: issued on login, stored in HttpOnly cookie, revocable via Redis blocklist
- **Rate limiting**: `express-rate-limit` on auth + message routes
- **Authorization guards**: membership checks enforced on all media and message actions

## New technologies introduced this phase
| Technology | What you'll use it for | Your current level |
|---|---|---|
| **AWS S3 + SDK v3** | Upload files; serve via signed URLs | New |
| **AWS IAM** | Least-privilege permissions for your API to access S3 | New |
| **express-rate-limit** | Throttle auth + message endpoints | New |
| **Refresh tokens** | Extend sessions securely without long-lived access tokens | Concept is new |

## Learning approach
Mini-lab S3 uploads in isolation (upload a file, retrieve it) before wiring it into the messaging flow.

---

## Section A — AWS S3 File Uploads

### A1) Mini-lab: S3 presigned URLs (before touching the real app)

> **Why**: AWS S3 has a security model that's easy to get wrong (public buckets, wrong permissions). Spend 1–2 days here learning the presigned URL pattern before wiring it into the chat.

- [ ] Create an AWS account (free tier) and create an S3 bucket. In **Block Public Access settings**, turn on all four blocks — the bucket must stay private.
- [ ] Create an IAM user with a minimal policy: only `s3:PutObject` and `s3:GetObject` on your specific bucket. Save the `ACCESS_KEY_ID` and `SECRET_ACCESS_KEY` to `.env`.
- [ ] Install `@aws-sdk/client-s3` and `@aws-sdk/s3-request-presigner` in `apps/api`.
- [ ] Write a throwaway script that generates a presigned `PutObject` URL (5 min expiry) and uses it to upload a text file via `fetch`.
- [ ] Write a second script that generates a presigned `GetObject` URL (1 hr expiry) and opens it in the browser — confirm the file is readable.

**Examples**

```ts
import { S3Client, PutObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const s3 = new S3Client({ region: process.env.AWS_REGION });

// Generate a presigned URL for UPLOADING (client puts the file directly to S3)
const uploadUrl = await getSignedUrl(
  s3,
  new PutObjectCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: `uploads/${userId}/${crypto.randomUUID()}.jpg`,
    ContentType: "image/jpeg",
  }),
  { expiresIn: 300 } // 5 minutes — client must upload within this window
);

// Generate a presigned URL for READING (short-lived access, no public URL)
const readUrl = await getSignedUrl(
  s3,
  new GetObjectCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: objectKey,
  }),
  { expiresIn: 3600 } // 1 hour
);
```

```bash
# Verify the bucket is private: this should return Access Denied
curl https://your-bucket.s3.amazonaws.com/uploads/some-file.jpg
```

**Done checks**
- [ ] You uploaded a file to S3 from a Node.js script using a presigned PUT URL.
- [ ] You can read it back using a presigned GET URL — but the direct S3 URL returns Access Denied.
- [ ] You can explain: why presigned URLs (not public files), what an IAM policy does, what `expiresIn` means.

---

### A2) Upload endpoint in the API

- [ ] Add `POST /api/v1/uploads/presign` (auth required): validate file metadata (`contentType`, `filename`, `sizeBytes`), then return a presigned PUT URL and the final S3 key.
- [ ] Enforce file constraints server-side: allowed MIME types (`image/jpeg`, `image/png`, `video/mp4`, etc.) and max size (e.g., 10MB).
- [ ] Store an `Upload` record in Postgres (`id`, `key`, `conversationId`, `uploadedBy`, `mimeType`, `status: "pending"`).
- [ ] After the client uploads to S3, the client calls `POST /api/v1/uploads/confirm` with the key — update the `Upload` record status to `"complete"` and emit a message with the media URL.
- [ ] Add a cleanup job concept (documented note — no real cron needed yet): uploads stuck as `pending` after 30 min should be considered failed.

**Examples**

```ts
// (example) presign endpoint response shape
// {
//   uploadUrl: "https://s3.amazonaws.com/...?X-Amz-Signature=...",  // PUT this
//   key: "uploads/userId/uuid.jpg",                                  // store and send back on confirm
//   expiresAt: "2026-06-25T14:00:00Z"
// }
```

```ts
// (example) allowed types + size validation with Zod
// const UploadRequestSchema = z.object({
//   contentType: z.enum(["image/jpeg", "image/png", "image/gif", "video/mp4"]),
//   filename: z.string().max(200),
//   sizeBytes: z.number().int().max(10 * 1024 * 1024), // 10MB limit
//   conversationId: z.string().cuid(),
// });
```

**Done checks**
- [ ] Requesting a presign for `application/exe` → rejected with 400.
- [ ] Requesting for `image/jpeg` within size limit → returns a valid presigned URL.
- [ ] Confirming the upload → message is emitted with the media URL.

---

### A3) Media messages in the UI

- [ ] Add a paperclip `<button aria-label="Attach file">` to the composer that opens a file input.
- [ ] On file select: call `/uploads/presign`, upload the file directly from the browser to the S3 URL using `fetch`, then call `/uploads/confirm`.
- [ ] Show an upload progress indicator while the file is being uploaded to S3.
- [ ] Render received media messages: image types → `<img>` with `alt`; other types → a styled download link.
- [ ] Fetch fresh presigned GET URLs when rendering media (request from your API, not storing raw S3 URLs long-term).

**Examples**

```ts
// (example) browser-side upload to S3 using the presigned PUT URL
// const uploadToS3 = async (presignedUrl: string, file: File) => {
//   const res = await fetch(presignedUrl, {
//     method: "PUT",
//     body: file,
//     headers: { "Content-Type": file.type },
//   });
//   if (!res.ok) throw new Error("Upload to S3 failed");
// };
```

**Done checks**
- [ ] Select an image → it appears in the chat for both users in real-time.
- [ ] Attempting to load the S3 URL directly in a new tab (without the signature) → Access Denied.

---

## Section B — Hardened Auth (Refresh Tokens)

### B1) Mini-lab: Understand access vs. refresh tokens

> **Why**: Refresh tokens are a new concept. Spend half a day understanding the problem they solve before implementing them, or you'll implement them wrong.

- [ ] Read about the problem: access tokens expire in 15 min — without refresh tokens, the user would need to log in every 15 minutes.
- [ ] Write a throwaway Express server with two routes: `/refresh` that issues a new short-lived access token given a valid refresh token, and `/logout` that revokes the refresh token.
- [ ] Try calling a protected route with an expired access token — observe the 401. Then call `/refresh` and retry with the new token — observe success.
- [ ] Write down in `_docs/decisions/auth-refresh-notes.md`: what a blocklist is, why refresh tokens are stored in Redis not Postgres.

**Examples**

```ts
// (example) the token lifecycle
// Login:   issue access_token (15m) + refresh_token (7d) → both in HttpOnly cookies
// Request: read access_token cookie → verify JWT → proceed
// Expired: 401 → client calls /auth/refresh → verify refresh_token → issue new access_token
// Logout:  add refresh_token ID to Redis blocklist → both cookies cleared
```

**Done checks**
- [ ] You can explain: why access tokens are short-lived, how the refresh flow works, what a blocklist does.

---

### B2) Refresh token integration

- [ ] On OAuth login success, issue both an `access_token` cookie (15 min) and a `refresh_token` cookie (7 days).
- [ ] Store the refresh token ID in Postgres (`RefreshToken` model with `userId`, `tokenId`, `expiresAt`, `revokedAt`).
- [ ] Add `POST /auth/refresh`: verify refresh token cookie → check it's not revoked → issue new access token cookie.
- [ ] Add `POST /auth/logout`: mark refresh token as revoked in Postgres (set `revokedAt`) → clear both cookies.
- [ ] In the web app, add a 401 response interceptor: automatically call `/auth/refresh` and retry the original request once before redirecting to login.

**Examples**

```ts
// (example) issuing both tokens on login
// const accessToken = jwt.sign({ userId }, env.JWT_SECRET, { expiresIn: "15m" });
// const refreshTokenId = crypto.randomUUID();
// const refreshToken = jwt.sign({ userId, tokenId: refreshTokenId }, env.JWT_REFRESH_SECRET, { expiresIn: "7d" });
//
// // Store the refresh token ID so we can revoke it
// await refreshTokenRepository.create({ userId, tokenId: refreshTokenId, expiresAt: addDays(new Date(), 7) });
//
// res.cookie("access_token", accessToken, { httpOnly: true, secure: isProd, sameSite: "strict", maxAge: 15 * 60 * 1000 });
// res.cookie("refresh_token", refreshToken, { httpOnly: true, secure: isProd, sameSite: "strict", path: "/auth/refresh", maxAge: 7 * 24 * 60 * 60 * 1000 });
```

**Done checks**
- [ ] Wait for access token to expire (or set it to 5s for testing) → app automatically refreshes and continues working.
- [ ] Call `/auth/logout` → refresh cookie is cleared → `/auth/refresh` now returns 401.

---

## Section C — Rate Limiting

### C1) Rate limiting on auth and message endpoints

- [ ] Install `express-rate-limit`: `npm install express-rate-limit`.
- [ ] Add a strict rate limiter to all `/auth/*` routes: max 10 requests per 15 minutes per IP.
- [ ] Add a message rate limiter: max 60 messages per minute per user.
- [ ] Add a general API rate limiter as a fallback on all `/api/v1/*` routes: max 200 requests per minute per IP.
- [ ] Confirm rate-limited requests receive a `429 Too Many Requests` response with a `Retry-After` header.

**Examples**

```ts
import rateLimit from "express-rate-limit";

// Strict limiter for auth routes
export const authRateLimit = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 10,
  standardHeaders: true,   // sends RateLimit-* headers
  legacyHeaders: false,
  message: { code: "rate_limited", message: "Too many attempts. Try again in 15 minutes." },
});

// General API limiter
export const apiRateLimit = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 200,
  standardHeaders: true,
  legacyHeaders: false,
});
```

**Done checks**
- [ ] Hit `/auth/google` 11 times rapidly → 11th request returns 429 with `Retry-After` header.
- [ ] Normal usage is completely unaffected.

---

## Exit criteria (demo script)

- [ ] Phase 01 + 02 features still work end-to-end.
- [ ] Attach an image in chat → it appears for both users; direct S3 URL → Access Denied.
- [ ] Wait for access token expiry → app auto-refreshes silently without re-login.
- [ ] Log out → refresh token revoked → `/auth/refresh` returns 401.
- [ ] Hit auth endpoint 11 times → 429 response on the 11th with a `Retry-After` header.
