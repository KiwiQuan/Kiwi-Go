# 🛠 Official Tech Stack: Real-Time Messaging Platform

This document outlines the finalized technical stack and the operational standards required to ensure high concurrency, type safety, and horizontal scalability.

---

## 🎨 Frontend (Client Side)

| Technology            | Best Practices & Conventions                                                                                                       | Limitations & Common Pitfalls                                                                                                       |
| :-------------------- | :--------------------------------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------- |
| **React (TS)**        | Use **Functional Components** and custom hooks to separate UI from logic. Implement **Atomic Design** for folder structure.        | **Pitfall:** Over-rendering components by passing large objects as props. **Mitigation:** Use `React.memo` or split components.     |
| **TypeScript**        | Enforce **Strict Mode**. Define global interfaces in a `types/` directory to share between frontend and backend.                   | **Limitation:** Can lead to "Type Gymnastics" if over-engineered. Avoid using `any` at all costs.                                   |
| **Redux Toolkit**     | Use **RTK Query** for server state (fetching chat history) and standard Slices for local UI state (modals, active tabs).           | **Pitfall:** Storing massive message histories in the store. **Mitigation:** Implement windowing/virtualization for long lists.     |
| **Tailwind & Shadcn** | Stick to the **Theme Configuration** (`tailwind.config.js`) for colors/spacing. Use Shadcn's CLI to only add necessary components. | **Consideration:** Tailwind can lead to long class strings. Use the `clsx` or `tailwind-merge` utilities to manage dynamic classes. |

---

## ⚙️ Backend (Server Side)

| Technology            | Best Practices & Conventions                                                                                                         | Limitations & Common Pitfalls                                                                                                                                 |
| :-------------------- | :----------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Express & Node.js** | Implement a **Three-Layer Architecture** (Controller, Service, Data Access). Use a global error-handling middleware.                 | **Limitation:** Express is unopinionated. **Pitfall:** Logic leakage where database queries are written directly inside route handlers.                       |
| **Socket.io**         | Use **Namespaces and Rooms** for scoping messages (e.g., `room_${conversation_id}`). Use the `@socket.io/redis-adapter` for scaling. | **Pitfall:** Memory leaks from not cleaning up listeners on `disconnect`. **Consideration:** Always implement a heartbeat/ping to detect "ghost" connections. |
| **JWT Auth**          | Store tokens in **HttpOnly Cookies** to prevent XSS attacks. Keep access tokens short-lived and implement refresh tokens.            | **Limitation:** JWTs are hard to revoke before expiration. **Mitigation:** Maintain a "blocklist" in Redis for logged-out tokens.                             |

---

## 🗄️ Database & In-Memory Logic

| Technology     | Best Practices & Conventions                                                                                                           | Limitations & Common Pitfalls                                                                                                                                                   |
| :------------- | :------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **PostgreSQL** | Use **Indexes** on `conversation_id` and `created_at` to ensure fast retrieval of chat history.                                        | **Pitfall:** N+1 query problems when fetching messages and their senders. **Mitigation:** Use Prisma's `include` or `select` carefully.                                         |
| **Prisma ORM** | Use **Prisma Migrations** for all schema changes. Leverage the generated client for 100% type safety between DB and API.               | **Limitation:** Connection pooling in serverless/highly scaled environments. **Mitigation:** Use **Prisma Accelerate** or a dedicated connection bouncer (PgBouncer).           |
| **Redis**      | Use **Pub/Sub** for message broadcasting and **Sets** for tracking online user IDs. Set TTLs (Time-To-Live) on temporary session data. | **Pitfall:** Redis is in-memory; if it restarts, unsaved volatile data is lost. **Mitigation:** Ensure persistence (RDB/AOF) is enabled if using it for more than just a cache. |

---

## 🚀 Infrastructure & DevOps

| Technology  | Best Practices & Conventions                                                                                                              | Limitations & Common Pitfalls                                                                                                          |
| :---------- | :---------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------- |
| **Docker**  | Use **Multi-stage builds** to keep production images small. Never include sensitive `.env` files in the image; use build-args or secrets. | **Pitfall:** Running containers as `root` user. **Best Practice:** Create a non-privileged user in the Dockerfile.                     |
| **AWS ECS** | Use **Fargate** for serverless scaling to avoid managing underlying EC2 instances. Implement **Health Checks** for auto-recovery.         | **Consideration:** Cold starts can occur during scaling. **Pitfall:** Hardcoding AWS resource ARNs; use environment variables instead. |
| **AWS S3**  | Use **Signed URLs** for media sharing to ensure files are only accessible to authorized users.                                            | **Pitfall:** Leaving S3 buckets public. **Mitigation:** Enable "Block Public Access" and use CloudFront or Signed URLs for delivery.   |

---

## 🏗 High-Level Conventions

### 1. Error Handling & Logging

- **Standard:** Use a library like `winston` or `pino` for structured JSON logging (essential for AWS CloudWatch).
- **Pitfall:** Sending raw error stack traces to the frontend. Always return a sanitized error message and a unique Error ID.

### 2. Environment Management

- **Standard:** Maintain strict parity between `.env.example` and actual environment variables.
- **Validation:** Use a tool like `zod` to validate that all required environment variables exist at server startup.

### 3. Scaling Strategy

- **Vertical:** Start with small ECS tasks (e.g., 0.5 vCPU, 1GB RAM).
- **Horizontal:** Use **Redis Pub/Sub** to ensure that when a message is sent to Server A, users connected to Server B still receive it in real-time.

### 4. Real-Time Data Flow

- **Authentication**: OAuth provider returns user info → Backend validates/creates user in PostgreSQL → JWT issued to client.
- **Presence**: Connection event adds User ID to Redis set → Global broadcast updates online status in UI.
- **Messaging**: Message saved to PostgreSQL → Published to Redis Pub/Sub → Broadcasted to all server instances → Delivered to relevant Socket.io clients.
