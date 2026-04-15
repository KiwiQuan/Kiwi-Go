# Phase 03 — Media + Security Hardening (production behavior)

## Goal
Add **file/image sharing** (per user flow) while leveling-up security: token lifecycle, authorization checks, rate limiting, and safe uploads.

## Scope
- Users can upload files/images and share them in chat.
- Access is controlled (no public buckets; signed URLs).
- Auth is hardened (refresh tokens and revocation strategy).
- Abuse controls exist (rate limiting, basic spam protections).

## Deliverables
- **Uploads**: S3 signed URL workflow (or presigned POST) + message with media URL
- **Authorization**: enforce conversation membership on all message/media actions
- **Token lifecycle**: refresh token flow + revocation/blocklist strategy
- **Security defaults**: CORS, secure cookies, input validation everywhere, sane limits

## Learning goals
- **Secure-by-default** thinking: least privilege, validate inputs, deny by default.
- **Cloud primitives**: S3 permissions, signed URLs, upload constraints.
- **Threat modeling**: what can go wrong (XSS, CSRF, SSRF, spam, token theft).

## Features

### 1) S3 upload pipeline (signed URLs)
- Create API endpoint to request an upload URL (auth required).
- Validate file metadata (type/size) and return a signed upload URL.
- Upload directly from web to S3 using the signed URL.
- Emit a message containing the final media URL + metadata (type, size).
- Render media messages (image preview + downloadable link) accessibly.

### 2) Media authorization + privacy
- Ensure only conversation members can create media messages for that conversation.
- Keep S3 bucket private; use signed GET URLs (or CloudFront signed URLs) for viewing.
- Add server-side metadata record for uploads (owner, conversationId, key, mime).
- Add cleanup strategy for failed/aborted uploads (best-effort scheduled cleanup).
- Add maximum retention rules (documented; optional enforcement).

### 3) Refresh tokens + revocation
- Add refresh token issuance (HttpOnly cookie) on login.
- Implement refresh endpoint returning new access token cookie.
- Add revocation on logout (store refresh token IDs / blocklist in Redis).
- Update websocket auth to handle access token rotation (re-auth on reconnect).
- Add “session expired” UX flow in web (re-login gracefully).

### 4) Rate limiting + basic abuse controls
- Add rate limits for auth endpoints and message send endpoints.
- Add websocket-side throttling for `typing` and message spam.
- Add payload size limits (REST + sockets) and enforce on server.
- Add basic audit logs for security-relevant actions (login/logout/upload).
- Add a simple “report message” placeholder endpoint (no UI required yet).

## Exit criteria (demo script)
- Upload an image → it appears in chat for another user.
- Attempt to access a media URL without membership → denied/signed URL fails.
- Logout → refresh token revoked; new session cannot be minted.
- Hammer message send → rate limiting triggers with clean errors.

