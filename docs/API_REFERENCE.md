# Arcadia — REST API Reference Documentation

[← Back to Showcase Overview](../README.md) • [Architecture](ARCHITECTURE.md) • [System Design](SYSTEM_DESIGN.md) • [Security Overview](SECURITY_OVERVIEW.md) • [Deployment Runbook](DEVELOPMENT_AND_DEPLOYMENT.md)

---

All application endpoints are versioned under the `/api/v1` prefix (with system liveness probes mounted at root `/health` and `/ready`). Unless otherwise specified, requests accepting a body require `Content-Type: application/json`.

All responses follow the standardized `ApiResponse<T>` envelope:
```json
{
  "success": true,
  "data": { ... }
}
```

---

## 1. Authentication Endpoints (`/api/v1/auth`)

### Register New User
- **Method:** `POST`
- **Route:** `/api/v1/auth/register`
- **Auth Required:** No
- **Rate Limit:** 15 requests per 15 minutes
- **Request Body:**
  ```json
  {
    "email": "user@example.com",
    "password": "Password123!",
    "displayName": "GamerTag"
  }
  ```
- **Response (201 Created):**
  Sets `arcadia_session` (HttpOnly, Secure) and `arcadia_csrf_token` cookies.
  ```json
  {
    "success": true,
    "data": {
      "user": {
        "id": "671df42a18b7625a6c11a001",
        "email": "user@example.com",
        "displayName": "GamerTag",
        "role": "user",
        "authProvider": "local",
        "createdAt": "2026-10-27T18:00:00.000Z"
      }
    }
  }
  ```

### User Login
- **Method:** `POST`
- **Route:** `/api/v1/auth/login`
- **Auth Required:** No
- **Rate Limit:** 15 requests per 15 minutes
- **Request Body:**
  ```json
  {
    "email": "user@example.com",
    "password": "Password123!"
  }
  ```
- **Response (200 OK):** Sets `arcadia_session` and `arcadia_csrf_token` cookies.

### User Logout
- **Method:** `POST`
- **Route:** `/api/v1/auth/logout`
- **Auth Required:** Yes (clears session cookies and atomically increments `tokenVersion` to invalidate active tokens)
- **Response (200 OK):**
  ```json
  {
    "success": true,
    "data": { "message": "Successfully logged out" }
  }
  ```

### Get Current Authenticated Session
- **Method:** `GET`
- **Route:** `/api/v1/auth/me`
- **Auth Required:** Yes (`arcadia_session` cookie or `Authorization: Bearer <token>`)
- **Response (200 OK):** Returns current user DTO (omits sensitive password hashes).

### Password Recovery Request
- **Method:** `POST`
- **Route:** `/api/v1/auth/forgot-password`
- **Auth Required:** No
- **Rate Limit:** 15 requests per 15 minutes
- **Request Body:** `{ "email": "user@example.com" }`
- **Response (200 OK):** Generates a time-bounded password reset token.

### Password Reset Execution
- **Method:** `POST`
- **Route:** `/api/v1/auth/reset-password`
- **Auth Required:** No
- **Rate Limit:** 15 requests per 15 minutes
- **Request Body:** `{ "token": "reset_token_hex", "password": "NewPassword123!" }`
- **Response (200 OK):** Updates password hash and increments `tokenVersion`.

> [!NOTE]
> Arcadia also includes optional Google OAuth endpoints (`GET /api/v1/auth/google`, `GET /api/v1/auth/google/url`, `GET /api/v1/auth/google/callback`, `POST /api/v1/auth/google`) active when Google Client ID and Secret credentials are provided in deployment settings.

---

## 2. User Profile Endpoints (`/api/v1/users`)

### Get User Profile
- **Method:** `GET`
- **Route:** `/api/v1/users/profile`
- **Auth Required:** Yes
- **Response (200 OK):** Current user profile with personal details.

### Update User Profile
- **Method:** `PATCH`
- **Route:** `/api/v1/users/profile`
- **Auth Required:** Yes
- **Headers:** `X-CSRF-Token: <token>`
- **Request Body:** `{ "displayName": "NewTag", "avatarUrl": "https://..." }`
- **Response (200 OK):** Updated user profile DTO.

---

## 3. Game Catalog Endpoints (`/api/v1/games`)

### Browse & Search Catalog
- **Method:** `GET`
- **Route:** `/api/v1/games`
- **Auth Required:** No
- **Query Parameters:**
  - `search` (string, optional): Keyword query.
  - `page` (number, default: `1`): Pagination index.
  - `limit` (number, default: `20`, max: `50`): Results per page.
  - `genres` (string, optional): Comma-separated genre slugs (e.g. `action,rpg`).
  - `platforms` (string, optional): Platform IDs.
  - `ordering` (string, default: `-added`): Sort ordering (`-metacritic`, `-released`, `-rating`).
- **Response (200 OK):** Paginated catalog results with metadata, ratings, and artwork.

### Featured Titles
- **Method:** `GET`
- **Route:** `/api/v1/games/featured`
- **Auth Required:** No
- **Response (200 OK):** Curated list of pinned titles and top-rated games.

### Game Details
- **Method:** `GET`
- **Route:** `/api/v1/games/:idOrSlug`
- **Auth Required:** No
- **Response (200 OK):** Full title metadata, system requirements, screenshots, and verified storefront links.

### Related Titles
- **Method:** `GET`
- **Route:** `/api/v1/games/:idOrSlug/related`
- **Auth Required:** No
- **Response (200 OK):** Contextually similar games based on shared genres and developer tags.

### Game Community Reviews
- **Method:** `GET`
- **Route:** `/api/v1/games/:gameId/reviews`
- **Auth Required:** No
- **Response (200 OK):** Community reviews for the specified title with author display names and ratings.

### Submit Game Review
- **Method:** `POST`
- **Route:** `/api/v1/games/:gameId/reviews`
- **Auth Required:** Yes
- **Headers:** `X-CSRF-Token: <token>`
- **Request Body:**
  ```json
  {
    "rating": 5,
    "headline": "Masterpiece of open-world design",
    "content": "Outstanding sound design, writing, and gameplay variety.",
    "isRecommended": true
  }
  ```
- **Response (201 Created):** Created review record.

---

## 4. Community Review Management (`/api/v1/reviews`)

### Update Own Review
- **Method:** `PUT`
- **Route:** `/api/v1/reviews/:reviewId`
- **Auth Required:** Yes (must be review author or administrator)
- **Headers:** `X-CSRF-Token: <token>`
- **Request Body:** `{ "rating": 4, "headline": "Updated Headline", "content": "Updated content..." }`
- **Response (200 OK):** Updated review object.

### Delete Review
- **Method:** `DELETE`
- **Route:** `/api/v1/reviews/:reviewId`
- **Auth Required:** Yes (must be review author or administrator)
- **Headers:** `X-CSRF-Token: <token>`
- **Response (200 OK):** Deletion confirmation.

---

## 5. User Collections: Library & Wishlist

### Get Personal Library
- **Method:** `GET`
- **Route:** `/api/v1/library`
- **Auth Required:** Yes
- **Response (200 OK):** Array of games tracked by the authenticated user with statuses: `playing`, `completed`, `backlog`, `dropped`.

### Check Library Status
- **Method:** `GET`
- **Route:** `/api/v1/library/check/:gameId`
- **Auth Required:** Yes
- **Response (200 OK):** Boolean indicator and status if the game exists in user's library.

### Add Game to Library
- **Method:** `POST`
- **Route:** `/api/v1/library/:gameId`
- **Auth Required:** Yes
- **Headers:** `X-CSRF-Token: <token>`
- **Request Body:**
  ```json
  {
    "gameSlug": "grand-theft-auto-v",
    "gameName": "Grand Theft Auto V",
    "gameCover": "https://media.rawg.io/...",
    "platform": "PC",
    "status": "playing"
  }
  ```
- **Response (201 Created):** Created library entry.

### Update Library Entry
- **Method:** `PATCH`
- **Route:** `/api/v1/library/:gameId`
- **Auth Required:** Yes
- **Headers:** `X-CSRF-Token: <token>`
- **Request Body:** `{ "status": "completed", "platform": "PC" }`
- **Response (200 OK):** Updated library entry.

### Remove from Library
- **Method:** `DELETE`
- **Route:** `/api/v1/library/:gameId`
- **Auth Required:** Yes
- **Headers:** `X-CSRF-Token: <token>`
- **Response (200 OK):** Deletion confirmation.

### Get Wishlist
- **Method:** `GET`
- **Route:** `/api/v1/wishlist`
- **Auth Required:** Yes
- **Response (200 OK):** List of bookmarked titles.

### Check Wishlist Status
- **Method:** `GET`
- **Route:** `/api/v1/wishlist/check/:gameId`
- **Auth Required:** Yes
- **Response (200 OK):** Boolean indicator if the game exists in user's wishlist.

### Add to Wishlist
- **Method:** `POST`
- **Route:** `/api/v1/wishlist/:gameId`
- **Auth Required:** Yes
- **Headers:** `X-CSRF-Token: <token>`
- **Response (201 Created):** Wishlist confirmation.

### Remove from Wishlist
- **Method:** `DELETE`
- **Route:** `/api/v1/wishlist/:gameId`
- **Auth Required:** Yes
- **Headers:** `X-CSRF-Token: <token>`
- **Response (200 OK):** Removal confirmation.

---

## 6. Administrative Control Plane (`/api/v1/admin`)

*All administrative routes require an active session belonging to a user with `role: "admin"`.*

### Verify Admin Privileges
- **Method:** `GET`
- **Route:** `/api/v1/admin/ping`
- **Auth Required:** Yes (`admin` role)
- **Response (200 OK):** `{ "success": true, "data": { "message": "Administrator privileges verified" } }`

### Operational Overview & Telemetry
- **Method:** `GET`
- **Route:** `/api/v1/admin/overview`
- **Auth Required:** Yes (`admin` role)
- **Response (200 OK):** System metrics, total users, cached titles count, and server uptime.

### List Users with Moderation Status
- **Method:** `GET`
- **Route:** `/api/v1/admin/users`
- **Auth Required:** Yes (`admin` role)
- **Query Parameters:** `page`, `limit`, `search`, `role`, `isBanned`.
- **Response (200 OK):** Paginated user list with moderation controls.

### Update User Role
- **Method:** `PATCH`
- **Route:** `/api/v1/admin/users/:userId/role`
- **Auth Required:** Yes (`admin` role)
- **Headers:** `X-CSRF-Token: <token>`
- **Request Body:** `{ "role": "admin" }` or `{ "role": "user" }`
- **Security Guard:** Sole remaining administrator cannot demote their own account.

### Update User Ban Status
- **Method:** `PATCH`
- **Route:** `/api/v1/admin/users/:userId/status`
- **Auth Required:** Yes (`admin` role)
- **Headers:** `X-CSRF-Token: <token>`
- **Request Body:** `{ "isBanned": true }`
- **Security Guard:** Atomically increments target user's `tokenVersion`, revoking all active sessions across devices.

### Catalog Management
- **Method:** `GET`
- **Route:** `/api/v1/admin/catalog`
- **Auth Required:** Yes (`admin` role)
- **Response (200 OK):** Internal catalog cache view.

### Toggle Featured Game Status
- **Method:** `PATCH`
- **Route:** `/api/v1/admin/catalog/:gameId/featured`
- **Auth Required:** Yes (`admin` role)
- **Headers:** `X-CSRF-Token: <token>`
- **Request Body:** `{ "isFeatured": true }`
- **Response (200 OK):** Updates featured catalog curation.

### Flush Catalog Cache
- **Method:** `POST`
- **Route:** `/api/v1/admin/catalog/flush`
- **Auth Required:** Yes (`admin` role)
- **Headers:** `X-CSRF-Token: <token>`
- **Response (200 OK):** Clears expired RAWG cached responses from MongoDB and records an audit log event.

### Admin Review Moderation
- **Method:** `GET`
- **Route:** `/api/v1/admin/reviews`
- **Auth Required:** Yes (`admin` role)
- **Response (200 OK):** List of all reviews across the platform for moderation review.

### Admin Review Deletion
- **Method:** `DELETE`
- **Route:** `/api/v1/admin/reviews/:reviewId`
- **Auth Required:** Yes (`admin` role)
- **Headers:** `X-CSRF-Token: <token>`
- **Response (200 OK):** Moderation deletion of offensive or violating reviews.

### Retrieve Audit Logs
- **Method:** `GET`
- **Route:** `/api/v1/admin/audit-logs`
- **Auth Required:** Yes (`admin` role)
- **Response (200 OK):** Chronological, immutable log of operational actions.

---

## 7. System Health & Probes

### Liveness Probe
- **Method:** `GET`
- **Route:** `/health`
- **Auth Required:** No
- **Response (200 OK):** `{ "status": "ok", "timestamp": "2026-10-27T18:00:00.000Z" }`

### Readiness Probe
- **Method:** `GET`
- **Route:** `/ready`
- **Auth Required:** No
- **Response (200 OK):** `{ "status": "ready", "database": "connected", "uptime": 1245.2 }` (Returns HTTP 503 if MongoDB connection is down).

---

[← Back to Showcase Overview](../README.md) • [Architecture](ARCHITECTURE.md) • [System Design](SYSTEM_DESIGN.md) • [Security Overview](SECURITY_OVERVIEW.md) • [Deployment Runbook](DEVELOPMENT_AND_DEPLOYMENT.md)
