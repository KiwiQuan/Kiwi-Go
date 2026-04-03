# 🛠 Official Tech Stack: Real-Time Messaging Platform

This document outlines the finalized technical stack for the messaging platform. The architecture is designed for high concurrency, type safety, and horizontal scalability.

---

## 🎨 Frontend (Client Side)

| Technology        | Role             | Description                                                                 |
| :---------------- | :--------------- | :-------------------------------------------------------------------------- |
| **React (TS)**    | UI Library       | Functional components with Hooks for a reactive user interface.             |
| **TypeScript**    | Language         | Ensures type safety across the frontend to catch errors during development. |
| **Redux Toolkit** | State Management | Manages global application state (auth, active rooms, message cache).       |
| **Tailwind CSS**  | Styling          | Utility-first CSS framework for rapid, responsive UI development.           |
| **Shadcn UI**     | UI Components    | Accessible primitives (Radix UI) styled with Tailwind for a polished look.  |

---

## ⚙️ Backend (Server Side)

| Technology     | Role          | Description                                                                  |
| :------------- | :------------ | :--------------------------------------------------------------------------- |
| **Node.js**    | Runtime       | The JavaScript environment for executing server-side code.                   |
| **Express**    | Web Framework | Lightweight and flexible framework for handling REST API endpoints.          |
| **TypeScript** | Language      | Provides strict typing for controllers, services, and middleware.            |
| **Socket.io**  | Real-Time     | Enables bi-directional, event-based communication between client and server. |

---

## 🗄️ Database & In-Memory Logic

| Technology     | Role             | Description                                                                           |
| :------------- | :--------------- | :------------------------------------------------------------------------------------ |
| **PostgreSQL** | Primary Database | Relational database for persistent storage of users, rooms, and messages.             |
| **Prisma**     | ORM              | Next-generation ORM for Type-safe database access and migrations.                     |
| **Redis**      | Pub/Sub & Cache  | Handles presence tracking (online status) and syncs messages across server instances. |

---

## 🚀 Infrastructure & DevOps

| Technology  | Role             | Description                                                             |
| :---------- | :--------------- | :---------------------------------------------------------------------- |
| **Docker**  | Containerization | Packages the application and its dependencies into a standardized unit. |
| **AWS ECS** | Orchestration    | Highly scalable container management service to run Docker containers.  |
| **AWS ECR** | Registry         | Docker container registry to store and manage production images.        |

---

## 🏗 High-Level Architecture & Data Flow

### 1. Authentication

- **Process**: User logs in via OAuth (Google/GitHub).
- **Storage**: User data is persisted in **PostgreSQL**.
- **Session**: A **JWT** is issued for stateless authentication across the API and WebSockets.

### 2. Real-Time Messaging Pipeline

- **Emission**: Client sends a message via **Socket.io**.
- **Persistence**: Express server validates and saves the message to **PostgreSQL** via **Prisma**.
- **Distribution**: The server publishes the message to a **Redis Pub/Sub** channel.
- **Broadcast**: All active server instances subscribe to Redis; when a message is received, they push it to the relevant connected clients via their own Socket.io pools.

### 3. Presence System

- **Tracking**: Connection/disconnection events update a "User Set" in **Redis**.
- **Visibility**: Clients query Redis (or receive updates) to show real-time "Online/Offline" indicators in the UI sidebar.
