# 🎨 Theme & Aesthetic Standards (Minimalist)

The application follows a "Modern Minimalist" aesthetic, utilizing Shadcn UI and Tailwind CSS to maintain a professional, clean, and focus-driven interface.

### 1. Color Palette (Functional & Neutral)

| Element                | Color / Style                       | Purpose                                  |
| :--------------------- | :---------------------------------- | :--------------------------------------- |
| **Primary Background** | Slate-50 (Light) / Slate-950 (Dark) | Clean, distraction-free canvas.          |
| **Surface/Cards**      | White / Slate-900                   | Sidebar and chat bubbles.                |
| **Accent (Presence)**  | Emerald-500                         | Indicates "Online" status in Redis.      |
| **Primary Action**     | Indigo-600                          | Used for "New Group" and "Send" buttons. |
| **Subtle Text**        | Slate-500                           | Timestamps and "Last Seen" data.         |

### 2. Typography

- **Font Family:** Use a clean, sans-serif stack for maximum readability in chat threads.
- **Hierarchy:** Use bold weights for usernames, regular weights for message content, and small, light weights for read receipts and timestamps.

### 3. Iconographic Language

- **Icon Set:** Use a consistent, thin-stroke icon set for a modern look.
- **Functional Icons:**
  - **Paperclip:** Triggers the S3 upload pipeline.
  - **Users/Plus:** Entry point for Group Management.
  - **Checkmarks:** Single check for "Sent" and double check for "Read Receipts".

### 4. Motion & Animation

- **Micro-interactions:** Use subtle transitions for opening the "New Group" modal or expanding the media explorer.
- **Thread Transitions:** New messages should slide up slightly or fade in to draw the eye without being jarring.
- **Layout Animations:** When a user is added to the "Online Users" set in Redis, their status indicator should fade into the "Online" state.

### 5. Shapes & Border Styles

- **Border Radius:** Use `rounded-lg` (8px) for chat bubbles and `rounded-full` for user avatars to create a friendly, modern feel.
- **Borders:** Use thin, high-contrast borders (Slate-200 or Slate-800) instead of heavy shadows to maintain the minimalist requirement.
