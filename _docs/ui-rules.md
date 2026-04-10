# 📐 UI Design Principles

These rules guide the structural and functional layout of the messaging platform to ensure a seamless user journey from login to real-time interaction.

### 1. Mobile-First Responsiveness

- **Bottom-Up Design:** Navigation (Groups, Chat, Profile) must be pinned to the bottom on mobile devices for thumb-reachability.
- **Adaptive Sidebar:** The sidebar containing the list of conversations must transition into a full-screen drawer on mobile while remaining a persistent column on desktop.
- **Fluid Widths:** Use flexible grids to ensure message bubbles and media previews scale across all device sizes without horizontal overflow.

### 2. Real-Time Feedback & Presence

- **Presence Indicators:** Use the "Online" emerald status indicator consistently across sidebar, chat header, and group member lists.
- **Typing States:** Display typing indicators immediately upon receiving the `typing` event via Socket.io to reduce perceived latency.
- **Optimistic UI:** When a user sends a message, it should appear in the local UI instantly with a "sending" state before PostgreSQL persistence is confirmed.

### 3. Performance-Centric Navigation

- **Infinite Scroll Virtualization:** Use windowing/virtualization to only render messages currently in the viewport to prevent DOM lag.
- **Progressive Disclosure:** Hide complex group configuration or member invitation tools within modals to keep the main chat interface clean.
- **Media States:** During AWS S3 uploads, show a progress bar or blur-hash placeholder in the thread so the user knows the file is processing.
