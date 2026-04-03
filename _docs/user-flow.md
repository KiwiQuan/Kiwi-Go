# 🗺️ User Flow: Real-Time Messaging Platform

This document defines the comprehensive user journey through the application, mapping UI interactions to backend processes and data persistence.

---

## 1. Authentication & Entry

- **Provider Selection**: The user lands on the login page and selects either Google or GitHub via OAuth.
- **Token Issuance**: After the provider validates the user, the backend generates a **JWT** for session management.
- **Profile Initialization**: The system checks if the user exists in **PostgreSQL**; if not, a new user record is created with their username, email, and avatar URL.
- **Redirect**: The frontend receives the JWT and redirects the user to the main chat dashboard.

---

## 2. Global Presence & Connectivity

- **WebSocket Handshake**: The React client establishes a connection with the Node.js server using **Socket.IO**.
- **Global Status Update**:
  - Upon connection, the server adds the user's ID to an "Online Users" set in **Redis**.
  - A global broadcast is emitted to all connected clients to update the **Online** status indicator in the UI.
- **Presence Visibility**: The sidebar displays a green indicator for all users currently marked as online in the Redis cache.

---

## 3. Conversation Discovery & Group Management

- **List Retrieval**: The client fetches a list of existing private and group conversations from PostgreSQL.
- **Group Creation Flow**:
  1.  **Initiation**: User clicks a "New Group" button in the sidebar.
  2.  **Configuration**: A modal appears where the user enters the group name and description.
  3.  **Member Invitation**: The user selects multiple contacts to invite to the channel.
  4.  **Creation**: The backend creates a new entry in the **Conversations** table (type: group) and links the selected users.
  5.  **Synchronization**: The server uses **Redis Pub/Sub** to notify all invited members' active instances to join the new WebSocket room.

---

## 4. Real-Time Messaging & History

- **Room Entry**: Selecting a conversation triggers a `join_room` event for that specific `conversation_id`.
- **Infinite Scroll History**:
  - Initially, the client loads the 25 most recent messages from PostgreSQL.
  - As the user scrolls to the top of the chat window, the frontend triggers a fetch request for the next batch of older messages based on the earliest `created_at` timestamp.
- **Active Interaction**:
  - **Typing Indicators**: As the user types in the input field, a `typing` event is emitted and broadcast to the room.
  - **Sending Messages**: When the user hits send, the message is saved to PostgreSQL and simultaneously broadcast via Redis to all server instances for real-time delivery.

---

## 5. Media Sharing

- **File Selection**: The user clicks the **Paperclip Icon** in the message input bar to open the file explorer.
- **Upload Pipeline**:
  1.  The selected file is uploaded to **AWS S3**.
  2.  Upon successful upload, a message is emitted containing the S3 media URL.
  3.  The recipient’s client renders the URL as an image, video, or file download link.

---

## 6. Disconnection & Cleanup

- **Termination**: The user logs out or closes the browser tab.
- **State Cleanup**:
  - The `disconnect` event triggers the backend to remove the user from the **Redis** online set.
  - A final "Last Seen" timestamp is updated in the PostgreSQL **Users** table.
  - A global broadcast updates all other clients, changing the user’s status indicator from green to offline.
