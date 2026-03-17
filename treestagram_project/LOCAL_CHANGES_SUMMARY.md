# Local Changes Summary — Compared to `origin/develop`

**Generated:** March 16, 2026 (updated March 17, 2026 — Notification feature added)  
**Branch:** `develop` (local is behind `origin/develop` by 8 commits)  
**Status:** All changes are unstaged (not committed)

---

## Overview

The local codebase has **major new features** added across both the Django backend and Svelte frontend. The changes implement a full **Posts system** (create, read, delete), **Likes**, **Comments** (with edit/delete), **User tagging**, a full **In-App Notifications system** (with real-time polling, bell dropdown panel, and mark-as-read), and various **UI/UX improvements**. There are also **4 new untracked files** (a changelog and three Django migrations).

---

## Table of Contents

1. [Modified Files (12 total)](#modified-files)
2. [New / Untracked Files (4 total)](#new-untracked-files)
3. [Feature-by-Feature Breakdown](#feature-by-feature-breakdown)
4. [API Endpoints Added](#api-endpoints-added)
5. [Database Changes](#database-changes)
6. [UI/UX Changes](#uiux-changes)

---

## Modified Files

### 1. `accounts/models.py` — New Django Models

**What changed:** Four entirely new models added after the existing `User` model.

| Model | Purpose | Key Fields |
|-------|---------|------------|
| **`Post`** | A tree observation / post in the feed | `author` (FK→User), `tree_name`, `body`, `health` (Good/Fair/Poor), `borough`, `image` (ImageField), `tagged_users` (M2M→User), `created_at`, `updated_at` |
| **`Like`** | A like on a post (unique per user+post) | `user` (FK→User), `post` (FK→Post), `created_at`. Has `unique_together = ('user', 'post')` |
| **`Comment`** | A comment on a post | `author` (FK→User), `post` (FK→Post), `text`, `created_at` |
| **`Notification`** | In-app notification for user activity | `recipient` (FK→User), `sender` (FK→User, nullable), `notif_type` (like/comment/tag/promotion), `post` (FK→Post, nullable), `comment` (FK→Comment, nullable), `message`, `is_read` (default False), `created_at` |

- `Post` orders by `-created_at` (newest first)
- `Comment` orders by `created_at` (chronological)
- `Notification` orders by `-created_at` (newest first)
- `tagged_users` is a ManyToManyField allowing posts to tag other users
- `Notification.sender` is nullable to support system-generated notifications (e.g. role promotions)
- `Notification.notif_type` choices: `like`, `comment`, `tag`, `promotion`

---

### 2. `accounts/api_views.py` — 14 New API View Functions + 3 Helpers

**~430 lines added.** This is the biggest backend change. Added:

#### Post / Like / Comment endpoints:

| Function | Method | Endpoint | Description |
|----------|--------|----------|-------------|
| `_post_to_dict()` | — | — | Helper: serializes a Post to JSON dict with author info, like count, liked status, comments, tagged users |
| `api_fetch_posts()` | GET | `/api/posts/` | Fetch all posts (public feed, newest first) |
| `api_fetch_my_posts()` | GET | `/api/my-posts/` | Fetch posts authored by the logged-in user |
| `api_fetch_my_tagged_posts()` | GET | `/api/my-tagged-posts/` | Fetch posts where the logged-in user is tagged |
| `api_create_post()` | POST | `/api/posts/create/` | Create a new post (supports multipart for image upload + tagged users) |
| `api_delete_post()` | POST | `/api/posts/<id>/delete/` | Delete a post (author only) |
| `api_toggle_like()` | POST | `/api/posts/<id>/like/` | Toggle like on a post; updates author's `total_likes_received` |
| `api_add_comment()` | POST | `/api/posts/<id>/comment/` | Add a comment to a post |
| `api_edit_comment()` | POST | `/api/comments/<id>/edit/` | Edit a comment (author only) |
| `api_delete_comment()` | POST | `/api/comments/<id>/delete/` | Delete a comment (comment author or post author) |

#### Notification endpoints (NEW):

| Function | Method | Endpoint | Description |
|----------|--------|----------|-------------|
| `_create_notification()` | — | — | Helper: creates a Notification record. **Auto-skips if sender == recipient** (no self-notifications) |
| `_notif_to_dict()` | — | — | Helper: serializes a Notification to JSON dict with sender info, post_id, timestamps |
| `api_notifications()` | GET | `/api/notifications/` | Fetch latest 50 notifications for the current user + unread count |
| `api_notifications_unread_count()` | GET | `/api/notifications/unread-count/` | Lightweight endpoint returning just the unread count (used for polling) |
| `api_notifications_mark_read()` | POST | `/api/notifications/mark-read/` | Mark specific notification IDs as read (JSON body: `{ ids: [...] }`) |
| `api_notifications_mark_all_read()` | POST | `/api/notifications/mark-all-read/` | Mark ALL of the user's unread notifications as read |

#### Notification triggers wired into existing views:

| Trigger Point | Notification Type | Who Gets Notified | Condition |
|---------------|-------------------|-------------------|-----------|
| `api_toggle_like()` | `like` | Post author | Only when a like is **created** (not on unlike) |
| `api_add_comment()` | `comment` | Post author | Always (skipped if you comment on your own post) |
| `api_create_post()` | `tag` | Each tagged user | For every user in the `tagged_users` list |
| `api_toggle_like()` | `promotion` | Post author | Only when `promote_if_eligible()` actually changes the role |
| `api_create_post()` | `promotion` | Post creator | Only when `promote_if_eligible()` actually changes the role |

Key behaviors:
- All write endpoints require authentication (return 401 if not logged in)
- Post creation updates `user.post_count` and calls `promote_if_eligible()` for role promotion
- Like toggling updates the post author's `total_likes_received` and calls `promote_if_eligible()`
- **Self-notifications are automatically suppressed** — `_create_notification()` returns `None` if `sender.id == recipient.id`
- Query optimization with `select_related` and `prefetch_related` throughout

---

### 3. `accounts/api_urls.py` — 14 New URL Routes

Added URL patterns for all the new endpoints:

```
posts/                              → api_fetch_posts
my-posts/                           → api_fetch_my_posts
my-tagged-posts/                    → api_fetch_my_tagged_posts
posts/create/                       → api_create_post
posts/<int:post_id>/delete/         → api_delete_post
posts/<int:post_id>/like/           → api_toggle_like
posts/<int:post_id>/comment/        → api_add_comment
comments/<int:comment_id>/edit/     → api_edit_comment
comments/<int:comment_id>/delete/   → api_delete_comment
notifications/                      → api_notifications          (NEW)
notifications/unread-count/         → api_notifications_unread_count  (NEW)
notifications/mark-read/            → api_mark_notifications_read     (NEW)
notifications/mark-all-read/        → api_mark_all_notifications_read (NEW)
```

---

### 4. `treestagram/urls.py` — Media File Serving Fix

**Problem fixed:** Uploaded post images returned 404 because `DEBUG=False` by default, which caused `static()` to return an empty list.

**Changes:**
- Added import: `from django.views.static import serve as static_serve`
- Added an explicit `re_path(r'^media/(?P<path>.*)$', static_serve, ...)` that works regardless of the `DEBUG` setting
- Removed the old `+ static(settings.MEDIA_URL, ...)` that only worked with `DEBUG=True`

---

### 5. `treestagram-svelte/src/lib/api.js` — 13 New Frontend API Functions

**~95 lines added.** All functions use the existing `apiFetch` wrapper.

#### Post / Like / Comment functions:

| Function | Method | Endpoint | Notes |
|----------|--------|----------|-------|
| `apiFetchPosts()` | GET | `/api/posts/` | Returns all posts |
| `apiFetchMyPosts()` | GET | `/api/my-posts/` | User's own posts |
| `apiFetchMyTaggedPosts()` | GET | `/api/my-tagged-posts/` | Posts where user is tagged |
| `apiCreatePost(data)` | POST | `/api/posts/create/` | Sends `FormData` (multipart) for image support |
| `apiDeletePost(postId)` | POST | `/api/posts/<id>/delete/` | Delete by ID |
| `apiToggleLike(postId)` | POST | `/api/posts/<id>/like/` | Toggle like |
| `apiAddComment(postId, text)` | POST | `/api/posts/<id>/comment/` | JSON body with `{ text }` |
| `apiEditComment(commentId, text)` | POST | `/api/comments/<id>/edit/` | JSON body with `{ text }` |
| `apiDeleteComment(commentId)` | POST | `/api/comments/<id>/delete/` | Delete comment |

#### Notification functions (NEW):

| Function | Method | Endpoint | Notes |
|----------|--------|----------|-------|
| `apiFetchNotifications()` | GET | `/api/notifications/` | Returns list of latest 50 notifications + unread count |
| `apiFetchUnreadCount()` | GET | `/api/notifications/unread-count/` | Lightweight call returning `{ unread_count: N }` |
| `apiMarkNotificationsRead(ids)` | POST | `/api/notifications/mark-read/` | Marks specific notification IDs as read |
| `apiMarkAllNotificationsRead()` | POST | `/api/notifications/mark-all-read/` | Marks all unread notifications as read |

---

### 6. `treestagram-svelte/src/routes/Home.svelte` — Live Feed with Real Data

**Massive overhaul.** Changed from static placeholder posts to a fully functional feed.

#### Script changes:
- Added imports: `apiFetchPosts`, `apiToggleLike`, `apiAddComment`, `apiEditComment`, `apiDeleteComment`
- New state variables: `posts`, `loading`, `commentDrafts`, `showCommentForm`, `editingCommentId`, `editingCommentText`
- `onMount` now calls `loadPosts()` which fetches real posts from the API
- Added functions:
  - `loadPosts()` — fetches posts from backend
  - `toggleLike(index)` — toggles like on a post
  - `toggleCommentFormVisibility(index)` — shows/hides comment input
  - `addComment(index)` — submits a new comment
  - `startEditComment()`, `cancelEditComment()`, `saveEditComment()` — inline comment editing
  - `deleteComment()` — deletes a comment
  - `healthIcon(h)` — returns emoji based on health status
- **Removed:** Static `placeholderPosts` array with hardcoded data

#### Template changes:
- **Removed:** "Create post" input bar at the top
- **Added:** Loading state ("Loading posts…") and empty state ("No posts yet")
- Posts now render from real API data with: tree name, borough, health badge, author username, body text, uploaded images, like button with count, comment section
- **Comment system:** Each post shows its comments, with inline edit/delete for own comments
- Comment form appears when "💬 Comment" is clicked
- Like button shows filled/unfilled state based on `post.liked`

#### Style changes:
- Replaced all hardcoded colors (`#faf9f6`, `#cdd9af`, `#a44a3f`, etc.) with CSS custom properties (`var(--t-bg-base)`, `var(--t-bg-elevated)`, `var(--t-text-brand)`, etc.) for proper theme support
- **Removed:** `.create-post`, `.create-avatar`, `.create-input`, `.create-btn` styles (~47 lines)
- **Added:** `.post-photo`, `.post-photo img` styles for uploaded images
- **Added:** Full comment system styles: `.post-comments`, `.comment-list`, `.comment-item`, `.comment-text`, `.comment-actions`, `.comment-action-btn`, `.comment-edit-row`, `.comment-edit-input`, `.comment-form`, `.comment-submit` (~106 lines)
- **Added:** `.action-btn.liked` state style
- Post cards now use theme variables for backgrounds, borders, shadows

---

### 7. `treestagram-svelte/src/routes/Profile.svelte` — Complete Rewrite (~1970 lines added)

**This is the largest change.** The profile page went from ~527 lines of static placeholder UI to ~2497 lines of fully functional profile.

#### Key features added:

**A. "Plant a Post" — 3-Step Carousel Modal:**
1. **Step 1 — Tree Info:** Tree name (required), borough dropdown (5 NYC boroughs), health status picker (Good/Fair/Poor)
2. **Step 2 — Photo:** Drag-and-drop / click-to-upload photo area with preview and remove option
3. **Step 3 — Observation:** Text area for notes, tag other users with `@username` syntax, live preview card showing what the post will look like
- Animated slide transitions between steps
- Spinner state while posting
- Step indicator dots (active, done, pending)

**B. Posts Grid:**
- Displays the user's actual posts in a 3-column Instagram-style grid
- Each post tile shows the image (or health emoji if no image)
- Hover overlay shows like count and comment count
- Click opens a detailed modal

**C. Tagged Posts Tab:**
- New "🏷️ Tagged" tab alongside "Posts"
- Shows posts where the current user is tagged by others
- Shows "📸 by @username" badge on hover

**D. Post Detail Modal (Instagram-style):**
- Split layout: image on the left, details on the right
- Header with author avatar, username, location
- Tree info card with health badge
- Body text, tagged users display
- Time ago display (e.g., "2h", "3d", "1w")
- Full comment section with:
  - Comment avatars (initial letter)
  - Inline edit (✏️) for own comments
  - Delete (🗑️) for own comments or post owner's comments
  - Comment input with send button
- Like button with heart animation on toggle
- 3-dot menu with "Delete Post" option (includes confirmation dialog with "Keep"/"Delete" buttons and spinner)

**E. Right Sidebar — Real Data:**
- "About Me" card now shows user's actual bio
- "My Recent Posts" shows up to 4 most recent posts with health icons
- "Progress to Credible User" shows real progress bars:
  - Posts: `user.post_count / 30`
  - Likes: `user.total_likes_received / 100`
  - Shows "✓ Met" badge and credible banner when both thresholds reached

**F. Profile Stats — Real Data:**
- Post count from `user.post_count`
- Likes received from `user.total_likes_received`
- Leaves count from `user.leaves`

#### Style changes:
- All hardcoded colors replaced with CSS custom properties for theme support
- Responsive design with `@media (max-width: 768px)` breakpoint
- Smooth animations: modal slide-up, carousel slide transitions, heart bounce, floating emoji, loading spinners, button breathing glow
- Custom scrollbar styling for modal body

---

### 8. `treestagram-svelte/src/components/LeftNav.svelte` — UI Polish + Notification Bell & Panel

**Changes (UI Polish):**
- **Theme toggle button commented out** (lines 65–74 wrapped in `<!-- -->`) — app now uses default theme only
- Nav label styling improvements:
  - Hover state: labels turn lighter color (`#dce8d4`)
  - Active state: labels also get lighter color + bold weight
  - Font size slightly increased (0.88rem → 0.9rem)
  - Added `text-shadow` for depth
  - Added color transition animation
  - Font weight changed to 500 (medium)
- Logout button label colors tweaked:
  - Default: `#cd3b3b` (was `#e57373`)
  - Hover: `#e05252` (new)
  - Added `text-shadow` for depth

**Changes (Notification System — NEW):**

The static bell icon in the nav was transformed into a fully interactive notification center:

#### Script changes:
- Imported `apiFetchNotifications`, `apiFetchUnreadCount`, `apiMarkNotificationsRead`, `apiMarkAllNotificationsRead` from `api.js`
- New reactive state: `showNotifPanel`, `notifications`, `unreadCount`, `loadingNotifs`, `pollTimer`
- **Polling mechanism:** Every **10 seconds**, the component polls the backend:
  - If the notification panel is **open** → fetches the full notifications list (keeps the panel live-updated)
  - If the notification panel is **closed** → fetches only the lightweight unread count (minimal overhead)
- Helper functions added:
  - `fetchUnreadCount()` — fetches unread count from backend
  - `fetchNotifications()` — fetches full notification list
  - `toggleNotifPanel()` — opens/closes the dropdown panel; on open, fetches fresh notifications
  - `handleClickOutside()` — closes the panel when clicking outside it
  - `markRead(id)` — marks a single notification as read (instant UI update + API call)
  - `markAllRead()` — marks all notifications as read
  - `timeAgo(dateStr)` — converts ISO timestamp to human-friendly format ("2m", "1h", "3d", "2w")
  - `typeIcon(type)` — returns emoji icon for notification type (❤️ like, 💬 comment, 🏷️ tag, ⭐ promotion)
- `onMount` sets up the polling interval + click-outside listener
- `onDestroy` cleans up the polling timer + click-outside listener

#### Template changes:
- Bell button now shows a **red unread badge** with the count when `unreadCount > 0`
- Clicking the bell toggles a **dropdown notification panel** with:
  - **Header:** "Notifications" title + "Mark all read" button
  - **Notification list:** Each item shows sender profile picture (or initial avatar), message, time ago, type icon, and read/unread styling
  - **Empty state:** "No notifications yet 🌿" when there are no notifications
  - **Loading state:** Spinner shown while fetching
- Unread notifications have a distinct background highlight; clicking one marks it as read
- Used `<button>` instead of `<div>` for click handlers (accessibility fix for a11y warnings)

#### Style changes (~120 lines added):
- `.notif-bell` — positioned button with relative positioning for badge
- `.notif-badge` — red circular badge with pulse animation for attention
- `.notif-panel` — absolute-positioned dropdown with shadow, border-radius, max-height with scroll
- `.notif-header` — flexbox header with title and "Mark all read" link
- `.notif-list` — scrollable list container
- `.notif-item` — individual notification row with hover effect and unread highlight
- `.notif-avatar` — circular avatar showing sender's profile picture or initial letter
- `.notif-body` — message text with type icon and time ago
- `.notif-empty` — centered empty state with muted styling
- Smooth transitions and hover effects throughout

---

### 9. `treestagram-svelte/src/theme.js` — Default Theme Changed to Light

**Changes:**
- **Migration logic added:** On first load, clears any stale `dark` preference from localStorage using a `treestagram-theme-v2` migration flag
- **Default theme changed from `dark` to `light`:** The `detectInitialTheme()` function no longer checks `prefers-color-scheme: dark`; it simply defaults to `'light'` if no saved preference exists
- **`resetToSystemTheme()` updated:** Now resets to `'light'` instead of checking OS dark mode preference

---

### 10. `treestagram-svelte/package-lock.json` & `node_modules/.package-lock.json`

**Minor change:** Three dependencies (`@sveltejs/vite-plugin-svelte`, `svelte`, `vite`) had `"peer": true` added to their entries. This is likely from an npm operation that resolved peer dependency relationships. **No functional change.**

---

### 11. `db.sqlite3` — Binary Database File

The SQLite database has been modified (new tables created from migrations, possibly test data added). This is a binary diff.

---

## New / Untracked Files

### 1. `CHANGELOG.md` (381 lines)

A detailed changelog documenting four phases of development:
1. Home Feed Posts Feature — adding `apiFetchPosts`, `apiToggleLike`, `apiAddComment` + backend
2. Profile Page Feature — adding `apiFetchMyPosts`, `apiCreatePost`, `apiDeletePost`, tagged posts + backend
3. Disable Theme Toggle — commenting out the theme cycle button
4. Fix Post Images Not Displaying — media serving fix in `urls.py`

### 2. `accounts/migrations/0002_post_comment_like.py`

Auto-generated Django migration that creates three new database tables:
- `accounts_post` — with all Post model fields
- `accounts_comment` — with all Comment model fields
- `accounts_like` — with unique_together constraint on (user, post)

### 3. `accounts/migrations/0003_post_tagged_users.py`

Auto-generated Django migration that adds the `tagged_users` ManyToManyField to the Post model, creating the `accounts_post_tagged_users` join table.

### 4. `accounts/migrations/0004_notification.py` (NEW)

Auto-generated Django migration that creates the `accounts_notification` table with fields:
- `recipient` (FK→User, `related_name='notifications'`)
- `sender` (FK→User, nullable, `related_name='sent_notifications'`, `on_delete=SET_NULL`)
- `notif_type` (CharField with choices: like, comment, tag, promotion)
- `message` (TextField)
- `post` (FK→Post, nullable)
- `comment` (FK→Comment, nullable)
- `is_read` (BooleanField, default=False)
- `created_at` (DateTimeField, auto_now_add)
- Ordered by `-created_at` (newest first)

---

## Feature-by-Feature Breakdown

### Feature 1: Posts System
| Layer | Files Changed |
|-------|--------------|
| Model | `accounts/models.py` (Post model) |
| API Views | `accounts/api_views.py` (create, fetch, delete) |
| API URLs | `accounts/api_urls.py` (5 routes) |
| Frontend API | `treestagram-svelte/src/lib/api.js` (5 functions) |
| Frontend UI | `Home.svelte` (feed display), `Profile.svelte` (grid + create modal) |
| Media serving | `treestagram/urls.py` (image serving fix) |
| Database | 2 migrations |

### Feature 2: Likes System
| Layer | Files Changed |
|-------|--------------|
| Model | `accounts/models.py` (Like model) |
| API Views | `accounts/api_views.py` (toggle like, update counts) |
| API URLs | `accounts/api_urls.py` (1 route) |
| Frontend API | `api.js` (`apiToggleLike`) |
| Frontend UI | `Home.svelte` (like button), `Profile.svelte` (modal like) |

### Feature 3: Comments System (with Edit/Delete)
| Layer | Files Changed |
|-------|--------------|
| Model | `accounts/models.py` (Comment model) |
| API Views | `accounts/api_views.py` (add, edit, delete comment) |
| API URLs | `accounts/api_urls.py` (3 routes) |
| Frontend API | `api.js` (`apiAddComment`, `apiEditComment`, `apiDeleteComment`) |
| Frontend UI | `Home.svelte` (inline comments), `Profile.svelte` (modal comments) |

### Feature 4: User Tagging
| Layer | Files Changed |
|-------|--------------|
| Model | `accounts/models.py` (tagged_users M2M on Post) |
| API Views | `accounts/api_views.py` (handle tagged_users in create, serialize in fetch) |
| Frontend UI | `Profile.svelte` (tag input in carousel, tagged tab, display in modal) |
| Database | 1 migration (0003) |

### Feature 5: In-App Notifications System (NEW)
| Layer | Files Changed |
|-------|--------------|
| Model | `accounts/models.py` (Notification model) |
| API Views | `accounts/api_views.py` (2 helpers + 4 endpoints + wired into like/comment/post/promote) |
| API URLs | `accounts/api_urls.py` (4 routes) |
| Frontend API | `treestagram-svelte/src/lib/api.js` (4 functions) |
| Frontend UI | `LeftNav.svelte` (bell badge, dropdown panel, polling, mark-as-read) |
| Database | 1 migration (0004) |

**How it works end-to-end:**
1. User A likes User B's post → backend creates a `Notification(type='like')` for User B
2. User B's `LeftNav` polls `/api/notifications/unread-count/` every 10 seconds → badge shows "1"
3. User B clicks the bell → panel opens, fetches full notification list from `/api/notifications/`
4. While panel is open, full list is re-fetched every 10 seconds (live updates)
5. User B clicks a notification → it is marked as read via `/api/notifications/mark-read/`
6. "Mark all read" button clears all unread notifications at once

**Notification types generated:**
| Type | Trigger | Message Example |
|------|---------|-----------------|
| `like` | Someone likes your post | "@alice liked your post "Oak Tree"" |
| `comment` | Someone comments on your post | "@bob commented on your post "Maple"" |
| `tag` | Someone tags you in a post | "@carol tagged you in a post "Birch"" |
| `promotion` | You get promoted to credible user | "Congratulations! You've been promoted to Credible Contributor!" |

### Feature 6: Theme & UI Polish
| Layer | Files Changed |
|-------|--------------|
| Theme default | `theme.js` (changed default from dark to light, migration logic) |
| Nav polish | `LeftNav.svelte` (hover/active styles, theme toggle disabled) |
| Theme tokens | `Home.svelte`, `Profile.svelte` (replaced hardcoded colors with CSS vars) |

---

## API Endpoints Added

| Method | URL | Auth? | Description |
|--------|-----|-------|-------------|
| GET | `/api/posts/` | No | Fetch all posts (feed) |
| GET | `/api/my-posts/` | Yes | Fetch logged-in user's posts |
| GET | `/api/my-tagged-posts/` | Yes | Fetch posts where user is tagged |
| POST | `/api/posts/create/` | Yes | Create a new post (multipart) |
| POST | `/api/posts/<id>/delete/` | Yes | Delete a post (author only) |
| POST | `/api/posts/<id>/like/` | Yes | Toggle like on a post |
| POST | `/api/posts/<id>/comment/` | Yes | Add a comment |
| POST | `/api/comments/<id>/edit/` | Yes | Edit a comment (author only) |
| POST | `/api/comments/<id>/delete/` | Yes | Delete a comment (author or post owner) |
| GET | `/api/notifications/` | Yes | Fetch latest 50 notifications + unread count **(NEW)** |
| GET | `/api/notifications/unread-count/` | Yes | Get unread notification count (lightweight, used for polling) **(NEW)** |
| POST | `/api/notifications/mark-read/` | Yes | Mark specific notification IDs as read **(NEW)** |
| POST | `/api/notifications/mark-all-read/` | Yes | Mark all unread notifications as read **(NEW)** |

---

## Database Changes

| Table | Operation | Migration |
|-------|-----------|-----------|
| `accounts_post` | CREATE | 0002 |
| `accounts_comment` | CREATE | 0002 |
| `accounts_like` | CREATE (with unique_together) | 0002 |
| `accounts_post_tagged_users` | CREATE (M2M join table) | 0003 |
| `accounts_notification` | CREATE (with FKs to User, Post, Comment) | 0004 **(NEW)** |

---

## UI/UX Changes

| Area | Change |
|------|--------|
| **Home Feed** | Replaced static placeholder posts with real API-driven feed |
| **Home Feed** | Removed "create post" input bar from top of feed |
| **Home Feed** | Added loading and empty states |
| **Home Feed** | Added like toggling with visual feedback |
| **Home Feed** | Added inline comment system with edit/delete |
| **Profile Page** | Complete rewrite from static to dynamic |
| **Profile Page** | Added "Plant a Post" 3-step carousel modal |
| **Profile Page** | Added Instagram-style posts grid with hover overlays |
| **Profile Page** | Added "Tagged" posts tab |
| **Profile Page** | Added post detail modal with image, comments, likes |
| **Profile Page** | Added post deletion with confirmation dialog |
| **Profile Page** | Real stats (post count, likes received, leaves) |
| **Profile Page** | Real progress bars for "Credible User" promotion |
| **Profile Sidebar** | Shows user's actual bio and recent posts |
| **Left Nav** | Theme toggle button disabled (commented out) |
| **Left Nav** | Improved hover/active label styling |
| **Left Nav** | 🔔 Notification bell with red unread badge (pulsing animation) **(NEW)** |
| **Left Nav** | Notification dropdown panel with sender avatar, message, time ago **(NEW)** |
| **Left Nav** | Mark individual / all notifications as read **(NEW)** |
| **Left Nav** | Auto-polling every 10s — lightweight count when closed, full list when open **(NEW)** |
| **Left Nav** | Click-outside-to-close behavior for notification panel **(NEW)** |
| **Theme** | Default changed from dark to light |
| **Theme** | Migration logic to clear stale dark preferences |
| **All pages** | Hardcoded colors replaced with CSS custom properties |
| **All pages** | Smooth animations (modals, carousel, heart bounce, spinners) |
| **All pages** | Responsive design for mobile (768px breakpoint) |
