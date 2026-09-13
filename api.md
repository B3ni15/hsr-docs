# HSR API

This is the backend API reference for the HSR web application. The API is implemented with Next.js App Router route handlers and returns JSON unless a route explicitly documents a redirect or a file response.

## Base URL

Replace `https://hsr.example.com` in the examples with the deployed HSR URL.

```bash
export HSR_URL="https://hsr.example.com"
```

## Authentication

### Browser session

Web users authenticate through the browser. The server stores the session in the httpOnly `auth-token` cookie.

### Bearer token

External clients may send a mobile access token on approved API prefixes.

```bash
export ACCESS_TOKEN="<access-token>"
curl -sS \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/auth/session"
```

The session response contains the current user, Discord roles, permission flags, suspensions, locale, maintenance state, and exam lock state. An invalid or expired token returns HTTP `401`.

## Response conventions

Successful JSON responses use HTTP `200`, unless an endpoint documents a redirect or another status. Errors use this shape:

```json
{ "error": "Human-readable error message" }
```

| Status | Meaning |
|---:|---|
| `200` | Request succeeded |
| `400` | Missing or invalid input |
| `401` | No valid session or bearer token |
| `403` | Authenticated, but not allowed or banned |
| `404` | Resource does not exist |
| `409` | Conflict with current state |
| `429` | Rate limit exceeded |
| `500` | Unexpected server-side failure |

## Route groups

The route names below are real route prefixes. A parameter is written as `[id]` in the source and as a concrete value in a request. Every one of the 336 route files under `src/app/api/` (456 method handlers total) is documented individually, with its own `curl` example, in [Full Endpoint Reference](#full-endpoint-reference) below.

### Authentication

- `/api/auth/login`, `/api/auth/logout`, `/api/auth/session`
- `/api/auth/discord/*`
- `/api/auth/roblox/link`, `/api/auth/callback`
- `/api/auth/spotify/*`
- `/api/auth/mobile/*` - see [Mobile API](mobile-api.md)
- `/api/auth/game/*` - game-client OAuth

### Member and public data

- `/api/profile/*`, `/api/profiles/[handle]/*`
- `/api/user/*`, `/api/sessions/*`
- `/api/leaderboard/*`, `/api/news`, `/api/changelog`
- `/api/wiki/*`, `/api/schedule/*`

Example public news request:

```bash
curl -sS "$HSR_URL/api/news?limit=6"
```

Response:

```json
{
  "items": [
    {
      "id": "discord-message-id",
      "title": "News title",
      "summary": "News summary",
      "publishedAt": "2026-09-13T10:00:00.000Z"
    }
  ]
}
```

`limit` is optional and is clamped to `1..20`. A server error returns HTTP `500` with `{ "error": "Failed to fetch news" }`.

### Duty and railway operations

- `/api/duty/*`
- `/api/schedule-request/*`
- `/api/dispatch-orders/*`
- `/api/route-permit/*`
- `/api/map/*`
- `/api/cargo/*`
- Train administration is handled under `/api/admin/trains`.

Example authenticated request:

```bash
curl -sS \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/duty/active?type=TRAIN_DRIVER"
```

The successful response is an array of active duty objects containing the duty member, start location, assigned trains, and station data. `type` is optional; valid values include `TRAIN_DRIVER` and `TRAIN_DISPATCHER`.

### Training, exams, and supervision

- `/api/training/*`
- `/api/exams/*`
- `/api/exam-slots/*`
- `/api/closing-exam-vote/[token]`
- `/api/supervisor-request/*`

These endpoints enforce training status, exam eligibility, permissions, and attempt state. Use the response's `error` field together with its HTTP status when displaying a refusal.

### Tickets, documents, and shop

- `/api/tickets/*`
- `/api/documents/*`
- `/api/shop/*`
- `/api/printful/*`
- `/api/verify/*`

### Admin and scheduled jobs

- `/api/admin/*` - permission-gated admin operations
- `/api/cron/*` - server-to-server scheduled jobs
- `/api/maintenance-status`

Admin APIs require a valid session and the permission appropriate to the operation. Cron endpoints are protected server-to-server endpoints, not general client APIs.

## Public routes

Common public endpoints include `/`, `/login`, `/wiki`, `/timetable`, `/support`, `/api/news`, `/api/changelog`, `/api/auth/mobile/*`, `/api/game/*`, and `/api/cron/*`.

## Security behavior

The middleware applies public-route allowlisting, incomplete-account enforcement, cookie and bearer authentication, and security headers. A route being publicly reachable does not mean every operation on it is anonymous; individual handlers still validate tokens, ownership, permissions, signatures, or request-specific tokens.

## Full Endpoint Reference

Every endpoint below is grouped by area, in the same order the sections above summarize them. Mobile OAuth, Live Activity, and push-token endpoints are documented in full in [Mobile API](mobile-api.md#full-endpoint-reference) instead of being repeated here.

- [Authentication (Web & Game Client)](#authentication-web--game-client)
- [Admin: Badges, Blacklist, Cargo, Diagnostics, Discord, Dispatch Orders, Documents](#admin-badges-blacklist-cargo-diagnostics-discord-dispatch-orders-documents)
- [Admin: Duty Management](#admin-duty-management)
- [Admin: Email, Exam Slots, Exams](#admin-email-exam-slots-exams)
- [Admin: Game Stats, Instructors, Locomotive Pool, Maintenance, Perf Stats, Permissions](#admin-game-stats-instructors-locomotive-pool-maintenance-perf-stats-permissions)
- [Admin: Schedule Requests, Shop, Stations, Stops](#admin-schedule-requests-shop-stations-stops)
- [Admin: Statistics, Supervision Rules, Terms, Discord Tickets, Track Map](#admin-statistics-supervision-rules-terms-discord-tickets-track-map)
- [Admin: Training Programmes & Content, Trains](#admin-training-programmes--content-trains)
- [Admin: Type Trainings, Users, Wagon Pool, Wrapped](#admin-type-trainings-users-wagon-pool-wrapped)
- [Cron Jobs](#cron-jobs)
- [Cargo, Changelog, Closing-Exam Vote, Dispatch Orders, Documents, Duty](#cargo-changelog-closing-exam-vote-dispatch-orders-documents-duty)
- [Exam Slots, Exams, Game Client, Leaderboard, Map, News, Payment Webhooks, Profiles](#exam-slots-exams-game-client-leaderboard-map-news-payment-webhooks-profiles)
- [Route Permits, Schedule, Search, Sessions, Shop, Stripe](#route-permits-schedule-search-sessions-shop-stripe)
- [Supervision, Supervisor Requests, Tickets, Training, Type Trainings, User, Verify, Wiki](#supervision-supervisor-requests-tickets-training-type-trainings-user-verify-wiki)

## Authentication (Web & Game Client)

### `GET /api/auth/discord`

**Why:** Entry point to attach a Discord account to a session that already exists — the second half of the "Roblox login is still reachable" path described in CLAUDE.md, for a member who signed in with Roblox first and now needs to link Discord (which is required for every protected route).

**Auth:** Requires an existing session (`auth-token` cookie, read via `getSession()`). No session → redirect to login.

**Request:** No query params or body.

```bash
curl -sS -i \
  --cookie "auth-token=<jwt>" \
  "$HSR_URL/api/auth/discord"
```

**Response:** Always a redirect (302), no JSON body.
- `302` → `/login?error=not_logged_in` if there is no session
- `302` → `/dashboard` if `session.discordId` is already set
- `302` → `https://discord.com/api/oauth2/authorize?...` (scope `identify email guilds.join`, `prompt=consent`) otherwise. Sets cookie `oauth-state` (httpOnly, `secure` in production, `sameSite=lax`, `maxAge=600`, `path=/`) before redirecting.

---

### `GET /api/auth/logout`

**Why:** Standard sign-out link target (e.g. a plain `<a href>`); revokes the tracked `WebSession` row so the session can't be replayed even if the cookie leaks, then clears the cookie.

**Auth:** None required — works whether or not a session is present (no-ops if there's no `webSessionId`).

**Request:** No params.

```bash
curl -sS -i --cookie "auth-token=<jwt>" "$HSR_URL/api/auth/logout"
```

**Response:** `302` → `/login`. Deletes the `auth-token` cookie. If a `webSessionId` was embedded in the JWT, the corresponding `WebSession.revokedAt` is stamped.

---

### `POST /api/auth/logout`

**Why:** JSON variant of the same sign-out, for client-side fetch-based logout (e.g. a logout button doing `fetch` instead of navigating).

**Auth:** None required.

**Request:** No body.

```bash
curl -sS -X POST --cookie "auth-token=<jwt>" "$HSR_URL/api/auth/logout"
```

**Response:**
```json
{ "success": true }
```
Also clears the `auth-token` cookie and revokes the `WebSession` row, same as the GET variant.

---

### `GET /api/auth/callback`

**Why:** The single registered Roblox OAuth redirect URI. Roblox is the identity provider for both a fresh Roblox-first sign-in and for linking Roblox onto an already-Discord-first account, so this route reads the `roblox-oauth-mode` cookie (`login` vs `link`) to decide which of the two flows to run — the intent can't ride in `state` because that slot is already used for CSRF.

**Auth:** Public path (no cookie required to reach it — Roblox redirects here with no user context). Internally: for `link` mode it requires an existing session (`getSession()`); for `login` mode it creates/looks up the user itself. Rate limited by IP (`authLimiter`).

**Request:** Query params `code` (string, required — the OAuth code), `error` (string, optional), `state` (string, required, checked against the `oauth-state` cookie via `validateOAuthState`).

```bash
curl -sS -i "$HSR_URL/api/auth/callback?code=<code>&state=<state>"
```

**Response:** Always a redirect, no JSON.
- `302` → `/login?error=<error>` if Roblox returned an `error`
- `302` → `/login?error=no_code` if no `code`
- `302` → `/login?error=invalid_state` if the state doesn't match
- **Link mode** (`roblox-oauth-mode=link` cookie): on success, merges/attaches the Roblox identity via `linkRobloxAccount()` and issues a fresh `auth-token` (a merge deletes the shell's session and creates a new `WebSession`); redirects to `/mobile-callback` or `/game-callback` if those OAuth contexts are pending, otherwise to `returnUrl` or `/dashboard`. Failure paths: `302` → `/link-roblox?error=roblox_already_linked` (conflict), `302` → `/login?error=banned&reason=...`, `302` → `/link-roblox?error=link_failed`.
- **Login mode**: upserts a `User` row by `robloxId`, refreshes Discord roles if linked, writes a `LOGIN` `ActivityLog` entry, sets `auth-token` (httpOnly, `secure` in prod, `sameSite=lax`, `maxAge=30d`), and redirects — to the mobile/game callback if a matching OAuth cookie is present and fresh (<10 min), to `/confirm-account` if the account has game history and is unconfirmed, to `/link-discord` if `discordId` is null, otherwise to `returnUrl` or `/dashboard`. Banned users get `302` → `/login?error=banned&reason=...`. Any thrown error → `302` → `/login?error=auth_failed`.

---

### `GET /api/auth/login`

**Why:** The quiet secondary Roblox sign-in entry point mentioned in CLAUDE.md — kept reachable because pre-Discord-first accounts (including ones the game server auto-created) are keyed by Roblox and would otherwise have no way in.

**Auth:** Public.

**Request:** No params.

```bash
curl -sS -i "$HSR_URL/api/auth/login"
```

**Response:** `302` → `https://apis.roblox.com/oauth/v1/authorize?...` (mode defaults to `login`). Sets `oauth-state` and `roblox-oauth-mode=login` cookies (httpOnly, `secure` in prod, `sameSite=lax`, `maxAge=600`).

---

### `GET /api/auth/session`

**Why:** The one endpoint the client (web and mobile) polls to learn who it is and what it may do — it assembles `roleContext` (priority/category/anchors/programs) because the browser has no database of its own to resolve the Discord rank hierarchy or training programmes against.

**Auth:** `auth-token` cookie **or** `Authorization: Bearer <jwt>` header (`getSessionWithBearer()` — mobile uses the bearer form).

**Request:** No params.

```bash
curl -sS \
  --cookie "auth-token=<jwt>" \
  "$HSR_URL/api/auth/session"
```

**Response:**
```json
{
  "user": {
    "id": "clx...",
    "robloxId": "123456",
    "name": "SomeUser",
    "discordId": "9876...",
    "discordRole": "・Commanding Officer",
    "discordRoles": ["role_id_1", "role_id_2"],
    "roleContext": { "priority": 42, "category": "operations", "anchors": {}, "programs": [] },
    "role": "USER",
    "banned": false,
    "permissionFlags": { "CAN_DRIVE": true, "MANAGE_TICKETS": false },
    "suspensions": [],
    "totalDonated": 0,
    "locale": "hu",
    "maintenance": { "enabled": false, "message": null },
    "examLock": null
  },
  "isDiscordLinked": true
}
```
- `401 { "user": null }` if there is no valid session, or if the underlying `User` row can no longer be found.

---

### `GET /api/auth/discord/callback`

**Why:** Completes the Discord-linking half of an account — either the ordinary "Discord-first, link Roblox later" flow's Discord *re*-link, or (more commonly per CLAUDE.md) attaching Discord to a Roblox-first account via `/link-discord`. Also folds away a Discord-only shell left behind by `absorbDiscordOnlyAccount()` if the Discord id is already claimed by an abandoned sign-up.

**Auth:** Requires an existing session (`getSession()`); Discord redirects here with no cookie context of its own beyond what the browser already carries, so this is effectively "must already be logged in." Rate limited by IP.

**Request:** Query params `code` (required), `error` (optional), `state` (required, checked against `oauth-state` cookie).

```bash
curl -sS -i "$HSR_URL/api/auth/discord/callback?code=<code>&state=<state>"
```

**Response:** Redirect only.
- `302` → `/link-discord?error=<error>` on an OAuth error
- `302` → `/link-discord?error=no_code`
- `302` → `/login?error=invalid_state`
- `302` → `/login?error=not_logged_in` if no session
- `302` → `/link-discord?error=email_not_shared` if Discord didn't return an email (also notifies via `notifyDiscordEmailIssue`)
- `302` → `/link-discord?error=discord_already_linked` if the Discord id belongs to a different, non-shell account
- On success: updates the `User` row with Discord identity fields, best-effort joins the guild (`addGuildMember`), force-refreshes Discord roles, writes a `DISCORD_LINK` `ActivityLog` entry, re-signs the `auth-token` cookie (preserving the existing `webSessionId`), then redirects to `returnUrl` or `/dashboard`
- `302` → `/link-discord?error=link_failed` on any other thrown error

---

### `GET /api/auth/discord/login`

**Why:** The primary login button on `/login` — Discord-first sign-in per CLAUDE.md's rewritten auth flow. Starts the OAuth handshake that `discord/login/callback` finishes.

**Auth:** Public.

**Request:** No params.

```bash
curl -sS -i "$HSR_URL/api/auth/discord/login"
```

**Response:** `302` → `https://discord.com/api/oauth2/authorize?...` (redirect_uri `.../api/auth/discord/login/callback`, scope `identify email guilds.join`). Sets `oauth-state` cookie (httpOnly, `secure` in prod, `sameSite=lax`, `maxAge=600`).

---

### `GET /api/auth/roblox/link`

**Why:** Starts "attach Roblox to the account I'm already signed into" — the second step of Discord-first sign-up, reached from `/link-roblox`. Distinguishes itself from `/api/auth/login` only by requiring a session and marking the callback as a link via `roblox-oauth-mode`.

**Auth:** Requires an existing session (`getSession()`).

**Request:** No params.

```bash
curl -sS -i --cookie "auth-token=<jwt>" "$HSR_URL/api/auth/roblox/link"
```

**Response:**
- `302` → `/login?error=not_logged_in` if no session
- `302` → `/dashboard` if the session already has a `robloxId`
- `302` → `https://apis.roblox.com/oauth/v1/authorize?...` otherwise, with `roblox-oauth-mode=link` cookie set (so `/api/auth/callback` runs `handleRobloxLink`)

---

### `GET /api/auth/spotify/link`

**Why:** Starts linking a member's Spotify account for the profile "now playing" card and HSR Wrapped's music section (see the Spotify Link section of CLAUDE.md). `/api/auth` is a public middleware path since OAuth callbacks arrive cookie-less from the provider, so this route checks the session itself.

**Auth:** Requires an existing session (`getSession()`); otherwise redirected to log in.

**Request:** No params.

```bash
curl -sS -i --cookie "auth-token=<jwt>" "$HSR_URL/api/auth/spotify/link"
```

**Response:**
- `302` → `/login?returnUrl=/profile/me/edit` if not logged in
- `302` → `/profile/me/edit?spotify=unavailable` if `SPOTIFY_CLIENT_ID`/`SPOTIFY_CLIENT_SECRET` aren't configured
- `302` → Spotify's authorize URL otherwise, with `spotify-oauth-state` cookie set (httpOnly, `secure` in prod, `sameSite=lax`, `maxAge=600`) — a CSRF guard, since without it anyone could hand a member a callback URL that links *their* Spotify account onto the member's profile

---

### `GET /api/auth/spotify/callback`

**Why:** Finishes the Spotify link started above, storing only a refresh token against the member's `UserProfile` (access tokens are minted on demand per CLAUDE.md's Spotify Link section).

**Auth:** Requires an existing session (`getSession()`).

**Request:** Query params `state` (required, compared to the `spotify-oauth-state` cookie), `code` (required), `error` (optional, set when the member cancels Spotify's consent screen).

```bash
curl -sS -i --cookie "auth-token=<jwt>" \
  "$HSR_URL/api/auth/spotify/callback?code=<code>&state=<state>"
```

**Response:**
- `302` → `/profile/me/edit?spotify=unavailable` if Spotify isn't configured
- `302` → `/profile/me/edit?spotify=cancelled` if `error` is set
- `302` → `/profile/me/edit?spotify=state` if `code`/`state` are missing or mismatched
- `302` → `/profile/me/edit?spotify=notallowed` or `?spotify=failed` if the token exchange fails
- `302` → `/profile/me/edit?spotify=linked` on success — steals the `spotifyId` from any other profile that held it, updates `UserProfile` with `spotifyId`/`spotifyRefreshToken`/`spotifyDisplayName`/`spotifyLinkedAt`, and writes a `SPOTIFY_LINKED` `ActivityLog` entry

---

### `POST /api/auth/game/token`

**Why:** Step 3 of the Game Client OAuth flow in CLAUDE.md — the game exchanges the one-time authorization code (minted by `/api/auth/game/complete`) for a bearer JWT + refresh token it can use against the game API.

**Auth:** Public (no session — the code itself is the credential). Rate limited by IP.

**Request:** JSON body: `code` (string, required), `state` (string, optional — checked against the code's stored `state` if present).

```bash
curl -sS -X POST \
  -H "Content-Type: application/json" \
  -d '{"code":"<authcode>","state":"<state>"}' \
  "$HSR_URL/api/auth/game/token"
```

**Response:**
```json
{
  "access_token": "<jwt>",
  "refresh_token": "<hex>",
  "token_type": "Bearer",
  "expires_in": 604800,
  "user": { "id": "clx...", "robloxId": "123456", "name": "SomeUser", "banned": false, "sessionVersion": 1 }
}
```
- `400 { "error": "Authorization code is required" }`
- `400 { "error": "Invalid or expired authorization code" }` (code missing, already used, or expired — checked via an atomic `updateMany`)
- `400 { "error": "State mismatch" }`
- `404 { "error": "User not found" }`
- `403 { "error": "User account is banned" }`
- `500 { "error": "Failed to exchange authorization code" }`

Creates a `GameClientSession` row (90-day expiry) and a `GAME_CLIENT_LOGIN` `ActivityLog` entry.

---

### `POST /api/auth/game/complete`

**Why:** Step 2 of the Game Client OAuth flow — once the player has logged into the web app (having been sent there by `/api/auth/game/authorize`), this mints the one-time authorization code the game exchanges for tokens, and optionally attaches a verified Steam id from the earlier `steam_ticket` check.

**Auth:** Requires an existing web session (`getSession()`), i.e. the player must have just logged in on the web.

**Request:** No body; reads context from the `game-oauth` cookie set by `/api/auth/game/authorize`.

```bash
curl -sS -X POST --cookie "auth-token=<jwt>; game-oauth=<cookie>" \
  "$HSR_URL/api/auth/game/complete"
```

**Response:**
```json
{ "redirectUrl": "hsr://auth/callback?code=<authcode>&state=<state>" }
```
- `401 { "error": "Not authenticated" }`
- `400 { "error": "No game OAuth context found" }`
- `400 { "error": "Invalid game OAuth context" }` (cookie doesn't parse as JSON — the cookie is also deleted)
- `400 { "error": "Game OAuth context expired" }` (older than 10 minutes — cookie deleted)
- `409 { "error": "This Steam account is already linked to another user" }`
- `500 { "error": "Failed to complete game authentication" }`

On success: links `steamId` to the user if provided and free, creates a `GameAuthCode` row (5-minute expiry), writes a `GAME_CLIENT_LOGIN` `ActivityLog` entry, and clears the `game-oauth` cookie.

---

### `POST /api/auth/game/revoke`

**Why:** Lets the game client (or a user managing their sessions) invalidate a `GameClientSession` — either one specific device or every device — the game-OAuth equivalent of web session revocation.

**Auth:** Bearer token or `auth-token` cookie (`getSessionWithBearer()`).

**Request:** JSON body: `refresh_token` (string, optional — revokes just that session; omit to revoke all of the caller's game sessions).

```bash
curl -sS -X POST \
  -H "Authorization: Bearer <jwt>" \
  -H "Content-Type: application/json" \
  -d '{"refresh_token":"<hex>"}' \
  "$HSR_URL/api/auth/game/revoke"
```

**Response:**
```json
{ "success": true }
```
- `401 { "error": "Unauthorized" }`
- `400 { "error": "Invalid refresh token" }` (when a `refresh_token` was passed but doesn't match a session)
- `403 { "error": "Forbidden" }` (the token belongs to a different user)
- `500 { "error": "Failed to revoke session" }`

---

### `POST /api/auth/game/refresh`

**Why:** Step "Tokens refreshed via `/api/auth/game/refresh`" from CLAUDE.md's Game Client OAuth flow — exchanges a still-valid, unrevoked refresh token for a new short-lived access token without re-running the whole OAuth dance.

**Auth:** Public (the refresh token itself is the credential). Rate limited by IP.

**Request:** JSON body: `refresh_token` (string, required).

```bash
curl -sS -X POST \
  -H "Content-Type: application/json" \
  -d '{"refresh_token":"<hex>"}' \
  "$HSR_URL/api/auth/game/refresh"
```

**Response:**
```json
{
  "access_token": "<jwt>",
  "refresh_token": "<hex>",
  "token_type": "Bearer",
  "expires_in": 604800,
  "user": { "id": "clx...", "robloxId": "123456", "name": "SomeUser", "banned": false, "sessionVersion": 1 }
}
```
- `400 { "error": "refresh_token is required" }`
- `401 { "error": "Invalid refresh token" }`
- `401 { "error": "Session has been revoked" }`
- `401 { "error": "Refresh token expired" }`
- `403 { "error": "User account is banned" }`
- `500 { "error": "Failed to refresh token" }`

Updates `GameClientSession.lastUsedAt` on success; the refresh token itself is not rotated.

---

### `GET /api/auth/game/authorize`

**Why:** Step 1 of the Game Client OAuth flow — the Roblox game client calls this to receive an OAuth URL/redirect it can send the player through in an embedded browser, optionally pre-verifying the player's Steam identity via a session ticket before the player ever logs in on the web.

**Auth:** Public.

**Request:** Query params: `redirect_uri` (string, required — must be `hsr://...` or `http(s)://localhost:29100-29200`), `state` (string, required), `steam_ticket` (string, optional, hex-encoded Steam session ticket), `platform` (string, optional).

```bash
curl -sS -i \
  "$HSR_URL/api/auth/game/authorize?redirect_uri=hsr://auth/callback&state=<state>&steam_ticket=<hex>"
```

**Response:**
- `400 { "error": "redirect_uri is required" }`
- `400 { "error": "Invalid redirect_uri. Must use hsr:// scheme or localhost:29100-29200" }`
- `400 { "error": "state is required" }`
- `302` → `/login?game=true` on success. Sets `game-oauth` cookie (JSON: `redirectUri`, `state`, `steamId`, `platform`, `createdAt`; httpOnly, `secure` in prod, `sameSite=lax`, `maxAge=600`). If `steam_ticket` was supplied and Steam validation is configured (`STEAM_WEB_API_KEY`/`STEAM_APP_ID`), the ticket is validated first and, if valid, `steamId` in the cookie is the *verified* id; an invalid/unconfigured check silently leaves `steamId` null rather than failing the request.

---

### `GET /api/auth/discord/login/callback`

**Why:** Finishes Discord-first sign-in/sign-up — the core of the rewritten auth flow in CLAUDE.md. An unrecognized Discord id is treated as a new sign-up (creates a bare `User` with no `robloxId`) rather than an error; a recognized one is an ordinary login. Also heals accounts that still carry the old Discord-username stand-in in their `name`/`displayName` columns.

**Auth:** Public (Discord redirects here with no session). Rate limited by IP.

**Request:** Query params `code` (required), `error` (optional), `state` (required, checked against `oauth-state` cookie).

```bash
curl -sS -i "$HSR_URL/api/auth/discord/login/callback?code=<code>&state=<state>"
```

**Response:** Redirect only.
- `302` → `/login?error=<error>` on an OAuth error
- `302` → `/login?error=no_code`
- `302` → `/login?error=invalid_state`
- `302` → `/login?error=email_not_shared` if this is a new sign-up and Discord didn't share an email (the whole app assumes one)
- On success for a **new account**: creates a `User` (no `robloxId`, `accountConfirmed: true` since there's no game history to confirm), writes a `REGISTER` `ActivityLog` entry
- On success for an **existing account**: clears the legacy Discord-name stand-in from `name`/`displayName` if still present, updates Discord fields, writes a `LOGIN` `ActivityLog` entry
- Either way: best-effort joins the guild (`addGuildMember`), syncs Discord roles, sets the `auth-token` cookie, and redirects — to `/mobile-callback`/`/game-callback` only if the account already has a `robloxId` (an account without one is not usable by those clients yet), to `/link-roblox` if `robloxId` is still null, otherwise to `returnUrl` or `/dashboard`
- `302` → `/login?error=auth_failed` on any other thrown error

## Admin: Badges, Blacklist, Cargo, Diagnostics, Discord, Dispatch Orders, Documents

### `GET /api/admin/badges`

**Why:** Renders the badge catalogue for `/admin/badges`. Per the Badges section, a badge is a row (not a hardcoded constant), so this also seeds any missing built-in badges into an empty table before listing — a no-op once a single badge exists.

**Auth:** `MANAGE_BADGES` (`requirePermission((p) => p.canManageBadges())`).

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/badges"
```

**Response:**
```json
{
  "badges": [
    {
      "id": "clx...",
      "key": "hundred_hours",
      "name": "Száz óra",
      "nameEn": "Hundred Hours",
      "emoji": "🏅",
      "rarity": "RARE",
      "awardMode": "AUTOMATIC",
      "ruleKey": "TOTAL_HOURS",
      "ruleThreshold": 100,
      "hidden": false,
      "enabled": true,
      "builtIn": true,
      "holders": 42
    }
  ],
  "ruleKeys": [{ "key": "TOTAL_HOURS", "label": "Total hours" }]
}
```

---

### `POST /api/admin/badges`

**Why:** Creates a new badge in the admin-managed catalogue (Badges section — "a badge is a row, not a constant").

**Auth:** `MANAGE_BADGES`.

**Request:** JSON body: `name` (string, required), `key` (string, optional — auto-derived from `name` if omitted; must be unique), `nameEn`, `description`, `descriptionEn` (optional strings), `emoji` (optional, ≤8 chars), `color` (optional), `rarity` (optional, one of `COMMON|RARE|EPIC|LEGENDARY`, default `COMMON`), `category` (optional), `awardMode` (optional, `MANUAL|AUTOMATIC`, default `MANUAL`), `ruleKey` (required if `awardMode: "AUTOMATIC"`, must be a valid badge rule key), `ruleThreshold` (number, default 1), `hidden` (boolean), `enabled` (boolean, default true), `sortOrder` (number).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"name":"Ezer kilométer","awardMode":"AUTOMATIC","ruleKey":"TOTAL_DISTANCE","ruleThreshold":1000}' \
  "$HSR_URL/api/admin/badges"
```

**Response:** `{ "badge": { "id": "...", "key": "ezer_kilometer", "name": "Ezer kilométer", ... } }`. Errors: 400 (missing name / could not derive key / automatic badge with no rule), 409 (key already exists).

---

### `POST /api/admin/badges/seed`

**Why:** Manual trigger to insert any built-in badge the catalogue is missing, without touching badges staff already edited — lets a built-in added in a later release be pulled in with no deploy-time hook (Badges section).

**Auth:** `MANAGE_BADGES`.

**Request:** No body.

```bash
curl -sS -X POST -b cookies.txt "$HSR_URL/api/admin/badges/seed"
```

**Response:** `{ "ok": true, "added": 2 }` (count of newly inserted built-ins).

---

### `PATCH /api/admin/badges/[id]`

**Why:** Edits an existing badge — re-pricing a threshold, renaming, toggling visibility, switching award mode.

**Auth:** `MANAGE_BADGES`.

**Request:** JSON body, all fields optional and only applied if present: `name` (non-empty if given), `nameEn` (≤120), `description`/`descriptionEn` (≤2000), `emoji` (≤8), `color` (≤20), `category` (≤60), `rarity` (one of the enum), `hidden` (boolean), `enabled` (boolean), `sortOrder` (number), `awardMode` (`MANUAL|AUTOMATIC`), `ruleKey` (valid rule key or null), `ruleThreshold` (number or null). Refused if the resulting state is `AUTOMATIC` with no `ruleKey`.

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"ruleThreshold":150,"enabled":false}' \
  "$HSR_URL/api/admin/badges/clx123"
```

**Response:** `{ "badge": { ...updated fields } }`. Errors: 404 (not found), 400 (empty name / automatic with no rule).

---

### `DELETE /api/admin/badges/[id]`

**Why:** Removes a badge from the catalogue. Built-ins cannot be deleted — their key is named by the seed and would come back on the next empty-table seed, silently redirecting an automatic rule onto a different row — so switching one off is the actual "we don't use this any more" edit (Badges section).

**Auth:** `MANAGE_BADGES`.

**Request:** No body.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/badges/clx123"
```

**Response:** `{ "ok": true }`. Errors: 404 (not found), 400 (`"Built-in badges cannot be deleted — switch it off instead."`).

---

### `POST /api/admin/badges/[id]/image`

**Why:** Uploads the badge's artwork to Cloudflare R2. Kept small (2MB cap) since a badge is drawn at 48px on a profile and in emails; the emoji stays as fallback for badges with no image.

**Auth:** `MANAGE_BADGES`.

**Request:** `multipart/form-data` with field `file` — image, one of `image/png`, `image/webp`, `image/jpeg`, `image/gif`, `image/svg+xml`, max 2MB. 503 if R2 is not configured.

```bash
curl -sS -X POST -b cookies.txt -F "file=@badge-art.png;type=image/png" \
  "$HSR_URL/api/admin/badges/clx123/image"
```

**Response:** `{ "imageUrl": "https://s3.hsrgames.hu/badges/clx123/<uuid>.png" }`. Errors: 404 (badge not found), 503 (R2 not configured), 400 (no file / invalid type / too large).

---

### `DELETE /api/admin/badges/[id]/image`

**Why:** Clears a badge's uploaded artwork, reverting display to the emoji fallback.

**Auth:** `MANAGE_BADGES`.

**Request:** No body.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/badges/clx123/image"
```

**Response:** `{ "ok": true }`.

---

### `GET /api/admin/badges/[id]/awards`

**Why:** Lists who holds a given badge, for the admin badge detail view.

**Auth:** `MANAGE_BADGES`.

**Request:** No params (path `id` only). Returns up to 200 most recent awards.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/badges/clx123/awards"
```

**Response:**
```json
{
  "awards": [
    {
      "userId": "usr_1",
      "name": "Kovács János",
      "image": "https://tr.rbxcdn.com/...",
      "awardedAt": "2026-08-01T12:00:00.000Z",
      "awardedBy": "Admin Name",
      "reason": "Manual grant for event participation"
    }
  ]
}
```

---

### `POST /api/admin/badges/[id]/awards`

**Why:** Manually grants a badge to one or more members. Per the Badges section, a rule is a floor, not a gate — an automatic badge can still be granted by hand for a milestone earned before the app started counting.

**Auth:** `MANAGE_BADGES`.

**Request:** JSON body: `userIds` (array of strings, required, at least one), `reason` (optional string, ≤500 chars).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"userIds":["usr_1","usr_2"],"reason":"Event participation"}' \
  "$HSR_URL/api/admin/badges/clx123/awards"
```

**Response:** `{ "ok": true, "awarded": 2 }` (count actually created; duplicates skipped). Sends a push notification per newly awarded member and invalidates their badge-sync throttle. Errors: 404 (badge or no such member), 400 (no userIds given).

---

### `DELETE /api/admin/badges/[id]/awards/[userId]`

**Why:** Revokes a badge from one member. If the badge is automatic and the member still meets its rule, the next sync re-grants it — revoking is for manual badges or correcting a mis-grant (Badges section).

**Auth:** `MANAGE_BADGES`.

**Request:** No body.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/badges/clx123/awards/usr_1"
```

**Response:** `{ "ok": true }`.

---

### `GET /api/admin/blacklist`

**Why:** Lists application-blacklist entries — people barred from applying to training programmes (not banned from the account). See "Admission Queue & Blacklist": a repeat applicant rarely returns on the same account, so entries are matched on Discord/Roblox id as well.

**Auth:** `MANAGE_BLACKLIST` (`requirePermission((p) => p.canManageBlacklist())`).

**Request:** Query param `all` — pass `1` to include lifted/expired entries; omitted defaults to active-only (not lifted, and either no expiry or not yet expired).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/blacklist?all=1"
```

**Response:**
```json
[
  {
    "id": "bl_1",
    "userId": "usr_1",
    "discordId": "123456789",
    "robloxId": "987654321",
    "displayName": "Kis Péter",
    "reason": "Repeated cheating during exam",
    "expiresAt": null,
    "liftedAt": null,
    "addedById": "usr_admin",
    "active": true,
    "userName": "Kis Péter",
    "addedByName": "Admin Name",
    "liftedByName": null
  }
]
```

---

### `POST /api/admin/blacklist`

**Why:** Adds a new blacklist entry, refusing further applications for the named person(s) — not a ban of their account (Admission Queue & Blacklist section).

**Auth:** `MANAGE_BLACKLIST`.

**Request:** JSON body: `reason` (string, required, ≤1000 chars), `expiresAt` (optional ISO date string, must be in the future), and at least one identifier: `userId`, `discordId`, `robloxId` (strings) — `displayName` optional override. If `userId` is given, `discordId`/`robloxId` are auto-filled from that account. If only `discordId`/`robloxId` are given, a matching account (if any) is resolved and `userId` filled in.

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"userId":"usr_1","reason":"Repeated cheating during exam"}' \
  "$HSR_URL/api/admin/blacklist"
```

**Response:** 201 with the created row, e.g. `{ "id": "bl_2", "userId": "usr_1", "discordId": "...", "robloxId": "...", "reason": "...", "expiresAt": null, ... }`. Errors: 400 (missing/too-long reason, invalid or past `expiresAt`, no identifiers given), 404 (`userId` does not resolve to a user).

---

### `DELETE /api/admin/blacklist/[id]`

**Why:** Lifts a blacklist entry. Stamps `liftedAt`/`liftedById` rather than deleting — the row is the record of when someone was listed and when it was lifted (Admission Queue & Blacklist section).

**Auth:** `MANAGE_BLACKLIST`.

**Request:** No body.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/blacklist/bl_1"
```

**Response:** `{ "success": true }`. Errors: 404 (not found), 400 (`"Already lifted"`).

---

### `POST /api/admin/cargo/regenerate`

**Why:** Regenerates the day's pool of cargo orders (normally done by the `cargo-generate` cron) — an admin override for a specific date. Deletes only `AVAILABLE` orders for that date, leaving `TAKEN` ones (active duties) untouched, then creates 20 fresh random orders.

**Auth:** `MANAGE_CARGO` permission flag (checked manually via `permissions.hasFlag(PERMISSION_FLAGS.MANAGE_CARGO)` after `requireAuth()`).

**Request:** JSON body: `date` (required, ISO date string).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"date":"2026-09-13"}' \
  "$HSR_URL/api/admin/cargo/regenerate"
```

**Response:** `{ "success": true, "generated": 20 }`. Errors: 400 (missing `date`), 403 (missing flag), 500 (internal error).

---

### `GET /api/admin/cargo/history`

**Why:** Lists completed cargo-driver duties for the admin cargo panel — an audit trail of who hauled which order.

**Auth:** `MANAGE_CARGO` (manual flag check).

**Request:** Query param `userId` (optional, filters to one member). Returns up to 100 most recent COMPLETED `CARGO_DRIVER` duties.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/cargo/history?userId=usr_1"
```

**Response:**
```json
[
  {
    "id": "duty_1",
    "dutyType": "CARGO_DRIVER",
    "status": "COMPLETED",
    "startTime": "2026-09-12T10:00:00.000Z",
    "user": { "id": "usr_1", "displayName": "Kis Péter", "name": null, "image": "https://..." },
    "cargoOrder": { "id": "co_1", "orderNumber": 512345, "wagonCount": 12, "totalWeightT": 1080, "destination": "Mecsekháza" }
  }
]
```

---

### `DELETE /api/admin/cargo/history`

**Why:** Removes a bogus/erroneous cargo-driver duty record from history.

**Auth:** `MANAGE_CARGO` (manual flag check).

**Request:** JSON body: `id` (required, the `Duty.id`; must be a `CARGO_DRIVER` duty).

```bash
curl -sS -X DELETE -b cookies.txt -H "Content-Type: application/json" \
  -d '{"id":"duty_1"}' \
  "$HSR_URL/api/admin/cargo/history"
```

**Response:** `{ "success": true }`. Errors: 400 (missing `id`), 404 (not found or not a cargo duty).

---

### `GET /api/admin/cargo/orders`

**Why:** Lists the cargo order pool for a given day, with the taking driver's name resolved, for the admin cargo management page.

**Auth:** `MANAGE_CARGO` (manual flag check).

**Request:** Query param `date` (optional ISO date string; defaults to today, UTC midnight).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/cargo/orders?date=2026-09-13"
```

**Response:**
```json
[
  {
    "id": "co_1",
    "orderNumber": 512345,
    "wagonCount": 12,
    "totalWeightT": 1080,
    "destination": "Mecsekháza",
    "date": "2026-09-13T00:00:00.000Z",
    "status": "TAKEN",
    "takenByUserId": "usr_1",
    "takenByUser": { "id": "usr_1", "displayName": "Kis Péter", "name": null }
  }
]
```

---

### `DELETE /api/admin/cargo/orders`

**Why:** Deletes a single cargo order outright (distinct from regenerating the whole day's pool).

**Auth:** `MANAGE_CARGO` (manual flag check).

**Request:** JSON body: `id` (required, the `CargoOrder.id`).

```bash
curl -sS -X DELETE -b cookies.txt -H "Content-Type: application/json" \
  -d '{"id":"co_1"}' \
  "$HSR_URL/api/admin/cargo/orders"
```

**Response:** `{ "success": true }`. Errors: 400 (missing `id`).

---

### `GET /api/admin/diagnostics/memory`

**Why:** Backs `/admin/diagnostics` ("Server Health") — answers "why is the container holding N GB?" without a heap dump, per the Deployment/Memory section: `heapUsed` well below `rss` means garbage/native buffers, `heapUsed` climbing toward the limit means a real leak, and the per-store sizes point at which in-memory collection is growing.

**Auth:** `VIEW_PERF_MONITOR` (`requirePermission((p) => p.canViewPerfMonitor())`).

**Request:** No params. Ensures the memory sampler and request-memory tracker are running (normally started at boot by `instrumentation.ts`).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/diagnostics/memory"
```

**Response:**
```json
{
  "rss": 512345600,
  "heapUsed": 210000000,
  "heapTotal": 300000000,
  "heapLimit": 2048000000,
  "external": 15000000,
  "history": [{ "t": 1757750400000, "rss": 500000000, "heapUsed": 200000000 }],
  "sse": { "/api/tickets/stream": 3, "/api/admin/tickets/stream": 1 },
  "requests": [{ "route": "/api/duty/dispatcher-slots", "deltaRss": 12000000 }],
  "stores": {
    "gamePerf": 120,
    "trackMapOpenTraces": 4,
    "rateLimiters": 88,
    "automodSpam": 30,
    "automodAiCache": 210,
    "automodAiCalls": 4521,
    "emailQueue": 3,
    "activityBuffer": 2,
    "activityVoiceSessions": 5
  }
}
```

---

### `POST /api/admin/discord/send-embed`

**Why:** Backs the admin embed builder — lets staff compose and post a Discord embed to any guild text channel without leaving the site.

**Auth:** `MANAGE_EMBED_BUILDER` (`requirePermission((p) => p.canUseEmbedBuilder())`).

**Request:** JSON body: `channelId` (string, required), `content` (optional string, plain message text), `embed` (required object with at least `title` or `description`; may also carry `color`, `url`, `timestamp`, `footer`, `image`, `thumbnail`, `author`, `fields`). Requires `DISCORD_BOT_TOKEN` configured server-side.

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"channelId":"123456789012345678","embed":{"title":"Announcement","description":"Service update","color":8388736}}' \
  "$HSR_URL/api/admin/discord/send-embed"
```

**Response:** `{ "success": true, "messageId": "111...", "channelId": "123..." }`. Errors: 400 (missing channelId / embed with no title or description), 500 (bot token not configured, or Discord API failure with its message forwarded).

---

### `GET /api/admin/discord/send-embed`

**Why:** Fetches the guild's text channels (and category names) so the embed builder can offer a channel picker instead of requiring a raw id.

**Auth:** `MANAGE_EMBED_BUILDER`.

**Request:** No params. Requires `DISCORD_BOT_TOKEN` and `DISCORD_GUILD_ID` configured.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/discord/send-embed"
```

**Response:**
```json
{
  "channels": [{ "id": "123456789012345678", "name": "announcements", "parentId": "999" }],
  "categories": { "999": "Information" }
}
```
Errors: 500 (bot token/guild id not configured, or Discord API failure).

---

### `GET /api/admin/dispatch-orders`

**Why:** The admin's full dispatch-order list — every written order between drivers/dispatchers, filterable, for oversight of the signing workflow (draft → awaiting MV → awaiting FSZT → finalized/void).

**Auth:** `MANAGE_DISPATCH_ORDERS` (`requirePermission((p) => p.canManageDispatchOrders())`).

**Request:** Query params (all optional): `status` (`DRAFT|AWAITING_MV|AWAITING_FSZT|FINALIZED|VOID`; `VOID` is synthetic and filters on `voidedAt != null`), `issuerId`, `recipientId`, `q` (text search on `content`), `from`/`to` (ISO date strings, filtered on `createdAt`). Returns up to 500, newest first.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/dispatch-orders?status=AWAITING_FSZT&from=2026-09-01"
```

**Response:** Array of serialized orders, e.g. `[{ "id": "do_1", "status": "AWAITING_FSZT", "content": "...", "issuerId": "usr_1", "recipientId": "usr_2", "revision": 2, "createdAt": "...", ... }]`.

---

### `GET /api/admin/dispatch-orders/[id]`

**Why:** Fetches a single dispatch order for the admin detail/edit view.

**Auth:** `MANAGE_DISPATCH_ORDERS`.

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/dispatch-orders/do_1"
```

**Response:** Serialized order object. Error: 404 `{ "error": "Nem található" }`.

---

### `PATCH /api/admin/dispatch-orders/[id]`

**Why:** Lets an admin correct the text of an order while preserving its workflow state (status, signatures) — bumps `revision` so a corrected order is distinguishable from the original.

**Auth:** `MANAGE_DISPATCH_ORDERS`.

**Request:** JSON body: `content` (string, required, non-empty after trim).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"content":"Corrected order text"}' \
  "$HSR_URL/api/admin/dispatch-orders/do_1"
```

**Response:** Updated serialized order. Errors: 404 (not found), 400 (empty content). Pushes a live update via `emitDispatchOrderUpdate` to both issuer and recipient.

---

### `DELETE /api/admin/dispatch-orders/[id]`

**Why:** Hard-deletes a dispatch order (distinct from voiding — this removes the record entirely, used for admin cleanup rather than the normal invalidation workflow).

**Auth:** `MANAGE_DISPATCH_ORDERS`.

**Request:** No body.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/dispatch-orders/do_1"
```

**Response:** `{ "ok": true }`. Error: 404 (not found). Logs `DISPATCH_ORDER_DELETE` and pushes a live update to both parties.

---

### `GET /api/admin/dispatch-orders/stats`

**Why:** Powers the dispatch-order admin dashboard — volume, status breakdown, top issuers and average time-to-finalize, so oversight staff can see whether the signing workflow is moving.

**Auth:** `MANAGE_DISPATCH_ORDERS`.

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/dispatch-orders/stats"
```

**Response:**
```json
{
  "total": 340,
  "voided": 12,
  "today": 5,
  "statusCounts": { "DRAFT": 2, "AWAITING_MV": 3, "AWAITING_FSZT": 1, "FINALIZED": 322 },
  "daily": [{ "date": "2026-09-07", "count": 8 }],
  "topIssuers": [{ "name": "Kis Péter", "count": 40 }],
  "avgSignMinutes": 14
}
```

---

### `POST /api/admin/dispatch-orders/[id]/void`

**Why:** Invalidates ("érvénytelenítés") an order without deleting it — the row remains as a record but is flagged void with a reason, and the recipient is notified.

**Auth:** `MANAGE_DISPATCH_ORDERS`.

**Request:** JSON body: `reason` (string, required, non-empty after trim).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"reason":"Elírás történt a rendelkezésben"}' \
  "$HSR_URL/api/admin/dispatch-orders/do_1/void"
```

**Response:** Serialized order with `voidedAt`, `voidedById`, `voidedByName`, `voidReason` set. Errors: 404 (not found), 409 (already voided), 400 (empty reason). Sends a push notification to the recipient and live updates to both parties.

---

### `GET /api/admin/documents`

**Why:** The signer queue for the internal document-signing feature (Document Signature Reminders section context) — lists submitted documents (never drafts) with status counts for the overview cards.

**Auth:** `SIGN_DOCUMENTS` via `requireSignerRole()` (checks `auth.permissions.canSignDocuments()`).

**Request:** Query params: `status` (optional, one of `PENDING|SIGNED|REJECTED|REVOKED`), `q` (optional text search across `title`, `verifyCode`, and the author's `name`/`displayName`/`discordUsername`). Returns up to 200, newest submitted first.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/documents?status=PENDING&q=kovacs"
```

**Response:**
```json
{
  "documents": [
    { "id": "doc_1", "title": "Belső szabályzat módosítás", "status": "PENDING", "verifyCode": "AB12CD", "requiredSignatures": 2, "signatures": [], "author": { "id": "usr_1", "displayName": "Kovács János" } }
  ],
  "counts": { "PENDING": 4, "SIGNED": 120, "REJECTED": 2, "REVOKED": 1 }
}
```

---

### `POST /api/admin/documents/[id]/reject`

**Why:** Refuses a pending document with a reason; the author may fix it and resubmit under the same verify code. Per the model, any signatures already collected are dropped — they were given to a version the author is about to change.

**Auth:** `SIGN_DOCUMENTS` via `requireSignerRole()`.

**Request:** JSON body: `reason` (string, required, non-empty; stored truncated to 2000 chars).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"reason":"A dátum hibás a 2. oldalon"}' \
  "$HSR_URL/api/admin/documents/doc_1/reject"
```

**Response:** Serialized document with `status: "REJECTED"`, `rejectedAt`, `rejectedById`, `rejectionReason`. Errors: 400 (missing reason), 404 (not found), 409 (`"Csak aláírásra váró dokumentum utasítható el"` — only a PENDING document can be rejected). Deletes any existing signature rows and their R2 signature images, logs `DOCUMENT_REJECTED`, notifies the author (push + `documentRejected` email template).

---

### `POST /api/admin/documents/[id]/revoke`

**Why:** Invalidates an already-signed document. The stored PDF is left untouched deliberately — copies exist outside the system — and the revocation is checkable via the public `/verify/<code>` page the PDF itself points to.

**Auth:** `SIGN_DOCUMENTS` via `requireSignerRole()`.

**Request:** JSON body: `reason` (string, required, non-empty; stored truncated to 2000 chars).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"reason":"A szabályzat visszavonásra került"}' \
  "$HSR_URL/api/admin/documents/doc_1/revoke"
```

**Response:** Serialized document with `status: "REVOKED"`, `revokedAt`, `revokedById`, `revokedByName` (name plus Discord role, e.g. `"Kis Péter — Forgalmi vezető"`), `revokeReason`. Errors: 400 (missing reason), 404 (not found), 409 (`"Csak aláírt dokumentum vonható vissza"` — only a SIGNED document can be revoked). Logs `DOCUMENT_REVOKED`, notifies the author (push + `documentRevoked` email template).

---

### `POST /api/admin/documents/[id]/sign`

**Why:** Records one required signature on a PENDING document; once the last required signature is in, renders and stores the final PDF and flips the document to SIGNED. A signer can only sign once (unique on `documentId`+`signerId`).

**Auth:** `SIGN_DOCUMENTS` via `requireSignerRole()`. Requires R2 storage configured.

**Request:** JSON body: `signerName` (string, required), `signerTitle` (string, required), `signatureImage` (optional — a PNG data URL exactly matching `data:image/png;base64,...`, max 2MB decoded; omit/null/empty for no drawn signature image).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"signerName":"Kis Péter","signerTitle":"Forgalmi vezető","signatureImage":"data:image/png;base64,iVBORw0K..."}' \
  "$HSR_URL/api/admin/documents/doc_1/sign"
```

**Response (still short of required signatures):** serialized document, still `status: "PENDING"`, with the new signature appended to `signatures`. Notifies the author by push only.

**Response (closing signature):** serialized document with `status: "SIGNED"`, `pdfKey`, `pdfSha256`, `pdfSize`, `signedAt`, and the closing signer's `signerId`/`signerName`/`signerTitle`. Notifies the author (push + `documentSigned` email template).

Errors: 500 (R2 not configured), 404 (document not found), 409 (`"Csak aláírásra váró dokumentum írható alá"` if not PENDING, `"Ezt a dokumentumot már aláírtad"` if the signer already signed, or `"A dokumentumot időközben már elbírálták"`/PDF-issuance retry exhaustion), 400 (missing `signerName`/`signerTitle`, invalid or oversized `signatureImage`).

## Admin: Duty Management

### `GET /api/admin/duty`

**Why:** Powers the admin duty dashboard — daily/monthly counts, a monthly top-performer leaderboard, the 50 most recent duties (with consist match info), and the list of approved driver schedule assignments nobody actually fulfilled (with a 1-minute grace-period and departure-time check per "Booking Time Conflicts"/"Schedule Assignment & Fulfillment Tracking").

**Auth:** `requirePermission((p) => p.canManageDuty())`. No rank-hierarchy guard (read-only).

**Request:** Query params, all optional:
- `view` — `"daily" | "monthly" | "all"` (default `"daily"`; read but not actually branched on in the handler beyond being parsed)
- `userId` — filter to one user
- `type` — `"TRAIN_DRIVER" | "TRAIN_DISPATCHER"` (anything else ignored)
- `unfulfilled` — `"today" | "all"` (default `"today"`) — scopes the unfulfilled-assignments window

```bash
curl -sS -X GET -b cookies.txt "$HSR_URL/api/admin/duty?unfulfilled=today"
```

**Response:**
```json
{
  "daily": { "total": 12, "active": 3, "completed": 9, "totalMinutes": 540, "totalHours": 9 },
  "monthly": {
    "total": 210, "totalMinutes": 9600, "totalHours": 160,
    "byType": { "trainDriver": 140, "trainDispatcher": 70 }
  },
  "leaderboard": [
    { "user": { "id": "u1", "displayName": "Kovács Béla", "name": "Bela123", "image": null, "discordRole": "Train Driver" }, "dutyCount": 18 }
  ],
  "recentDuties": [
    {
      "id": "d1", "userId": "u1", "dutyType": "TRAIN_DRIVER", "status": "COMPLETED",
      "startTime": "2026-09-12T10:00:00.000Z", "endTime": "2026-09-12T11:30:00.000Z",
      "user": { "id": "u1", "displayName": "Kovács Béla" },
      "startLocation": { "id": "s1", "name": "Budapest" },
      "assignedTrains": [{ "train": { "id": "t1", "trainNumber": "123" } }],
      "consistMatch": { "matched": true }
    }
  ],
  "unfulfilledAssignments": [
    { "id": "req1", "user": { "id": "u2", "displayName": "..." }, "train": { "id": "t2", "trainNumber": "456", "trainType": "IC" }, "date": "2026-09-13T00:00:00.000Z" }
  ]
}
```
Error: `500 { "error": "Failed to fetch duty statistics" }`.

### `GET /api/admin/duty/strikes`

**Why:** Lists weekly duty compliance strikes (warnings/terminations) for the admin panel, with each strike's appeal link precomputed.

**Auth:** `requirePermission((p) => p.hasFlag("MANAGE_DUTY"))`.

**Request:** Query params, both optional:
- `active` — `"true"` filters to `clearedAt: null` and `result != "TERMINATION"`
- `userId` — must match `UUID_REGEX` (`/^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i`) or `400` is returned

```bash
curl -sS -X GET -b cookies.txt "$HSR_URL/api/admin/duty/strikes?active=true"
```

**Response:**
```json
[
  {
    "id": "strike1", "userId": "u1", "weekStart": "2026-09-07T00:00:00.000Z",
    "result": "WARNING", "clearedAt": null, "manual": false, "note": null,
    "appealToken": "9d2e...", "appealDeadline": "2026-09-27T00:00:00.000Z",
    "user": { "id": "u1", "displayName": "...", "discordUsername": "..." },
    "appeal": null,
    "appealUrl": "https://hsrgames.hu/duty-appeal/strike1?token=9d2e..."
  }
]
```
Error: `400 { "error": "Invalid userId format" }` when `userId` fails the UUID regex.

### `GET /api/admin/duty/calendar`

**Why:** Feeds the admin calendar view — either every duty across all users in a period (global view) or one user's duties, grouped by date, with aggregate stats.

**Auth:** `requirePermission((p) => p.canManageDuty())`.

**Request:** Query params, all optional:
- `userId` — if omitted, returns the global (all-users) view; if present, must resolve to an existing `User` or `404`
- `month` — `"YYYY-MM"`, takes precedence over `year`
- `year` — `"YYYY"`
- If neither `month` nor `year` given, defaults to the current calendar month (server local time, not UTC)

```bash
curl -sS -X GET -b cookies.txt "$HSR_URL/api/admin/duty/calendar?userId=u1&month=2026-09"
```

**Response (with `userId`):**
```json
{
  "user": { "id": "u1", "displayName": "...", "name": "...", "image": null, "discordRole": "..." },
  "period": { "start": "2026-09-01T00:00:00.000Z", "end": "2026-09-30T23:59:59.000Z" },
  "stats": {
    "totalDuties": 14, "completedDuties": 12, "totalMinutes": 1020, "totalHours": 17,
    "byType": { "trainDriver": 10, "trainDispatcher": 4 }
  },
  "duties": [ { "id": "d1", "date": "2026-09-05T00:00:00.000Z", "dutyType": "TRAIN_DRIVER", "...": "..." } ],
  "dutyByDate": { "2026-09-05": [ { "id": "d1" } ] }
}
```
Without `userId`, `"user"` is `null` and duties/stats span all users. Error: `404 { "error": "User not found" }`; `500 { "error": "Failed to fetch calendar data" }`.

### `POST /api/admin/duty/reset`

**Why:** The daily duty reset — closes out any duty still `ACTIVE` from a previous day (auto-ended at midnight) and deletes orphaned `DutyTrain` assignments left behind by old active or completed duties, so stale state doesn't leak into the next day's board (see "Duty Re-entry After a Drop-out").

**Auth:** Multi-path: (1) `Authorization: Bearer <CRON_SECRET>`, (2) `x-api-key: <GAME_API_KEY>`, or (3) a logged-in session where `permissions.canViewAdminPanel()` is true (built from `syncDiscordRoles` + `resolveUserFlags`, not a bare `requirePermission` call). No specific `MANAGE_DUTY` flag check — the admin-panel-visibility gate is the guard. This route is listed in `PUBLIC_PATHS` so those bearer/api-key checks are the only thing stopping an anonymous caller.

**Request:** No body required.

```bash
curl -sS -X POST -H "Authorization: Bearer $CRON_SECRET" "$HSR_URL/api/admin/duty/reset"
```

**Response:**
```json
{
  "message": "Daily reset completed",
  "endedDuties": 3,
  "deletedAssignments": 5,
  "resetDate": "2026-09-13T00:00:00.000Z"
}
```
Error: `401 { "error": "Unauthorized" }`; `500 { "error": "Failed to reset duties" }`.

### `GET /api/admin/duty/reset`

**Why:** Dry-run/preview counterpart to the reset POST — reports how many active duties and orphaned assignments would be cleaned up, without changing anything (used by the admin UI's "Reset" confirmation).

**Auth:** Same multi-path check as the POST above (`CRON_SECRET` bearer, `GAME_API_KEY` header, or session with `canViewAdminPanel()`).

**Request:** No params.

```bash
curl -sS -X GET -H "Authorization: Bearer $CRON_SECRET" "$HSR_URL/api/admin/duty/reset"
```

**Response:**
```json
{
  "activeDutiesToEnd": 3,
  "assignmentsToDelete": 5,
  "totalItemsToReset": 8,
  "message": "3 active duties and 5 train assignments will be reset"
}
```
Error: `401 { "error": "Unauthorized" }`; `500 { "error": "Failed to check duties" }`.

### `GET /api/admin/duty/missed-appeals`

**Why:** Lists appeals submitted against missed schedule assignments, for the admin review queue.

**Auth:** `requirePermission((p) => p.hasFlag("MANAGE_DUTY"))`.

**Request:** Query param `status` (optional) — `"PENDING" | "APPROVED" | "REJECTED"`, cast directly with no validation against the enum.

```bash
curl -sS -X GET -b cookies.txt "$HSR_URL/api/admin/duty/missed-appeals?status=PENDING"
```

**Response:**
```json
[
  {
    "id": "appeal1", "status": "PENDING", "createdAt": "2026-09-10T12:00:00.000Z",
    "user": { "id": "u1", "displayName": "...", "discordUsername": "..." },
    "missedAssignment": { "id": "ma1", "date": "2026-09-08T00:00:00.000Z", "train": { "trainNumber": "123", "trainType": "IC" } },
    "reviewedBy": null
  }
]
```

### `GET /api/admin/duty/weekly-driver`

**Why:** Previews the weekly driver summary/announcement before it is sent (which trains/hours/members it would report on).

**Auth:** `requirePermission((p) => p.canManageDuty())`.

**Request:** No params.

```bash
curl -sS -X GET -b cookies.txt "$HSR_URL/api/admin/duty/weekly-driver"
```

**Response:** Shape of `getWeeklyDriverPreview()` — e.g.:
```json
{
  "weekStart": "2026-09-07T00:00:00.000Z",
  "weekEnd": "2026-09-13T23:59:59.000Z",
  "entries": [ { "userId": "u1", "displayName": "...", "totalHours": 12.5 } ]
}
```
Error: `500 { "error": "Failed to load weekly driver preview" }`.

### `POST /api/admin/duty/weekly-driver`

**Why:** Manually triggers the weekly driver summary announcement (normally sent by the `weekly-driver` cron) — resets the weekly stats it reports on as part of sending.

**Auth:** `requirePermission((p) => p.canManageDuty())`.

**Request:** JSON body, optional:
- `resetAfterSend` — boolean, defaults to `true`. Passing `false` is rejected with `400`, since a send that doesn't reset the weekly stats is not supported by this route.

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"resetAfterSend": true}' \
  "$HSR_URL/api/admin/duty/weekly-driver"
```

**Response:**
```json
{ "success": true, "sent": true, "recipientCount": 1 }
```
(Extra fields come from `sendWeeklyDriverAnnouncement()`'s return value, spread onto the response.) Error: `400 { "error": "Weekly driver send must reset weekly stats" }`; `500 { "error": "Failed to send weekly driver" }`.

### `GET /api/admin/duty/appeals`

**Why:** Lists appeals against weekly duty compliance strikes (warnings/terminations), for the admin review queue.

**Auth:** `requirePermission((p) => p.hasFlag("MANAGE_DUTY"))`.

**Request:** Query param `status` (optional) — cast with `as any`, no validation.

```bash
curl -sS -X GET -b cookies.txt "$HSR_URL/api/admin/duty/appeals?status=PENDING"
```

**Response:**
```json
[
  {
    "id": "appeal1", "status": "PENDING", "createdAt": "2026-09-10T12:00:00.000Z",
    "user": { "id": "u1", "displayName": "...", "discordUsername": "..." },
    "strike": { "id": "strike1", "weekStart": "2026-09-07T00:00:00.000Z", "result": "WARNING", "removedRoles": [] },
    "reviewedBy": null
  }
]
```

### `GET /api/admin/duty/weekly-check-setting`

**Why:** Reads whether the Monday weekly-duty-compliance cron run is enabled at all (`WeeklyDutySettings` singleton, id 1).

**Auth:** `requireAuth()` then an inline check: `permissions.canManageDuty() || permissions.canViewAdminPanel()` — either is sufficient, not `MANAGE_DUTY` alone.

**Request:** No params.

```bash
curl -sS -X GET -b cookies.txt "$HSR_URL/api/admin/duty/weekly-check-setting"
```

**Response:**
```json
{ "enabled": true }
```
A missing settings row defaults to `enabled: true` ("the check predates the toggle"). Error: `403 { "error": "Forbidden" }`; `500 { "error": "Failed to fetch weekly duty check setting" }`.

### `PUT /api/admin/duty/weekly-check-setting`

**Why:** Toggles the Monday weekly duty compliance run on/off — writes an activity log entry since a disabled check silently means nobody gets a strike that week.

**Auth:** Same as GET: `requireAuth()` + (`canManageDuty()` or `canViewAdminPanel()`).

**Request:** JSON body:
- `enabled` — coerced with `Boolean(body.enabled)`, no type validation beyond that

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"enabled": false}' \
  "$HSR_URL/api/admin/duty/weekly-check-setting"
```

**Response:**
```json
{ "enabled": false }
```
Error: `403 { "error": "Forbidden" }`; `500 { "error": "Failed to update weekly duty check setting" }`.

### `POST /api/admin/duty/unfulfilled-appeal-link`

**Why:** Generates (or reuses) a token-authenticated appeal link for a specific missed driver assignment directly from the "unfulfilled today" list, resolving to the same `MissedAssignment` row (keyed by the pair's first train) the nightly cron would create after midnight — so the link doesn't fork into a duplicate record.

**Auth:** `requirePermission((p) => p.hasFlag("MANAGE_DUTY"))`.

**Request:** JSON body, all required:
- `userId` — string
- `trainId` — string, must resolve to an existing `Train` (`404` otherwise)
- `date` — parseable date string (`400 "Invalid date"` if not)

A token/deadline is reused if one already exists and hasn't expired; otherwise a new `randomUUID()` token is minted with a 15-day (`APPEAL_DEADLINE_DAYS`) deadline. Refuses if an appeal already exists for the resolved record (`409`).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"userId":"u1","trainId":"t1","date":"2026-09-12"}' \
  "$HSR_URL/api/admin/duty/unfulfilled-appeal-link"
```

**Response:**
```json
{
  "url": "https://hsrgames.hu/missed-assignment-appeal/ma1?token=9d2e...",
  "token": "9d2e...",
  "deadline": "2026-09-27T00:00:00.000Z"
}
```
Error: `400 { "error": "userId, trainId and date are required" }` / `{ "error": "Invalid date" }`; `404 { "error": "Train not found" }`; `409 { "error": "Appeal already submitted" }`.

### `GET /api/admin/duty/exemptions`

**Why:** Lists weekly-duty exemptions (leave grants) for the admin panel, optionally scoped to one week.

**Auth:** `requirePermission((p) => p.hasFlag("MANAGE_DUTY"))`.

**Request:** Query param `weekStart` (optional) — parsed with `new Date(weekStart)`, no format validation.

```bash
curl -sS -X GET -b cookies.txt "$HSR_URL/api/admin/duty/exemptions?weekStart=2026-09-07"
```

**Response:**
```json
[
  {
    "id": "ex1", "userId": "u1", "weekStart": "2026-09-07T00:00:00.000Z", "weekEnd": "2026-09-13T00:00:00.000Z",
    "reason": "Vacation", "deactivatedAt": null,
    "user": { "id": "u1", "displayName": "...", "name": "...", "image": null },
    "grantedBy": { "id": "u2", "displayName": "...", "name": "..." }
  }
]
```

### `POST /api/admin/duty/exemptions`

**Why:** Grants a member a weekly-duty exemption (excused from the weekly duty minimum) for a date range — same writer path (`createLeaveExemptions`) the support bot's leave flow uses, and emails the member so the exemption isn't only visible in an admin table.

**Auth:** `requirePermission((p) => p.hasFlag("MANAGE_DUTY"))`.

**Request:** JSON body:
- `userId` — required
- `weekStart` — required, `"YYYY-MM-DD"` (strict, validated via `parseDateOnly`; `400` on bad format)
- `weekEnd` — required, `"YYYY-MM-DD"`; must be `>= weekStart` (`400` otherwise)
- `reason` — optional string

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"userId":"u1","weekStart":"2026-09-07","weekEnd":"2026-09-13","reason":"Vacation"}' \
  "$HSR_URL/api/admin/duty/exemptions"
```

**Response (`201`):** Returns whatever `createLeaveExemptions()` returns (one or more created `WeeklyDutyExemption` rows), e.g.:
```json
[ { "id": "ex1", "userId": "u1", "weekStart": "2026-09-07T00:00:00.000Z", "weekEnd": "2026-09-13T00:00:00.000Z", "reason": "Vacation" } ]
```
Error: `400 { "error": "userId, weekStart, and weekEnd required" }` / `{ "error": "Invalid date format" }` / `{ "error": "weekEnd must be on or after weekStart" }`.

### `POST /api/admin/duty/[id]/remove`

**Why:** Force-ends a still-`ACTIVE` duty (marks it `COMPLETED`) from the admin panel — an admin intervention distinct from the daily auto-reset, e.g. a stuck session.

**Auth:** `requirePermission((p) => p.isSiteAdmin())` — **not** `MANAGE_DUTY`; this action is explicitly restricted to site admins.

**Request:** Path param `id` (duty id). JSON body optional: `reason` — string, appended to the duty's `notes`.

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"reason":"Stuck session"}' \
  "$HSR_URL/api/admin/duty/d1/remove"
```

**Response:**
```json
{
  "message": "Duty force removed successfully",
  "duty": {
    "id": "d1", "status": "COMPLETED", "endTime": "2026-09-13T14:00:00.000Z",
    "notes": "[FORCE REMOVED by Admin Name at 2026-09-13T14:00:00.000Z: Stuck session]",
    "user": { "id": "u1", "displayName": "..." },
    "startLocation": { "id": "s1", "name": "Budapest" },
    "assignedTrains": []
  }
}
```
Error: `404 { "error": "Duty not found" }`; `500 { "error": "Failed to force remove duty" }`.

### `DELETE /api/admin/duty/[id]/delete`

**Why:** Permanently deletes a duty record (cascades to its `DutyTrain` rows) — for cleaning up erroneous entries entirely, as opposed to force-ending one still in progress.

**Auth:** `requirePermission((p) => p.isSiteAdmin())` — **not** `MANAGE_DUTY`; site-admin only.

**Request:** Path param `id` (duty id). No body.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/duty/d1/delete"
```

**Response:**
```json
{
  "message": "Duty deleted successfully",
  "deletedDuty": { "id": "d1", "user": "Kovács Béla", "dutyType": "TRAIN_DRIVER", "date": "2026-09-12T00:00:00.000Z" }
}
```
Error: `404 { "error": "Duty not found" }`; `500 { "error": "Failed to delete duty" }`.

### `PATCH /api/admin/duty/exemptions/[id]`

**Why:** Deactivates or reactivates an existing exemption without deleting it — e.g. a granted leave that needs to be cut short, then possibly restored.

**Auth:** `requirePermission((p) => p.hasFlag("MANAGE_DUTY"))`.

**Request:** Path param `id` (exemption id). JSON body: `deactivate` — truthy sets `deactivatedAt` to now, falsy/absent clears it to `null`.

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"deactivate": true}' \
  "$HSR_URL/api/admin/duty/exemptions/ex1"
```

**Response:**
```json
{ "id": "ex1", "userId": "u1", "deactivatedAt": "2026-09-13T14:00:00.000Z", "...": "..." }
```
Error: `404 { "error": "Not found" }`.

### `DELETE /api/admin/duty/exemptions/[id]`

**Why:** Permanently revokes an exemption.

**Auth:** `requirePermission((p) => p.hasFlag("MANAGE_DUTY"))`.

**Request:** Path param `id` (exemption id). No body.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/duty/exemptions/ex1"
```

**Response:**
```json
{ "success": true }
```
Error: `404 { "error": "Not found" }`.

### `PATCH /api/admin/duty/missed-appeals/[id]`

**Why:** Approves or rejects a member's appeal against a missed schedule assignment; on approval the assignment is effectively excused (the appeal is logged and the member emailed either way).

**Auth:** `requirePermission((p) => p.hasFlag("MANAGE_DUTY"))`. No rank-hierarchy guard.

**Request:** Path param `id` (appeal id). JSON body: `status` — must be `"APPROVED"` or `"REJECTED"` (`400` otherwise). Fails with `400` if the appeal is not still `PENDING`.

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"status": "APPROVED"}' \
  "$HSR_URL/api/admin/duty/missed-appeals/appeal1"
```

**Response:**
```json
{ "success": true }
```
Side effects: writes `logAppealDecision()`, and if `user.discordEmail` is set, sends the `missedAssignmentAppealDecision` template email in the user's locale. Error: `404 { "error": "Not found" }`; `400 { "error": "Invalid status" }` / `{ "error": "Appeal already reviewed" }`.

### `PATCH /api/admin/duty/appeals/[id]`

**Why:** Approves or rejects a member's appeal against a weekly duty compliance strike; approval clears the strike and restores whatever Discord roles the strike had removed.

**Auth:** `requirePermission((p) => p.hasFlag("MANAGE_DUTY"))`. No rank-hierarchy guard.

**Request:** Path param `id` (appeal id). JSON body: `status` — must be `"APPROVED"` or `"REJECTED"` (`400` otherwise). Fails with `400` if the appeal is not still `PENDING`.

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"status": "APPROVED"}' \
  "$HSR_URL/api/admin/duty/appeals/appeal1"
```

**Response:**
```json
{ "success": true }
```
Side effects on approval: clears `weeklyDutyStrike.clearedAt`, re-adds each role in `strike.removedRoles` to the user via `addDiscordRole` (300ms spacing between calls), logs `APPEAL_APPROVED` to `ActivityLog`, emails `dutyStrikeAppealDecision` if `discordEmail` set. On rejection: logs `APPEAL_REJECTED` and still emails the decision. Error: `404 { "error": "Not found" }`; `400 { "error": "Invalid status" }` / `{ "error": "Appeal already reviewed" }`.

### `POST /api/admin/duty/strikes/[id]/appeal-link`

**Why:** Generates (or reuses) a token-authenticated appeal link for an existing weekly duty strike, so an admin can hand it to a member who missed the automatic notification.

**Auth:** `requirePermission((p) => p.hasFlag("MANAGE_DUTY"))`.

**Request:** Path param `id` (strike id). No body. Refuses with `409` if the strike already has an appeal. Reuses the existing token/deadline unless missing or expired, otherwise mints a new `randomUUID()` token with a 15-day (`APPEAL_DEADLINE_DAYS`) deadline.

```bash
curl -sS -X POST -b cookies.txt "$HSR_URL/api/admin/duty/strikes/strike1/appeal-link"
```

**Response:**
```json
{
  "url": "https://hsrgames.hu/duty-appeal/strike1?token=9d2e...",
  "token": "9d2e...",
  "deadline": "2026-09-27T00:00:00.000Z"
}
```
Error: `404 { "error": "Strike not found" }`; `409 { "error": "Appeal already submitted" }`.

### `POST /api/admin/duty/users/[id]/warn`

**Why:** Issues a manual weekly-duty warning outside the Monday cron — writes the same `WeeklyDutyStrike` row shape the cron writes (`result: "WARNING"`), so a later miss escalates from it rather than starting a fresh cycle.

**Auth:** `requirePermission((p) => p.hasFlag("MANAGE_DUTY"))`. No rank-hierarchy guard in this route (unlike terminate, which blocks self-termination — warn has no such check).

**Request:** Path param `id` (target user id). JSON body: `note` — required non-empty string (trimmed), the warning reason (`400` if blank). Refuses with `409` if the user already has an active `WARNING`-result strike, or if a strike already exists for the current week (Prisma `P2002` unique violation on `userId_weekStart`).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"note": "Missed weekly minimum"}' \
  "$HSR_URL/api/admin/duty/users/u1/warn"
```

**Response:**
```json
{ "success": true, "strikeId": "strike2" }
```
Side effects: writes `WEEKLY_STRIKE_ISSUED` to `ActivityLog`, emails `dutyManualWarning` template if `discordEmail` set, with an appeal link. Error: `404 { "error": "User not found" }`; `400 { "error": "A reason is required to issue a manual warning" }`; `409 { "error": "This user already has an active warning. You can only terminate them now." }` / `{ "error": "A strike already exists for this user this week" }`.

### `POST /api/admin/duty/users/[id]/terminate`

**Why:** Manually terminates a member's weekly-duty standing (role removal) independent of any prior warning — a direct termination requires no second offense. Removes the same role set the Monday cron would and closes any open warning so a re-granted role starts clean.

**Auth:** `requirePermission((p) => p.hasFlag("MANAGE_DUTY"))`. Additional guards: cannot terminate yourself (`id === session.id` → `400`); cannot terminate a user holding an immune role (`isImmune()` against `getImmuneRoleIds()` → `400`) — this is the closest thing to a rank-hierarchy guard on this route, expressed as role immunity rather than a priority comparison.

**Request:** Path param `id` (target user id). JSON body:
- `note` — required non-empty string (trimmed), reason (`400` if blank)
- `confirm` — must be exactly boolean `true` (`400` otherwise)

Discord role removal happens **before** any DB write: if `removeDiscordRole()` fails for any role, the route returns `502` and records nothing.

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"note": "Repeated non-compliance", "confirm": true}' \
  "$HSR_URL/api/admin/duty/users/u1/terminate"
```

**Response:**
```json
{ "success": true, "strikeId": "strike3" }
```
Side effects: removes each non-immune Discord role from the user (300ms spacing), grants back `MEMBER_ROLE_ID`, upserts a `WeeklyDutyStrike` (`result: "TERMINATION"`, `removedRoles` recorded) keyed on `userId_weekStart`, closes any other open `WARNING` strike for the user, writes `WEEKLY_TERMINATION` to `ActivityLog`, emails `dutyManualTermination` if `discordEmail` set. Error: `404 { "error": "User not found" }`; `400 { "error": "A reason is required to terminate a user" }` / `{ "error": "confirm must be true" }` / `{ "error": "You cannot terminate yourself" }` / `{ "error": "This user is immune to duty termination and cannot be terminated here" }`; `502 { "error": "Failed to remove a Discord role — some roles may already be gone. Nothing was recorded." }`.

## Admin: Email, Exam Slots, Exams

### `GET /api/admin/email/queue`

**Why:** The email outbox is the only record answering "did this member get theirs?" — everything is queued through `EmailOutbox` rather than an in-process array, so this lists that all-time record, paged and filterable.

**Auth:** `MANAGE_USERS`

**Request:** Query params, all optional: `recipient` (string, substring match), `status` (`PENDING`/`SENDING`/`SENT`/`FAILED`), `limit` (number, clamped 1-200, default 100), `offset` (number, default 0).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/email/queue?status=FAILED&limit=50"
```

**Response:**
```json
{
  "stats": {
    "pending": 3,
    "sending": 0,
    "failed": 2,
    "sentLast24h": 148,
    "oldestPendingAt": "2026-09-13T08:12:00.000Z"
  },
  "entries": [
    {
      "id": "cmeoutbox1",
      "recipient": "user@example.com",
      "subject": "Your closing exam result",
      "templateId": "closingExamResult",
      "status": "FAILED",
      "attempts": 6,
      "runAfter": null,
      "sentAt": null,
      "failedAt": "2026-09-13T09:00:00.000Z",
      "lastError": "550 5.1.1 unknown recipient",
      "createdAt": "2026-09-12T20:00:00.000Z"
    }
  ],
  "total": 2,
  "limit": 50,
  "offset": 0
}
```
Error: `401`/`403` if the flag is missing.

### `POST /api/admin/email/queue`

**Why:** Lets staff re-queue a letter that permanently failed, or force the drain to run now instead of waiting for the next `/api/cron/email-queue` tick.

**Auth:** `MANAGE_USERS`

**Request:** JSON body: `action` (`"retry"` | `"drain"`, required), `id` (string, the `EmailOutbox` row id — required only for `"retry"`).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"action":"retry","id":"cmeoutbox1"}' \
  "$HSR_URL/api/admin/email/queue"
```

**Response:**
```json
{ "message": "Re-queued" }
```
`action: "drain"` responds `{ "message": "Drained", "sent": 12, "failed": 1, "retried": 0 }`. Errors: `400` "id is required" / "Unknown action", `404` "Not a failed email" (retry on a row that isn't FAILED).

### `GET /api/admin/email/templates`

**Why:** Backs the Templates tab: the full registry index (id, category, label, call sites) plus the wiring report, so the panel can show which templates nothing currently sends without a grep.

**Auth:** `MANAGE_USERS`

**Request:** No parameters.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/email/templates"
```

**Response:**
```json
{
  "templates": [
    {
      "id": "closingExamResult",
      "category": "training",
      "label": "Closing exam result",
      "usage": [{ "file": "src/lib/closing-exam-result.ts", "line": 40 }]
    }
  ],
  "wiring": {
    "dynamicSends": [{ "file": "src/lib/admission/queue.ts", "line": 88 }],
    "unregisteredBuilders": [{ "fn": "buildSomeEmail", "file": "src/lib/email/email.ts", "line": 10 }],
    "rawSends": [{ "fn": "sendRawEmail", "file": "src/app/api/admin/email/send/route.ts", "line": 113 }]
  }
}
```

### `POST /api/admin/email/templates`

**Why:** Renders one template from its own sample data and mails it to the admin who asked — the same letter a member would get, so the whole list can be sanity-checked without waiting for the real trigger. Deliberately takes no recipient parameter: a test that could be aimed at somebody else would be a second, unaudited blast endpoint.

**Auth:** `MANAGE_USERS`, plus a 10-per-minute-per-admin rate limit (separate from the 5-minute `email-cooldown`, since the tab exists to click through the whole list).

**Request:** JSON body: `id` (string, must be a known `EmailTemplateId`, required). No recipient field — it always sends to the calling admin's own `discordEmail`.

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"id":"closingExamResult"}' \
  "$HSR_URL/api/admin/email/templates"
```

**Response:**
```json
{ "message": "Test sent to admin@example.com", "subject": "[TEST] Your closing exam result" }
```
Errors: `400` "Unknown template" / "Your account has no email address, so there is nowhere to send the test.", `429` rate limited.

### `GET /api/admin/email/send`

**Why:** Feeds the blast picker with the audiences it can target (and their live sizes), built from data (including training programmes) rather than a hardcoded list in the page.

**Auth:** `MANAGE_USERS`

**Request:** No parameters.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/email/send"
```

**Response:**
```json
{
  "audiences": [
    { "key": "all", "label": "All users", "count": 512 },
    { "key": "trainees", "label": "Trainees", "count": 34, "programs": [{ "key": "DRIVER", "name": "Train driver", "count": 20 }] }
  ]
}
```

### `POST /api/admin/email/send`

**Why:** The custom email blast — one written letter sent either to a single member by email or to a whole audience. Queueing the whole audience as one `createMany` write (via `sendRawEmails`) rather than one row at a time is what keeps a few-hundred-recipient blast from being cut off half-written mid-request.

**Auth:** `MANAGE_USERS`

**Request:** JSON body: `subject` (string, required, non-empty), `bodyHtml` (string, required, non-empty), `email` (string, optional — single-recipient mode, matched against `discordEmail`), `audience` (string, optional, must be a known `EmailAudience` key, default `"all"` — ignored if `email` is set), `programKey` (string, optional, only used when `audience === "trainees"`).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"subject":"Maintenance notice","bodyHtml":"<p>Service will be down Friday.</p>","audience":"trainees","programKey":"DRIVER"}' \
  "$HSR_URL/api/admin/email/send"
```

**Response (audience mode):**
```json
{ "message": "Queued email blast to 20 Train driver trainees", "queued": 20, "queueLength": 3 }
```
Single-recipient mode responds `{ "message": "Queued email to user@example.com", "queueLength": 3 }`. Errors: `400` "Subject is required" / "Body is required" / "Unknown audience" / "That audience has nobody with an email address in it", `404` "User not found" (single-recipient mode, no matching `discordEmail`).

### `GET /api/admin/email/queue/stream`

**Why:** Live version of the outbox listing (`GET /api/admin/email/queue`), so a stuck or failing queue is visible without refreshing — a row's *status* changing (PENDING → SENDING → SENT, or a failure landing) is the interesting event, not new rows.

**Auth:** `MANAGE_USERS`

**Request:** Same query params as `GET /api/admin/email/queue` (`recipient`, `status`, `limit`, `offset`), read once at connect time and reused for every tick.

```bash
curl -sS -N -b cookies.txt "$HSR_URL/api/admin/email/queue/stream?status=PENDING"
```

**Response:** SSE stream (25s heartbeat). Every 3 seconds the server re-runs the same query as the REST route and pushes the full current page as one event — skipped if byte-identical to the last push:
```
data: {"stats":{"pending":3,"sending":0,"failed":2,"sentLast24h":148,"oldestPendingAt":"2026-09-13T08:12:00.000Z"},"entries":[...],"total":3,"limit":100,"offset":0}

```

### `GET /api/admin/exam-slots`

**Why:** Lists every closing-exam time slot (manually scheduled or vote-scheduled) for the admin's Exam slots tab.

**Auth:** `MANAGE_EXAMS`

**Request:** No parameters.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/exam-slots"
```

**Response:**
```json
[
  {
    "id": "cmeslot1",
    "examId": "cmeexam1",
    "scheduledAt": "2026-09-14T14:00:00.000Z",
    "status": "SCHEDULED",
    "hostId": "cmeuser1",
    "discordEventId": null,
    "exam": { "id": "cmeexam1", "title": "Driver closing exam", "trainingBranch": "DRIVER" },
    "_count": { "registrations": 5 }
  }
]
```

### `POST /api/admin/exam-slots`

**Why:** Manually schedules a closing-exam slot — the same lifecycle the daily vote's cron also creates through `announceExamSlot()` — and invites every eligible trainee of that branch by email. This is the manual form kept alongside the vote; a closing exam otherwise carries no other run buttons on `/admin/exams` (see "A closing exam therefore carries no run buttons on /admin/exams" — Start/End/the cron are the whole lifecycle).

**Auth:** `MANAGE_EXAMS`

**Request:** JSON body: `examId` (string, required, must reference an `Exam` with `audience === "CLOSING_EXAM"`), `scheduledAt` (ISO datetime string, required).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"examId":"cmeexam1","scheduledAt":"2026-09-14T14:00:00.000Z"}' \
  "$HSR_URL/api/admin/exam-slots"
```

**Response (201):**
```json
{
  "id": "cmeslot1",
  "examId": "cmeexam1",
  "scheduledAt": "2026-09-14T14:00:00.000Z",
  "status": "SCHEDULED",
  "hostId": "cmeuser1",
  "discordEventId": null
}
```
Errors: `400` "examId and scheduledAt required" / "Invalid exam" (not found, or not a CLOSING_EXAM).

### `GET /api/admin/exam-slots/[id]`

**Why:** Detail view of one slot including its registrations, for the slot detail page.

**Auth:** `MANAGE_EXAMS`

**Request:** Path param `id` (slot id).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/exam-slots/cmeslot1"
```

**Response:**
```json
{
  "id": "cmeslot1",
  "examId": "cmeexam1",
  "scheduledAt": "2026-09-14T14:00:00.000Z",
  "status": "SCHEDULED",
  "exam": { "id": "cmeexam1", "title": "Driver closing exam" },
  "registrations": [
    {
      "id": "cmereg1",
      "userId": "cmeuser2",
      "attended": null,
      "user": { "id": "cmeuser2", "displayName": "Someone", "name": "someone", "discordId": "111" }
    }
  ]
}
```
Error: `404` "Not found".

### `DELETE /api/admin/exam-slots/[id]`

**Why:** Cancels a scheduled/active slot. Everyone still registered on it (not already `COMPLETED`) is mailed `examSlotCancelled` *before* the delete — deleting first would leave nobody to notify — and told this is not a no-show, since a missed exam otherwise costs deadline days.

**Auth:** `MANAGE_EXAMS`

**Request:** Path param `id` (slot id). No body.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/exam-slots/cmeslot1"
```

**Response:**
```json
{ "ok": true, "notified": 5 }
```
Error: `404` "Not found".

### `PUT /api/admin/exam-slots/[id]/attendance`

**Why:** A manual override for correcting attendance after the fact. `closeExamSlot()` already decides `attended` from whether the registrant has any `ExamAttempt` row — this route no longer opens the exam or touches Discord, since access is the registration itself on an ACTIVE slot, not this flag.

**Auth:** `MANAGE_EXAMS`

**Request:** JSON body: `attendance` (object, required — map of `userId` → `boolean`).

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"attendance":{"cmeuser2":true,"cmeuser3":false}}' \
  "$HSR_URL/api/admin/exam-slots/cmeslot1/attendance"
```

**Response:**
```json
{ "ok": true }
```
Error: `400` "attendance object required".

### `POST /api/admin/exam-slots/[id]/start`

**Why:** Opens a slot by hand, calling the exact `startExamSlot()` the closing-exam vote's cron calls at the scheduled hour — so a slot an admin opened and one the clock opened behave identically to candidates.

**Auth:** `MANAGE_EXAMS`

**Request:** Path param `id` (slot id). No body.

```bash
curl -sS -X POST -b cookies.txt "$HSR_URL/api/admin/exam-slots/cmeslot1/start"
```

**Response:**
```json
{ "ok": true, "notified": 5 }
```
Errors: `404` "Not found", `400` "Slot not in SCHEDULED state".

### `POST /api/admin/exam-slots/[id]/end`

**Why:** Force-stops a slot before its 1-hour window elapses, running the exact same `closeExamSlot()` the `exam-slot-close` cron runs automatically — no-shows are penalised and attendees left to the exam's own grading either way.

**Auth:** `MANAGE_EXAMS`

**Request:** Path param `id` (slot id). No body.

```bash
curl -sS -X POST -b cookies.txt "$HSR_URL/api/admin/exam-slots/cmeslot1/end"
```

**Response:**
```json
{ "ok": true, "attended": 4, "noShow": 1 }
```
`{ "ok": true }` alone if the slot was already closed. Error: `404` "Not found".

### `GET /api/admin/exams`

**Why:** Lists every exam for the admin panel's exam section. An application-form-only reviewer (holding `MANAGE_APPLICATION_FORMS` but not `MANAGE_EXAMS`) sees only `isApplicationForm` exams, so the list stays empty of rows they cannot open rather than full of rows that 404.

**Auth:** `canAccessExamSection()` — `MANAGE_EXAMS` OR `MANAGE_APPLICATION_FORMS`.

**Request:** No parameters. Response carries header `X-Application-Forms-Only: "1" | "0"` telling the client which scope applied.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/exams"
```

**Response:**
```json
[
  {
    "id": "cmeexam1",
    "title": "Driver closing exam",
    "audience": "CLOSING_EXAM",
    "trainingBranch": "DRIVER",
    "isOpen": false,
    "isApplicationForm": false,
    "_count": { "questions": 12, "attempts": 30 }
  }
]
```

### `POST /api/admin/exams`

**Why:** Creates a new exam or application form. The `isApplicationForm` flag is decided server-side — a reviewer holding only `MANAGE_APPLICATION_FORMS` cannot create a regular exam, nor an application form they could not later see. A `CLOSING_EXAM` audience requires a training branch, since the slot ping, invite emails and the 3-strikes removal all key off it.

**Auth:** `canAccessExamSection()` — `MANAGE_EXAMS` OR `MANAGE_APPLICATION_FORMS`.

**Request:** JSON body: `title` (string, required), `description` (string, optional), `timeLimitMode` (`"PER_QUESTION"` | `"TOTAL"`, required), `timeLimitSeconds` (number, required, parsed as int), `passingScore` (number 0-100, required), `audience` (one of `ALL`/`TRAINEE_DRIVER`/`TRAINEE_DISPATCHER`/`SELECTED_USERS`/`CLOSING_EXAM`, default `"ALL"`), `allowedUserIds` (array, required non-empty if `audience === "SELECTED_USERS"`), `exitIsViolation` (boolean, default `true`), `applicationFormType` (string, optional), `randomQuestionCount` (number, optional), `trainingBranch` (string, required if `audience === "CLOSING_EXAM"`), `isApplicationForm` (boolean — ignored/forced `true` for application-form-only callers).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"title":"Driver closing exam","timeLimitMode":"TOTAL","timeLimitSeconds":3600,"passingScore":80,"audience":"CLOSING_EXAM","trainingBranch":"DRIVER"}' \
  "$HSR_URL/api/admin/exams"
```

**Response (201):**
```json
{
  "id": "cmeexam1",
  "title": "Driver closing exam",
  "audience": "CLOSING_EXAM",
  "trainingBranch": "DRIVER",
  "isOpen": false,
  "isApplicationForm": false,
  "createdById": "cmeuser1"
}
```
Errors: `400` "Missing required fields" / "Invalid audience" / "Invalid time limit mode" / "Passing score must be between 0 and 100" / "Select at least one user for selected users audience" / "Closing exam requires a training branch".

### `GET /api/admin/exams/users`

**Why:** Backs the "selected users" audience picker (used both for regular exams and application forms) with a searchable/id-lookup user list.

**Auth:** `canAccessExamSection()` — `MANAGE_EXAMS` OR `MANAGE_APPLICATION_FORMS`.

**Request:** Query params, both optional: `search` (string, matched against `name`/`displayName`/`robloxId`/`discordUsername`), `ids` (comma-separated user id list — takes priority over `search` when present).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/exams/users?search=kov"
```

**Response:**
```json
{
  "users": [
    { "id": "cmeuser2", "name": "kovacs", "displayName": "Kovács János", "image": "https://...", "discordRole": "Trainee driver" }
  ]
}
```
Error: `500` "Failed to search users".

### `GET /api/admin/exams/folders`

**Why:** Lists exam folders in display order. Folders are pure filing with no access control of their own — anyone who may open the exam section may file in them; per-exam access is still enforced separately.

**Auth:** `requireExamSectionAccess()` — `canAccessExamSection()`.

**Request:** No parameters.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/exams/folders"
```

**Response:**
```json
[
  { "id": "cmefolder1", "name": "Driver", "color": "#4f46e5", "order": 0, "createdById": "cmeuser1" }
]
```

### `POST /api/admin/exams/folders`

**Why:** Creates a new folder, appended to the end of the display order so a new folder never jumps the queue an admin has already arranged.

**Auth:** `requireExamSectionAccess()` — `canAccessExamSection()`.

**Request:** JSON body: `name` (string, required, trimmed, max 80 chars), `color` (string, optional, max 16 chars).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"name":"Driver","color":"#4f46e5"}' \
  "$HSR_URL/api/admin/exams/folders"
```

**Response (201):**
```json
{ "id": "cmefolder1", "name": "Driver", "color": "#4f46e5", "order": 0, "createdById": "cmeuser1" }
```
Errors: `400` "Folder name is required" / "Folder name must be at most 80 characters".

### `PATCH /api/admin/exams/folders/[id]`

**Why:** Renames, recolours or manually reorders one folder.

**Auth:** `requireExamSectionAccess()` — `canAccessExamSection()`.

**Request:** Path param `id`. JSON body, all optional: `name` (string, non-empty, max 80 chars), `color` (string or null, max 16 chars), `order` (integer).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"name":"Drivers"}' \
  "$HSR_URL/api/admin/exams/folders/cmefolder1"
```

**Response:**
```json
{ "id": "cmefolder1", "name": "Drivers", "color": "#4f46e5", "order": 0, "createdById": "cmeuser1" }
```
Errors: `404` "Folder not found", `400` "Folder name is required" / "Folder name must be at most 80 characters" / "order must be an integer".

### `DELETE /api/admin/exams/folders/[id]`

**Why:** Deletes a folder. The exams inside are not deleted with it — the schema's `onDelete: SetNull` puts them back in the unfiled list — so this is tidying, never a way to lose an exam.

**Auth:** `requireExamSectionAccess()` — `canAccessExamSection()`.

**Request:** Path param `id`.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/exams/folders/cmefolder1"
```

**Response:**
```json
{ "success": true, "unfiled": 3 }
```
Error: `404` "Folder not found".

### `GET /api/admin/exams/[id]`

**Why:** Fetches one exam with its questions, for the exam editor.

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` if the exam is an application form (otherwise `404`, not `403` — to that caller the exam does not exist).

**Request:** Path param `id`.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/exams/cmeexam1"
```

**Response:**
```json
{
  "id": "cmeexam1",
  "title": "Driver closing exam",
  "audience": "CLOSING_EXAM",
  "questions": [{ "id": "cmeq1", "type": "MULTIPLE_CHOICE", "order": 0, "questionText": "...", "options": ["A", "B"], "correctAnswer": "A", "points": 1 }],
  "_count": { "attempts": 30 }
}
```
Error: `404` "Exam not found".

### `PUT /api/admin/exams/[id]`

**Why:** Full update of an exam's settings. Clearing `isApplicationForm` is restricted to full exam management, since it would move the exam out of an application-form-only caller's own reach.

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` for its own application forms.

**Request:** Path param `id`. JSON body, all optional except validated combinations: `title`, `description`, `timeLimitMode` (`"PER_QUESTION"`|`"TOTAL"`), `timeLimitSeconds` (positive integer), `passingScore` (0-100), `audience` (one of the valid audiences), `allowedUserIds` (required non-empty array if audience is `SELECTED_USERS`), `exitIsViolation` (boolean), `isApplicationForm` (boolean, only honoured for `MANAGE_EXAMS` callers), `applicationFormType`, `randomQuestionCount`, `trainingBranch` (falls back to the exam's stored branch if omitted; required if the effective audience is `CLOSING_EXAM`).

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"passingScore":85}' \
  "$HSR_URL/api/admin/exams/cmeexam1"
```

**Response:**
```json
{ "id": "cmeexam1", "title": "Driver closing exam", "passingScore": 85, "audience": "CLOSING_EXAM", "trainingBranch": "DRIVER" }
```
Errors: `404` "Exam not found", `400` "Invalid audience" / "Passing score must be between 0 and 100" / "timeLimitSeconds must be a positive integer" / "Select at least one user for selected users audience" / "Closing exam requires a training branch".

### `PATCH /api/admin/exams/[id]`

**Why:** Moves the exam into a folder (or out of one). Kept separate from `PUT` because filing is a one-field drag on the list page, and sending the whole exam back to move it would overwrite whatever somebody else changed concurrently.

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` for its own application forms.

**Request:** Path param `id`. JSON body: `folderId` (string or `null`/`""` to unfile — the `folderId` key must be present).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"folderId":"cmefolder1"}' \
  "$HSR_URL/api/admin/exams/cmeexam1"
```

**Response:**
```json
{ "id": "cmeexam1", "folderId": "cmefolder1" }
```
Errors: `400` "Nothing to update" (missing `folderId` key), `404` "Folder not found".

### `DELETE /api/admin/exams/[id]`

**Why:** Deletes the exam, cascading to its time slots and their registrations. Every candidate registered on a not-yet-sat slot is mailed `examSlotCancelled` before the delete runs — afterwards there is nobody left to name.

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` for its own application forms.

**Request:** Path param `id`.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/exams/cmeexam1"
```

**Response:**
```json
{ "success": true, "notified": 5 }
```
Error: `404` "Exam not found".

### `GET /api/admin/exams/[id]/stream`

**Why:** Live exam monitoring — active attempts, question progress and violations as they happen, so proctoring does not require polling.

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` for its own application forms.

**Request:** Path param `id`. No query params.

```bash
curl -sS -N -b cookies.txt "$HSR_URL/api/admin/exams/cmeexam1/stream"
```

**Response:** SSE stream (30s heartbeat). First event is `type: "initial"` with every in-progress attempt and stats; subsequent events fire on exam activity:
```
data: {"type":"initial","activeAttempts":[{"id":"cmeattempt1","user":{"id":"cmeuser2","name":"kovacs","displayName":"Kovács János","image":null},"currentQuestionIndex":3,"totalQuestions":12,"startedAt":"2026-09-13T09:00:00.000Z","answeredCount":3}],"stats":{"totalQuestions":12,"activeCount":1,"completedCount":29}}

data: {"type":"answer_submitted","attemptId":"cmeattempt1","questionIndex":4,"totalQuestions":12}

data: {"type":"student_completed","attemptId":"cmeattempt1","userId":"cmeuser2","score":91.7,"passed":true}

data: {"type":"violation","attemptId":"cmeattempt1","userId":"cmeuser2","reason":"TAB_SWITCH"}

data: {"type":"force_stopped","attemptId":"cmeattempt1"}
```

### `POST /api/admin/exams/[id]/toggle`

**Why:** Opens or closes an exam for new attempts. Closing does not end attempts already in progress — it counts them so the admin knows how many can still finish. A `CLOSING_EXAM` never uses this: it has no open/close toggle on `/admin/exams` because Start/End (or the vote's cron) is its whole lifecycle — see "A closing exam therefore carries no run buttons on /admin/exams".

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` for its own application forms.

**Request:** Path param `id`. No body.

```bash
curl -sS -X POST -b cookies.txt "$HSR_URL/api/admin/exams/cmeexam1/toggle"
```

**Response:**
```json
{ "isOpen": true, "activeAttempts": 0 }
```
Errors: `404` "Exam not found", `400` "Cannot open an exam with no questions".

### `GET /api/admin/exams/[id]/results`

**Why:** Loads every non-archived and archived attempt with full answers, for the results/grading page.

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` for its own application forms.

**Request:** Path param `id`.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/exams/cmeexam1/results"
```

**Response:**
```json
{
  "exam": { "id": "cmeexam1", "title": "Driver closing exam", "questions": [...], "hasPendingEmails": true },
  "attempts": [
    {
      "id": "cmeattempt1",
      "status": "COMPLETED",
      "score": 91.7,
      "passed": true,
      "user": { "id": "cmeuser2", "name": "kovacs", "displayName": "Kovács János", "image": null, "robloxId": "123" },
      "answers": [{ "questionId": "cmeq1", "isCorrect": true, "pointsAwarded": 1, "question": { "questionText": "..." } }]
    }
  ],
  "archivedAttempts": []
}
```
Error: `404` "Exam not found".

### `PATCH /api/admin/exams/[id]/results`

**Why:** Restores an archived attempt back onto the live results list.

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` for its own application forms.

**Request:** Path param `id`. JSON body: `attemptId` (string, required — must be an archived attempt of this exam).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"attemptId":"cmeattempt1"}' \
  "$HSR_URL/api/admin/exams/cmeexam1/results"
```

**Response:**
```json
{ "success": true }
```
Errors: `400` "Missing attemptId", `404` "Archived attempt not found".

### `DELETE /api/admin/exams/[id]/results`

**Why:** Archives (soft) or permanently deletes one attempt (and its answers, if permanent).

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` for its own application forms.

**Request:** Path param `id`. JSON body: `attemptId` (string, required), `permanent` (boolean, optional — `true` hard-deletes the attempt and its answers, otherwise it is archived).

```bash
curl -sS -X DELETE -b cookies.txt -H "Content-Type: application/json" \
  -d '{"attemptId":"cmeattempt1","permanent":false}' \
  "$HSR_URL/api/admin/exams/cmeexam1/results"
```

**Response:**
```json
{ "success": true }
```
Errors: `400` "Missing attemptId", `404` "Attempt not found".

### `POST /api/admin/exams/[id]/send-results`

**Why:** Queues result processing for every finished-but-unemailed attempt as one `AdmissionJob` per candidate, rather than emailing/pulling/ranking everyone inside the request — the old version ran past the proxy timeout on a 30-applicant form. The worker (kicked here, and re-run by `/api/cron/admission-queue`) does the actual work per candidate. **A `CLOSING_EXAM` still accepts this call to (re-)queue a stuck job, but the button is hidden on `/admin/exams` for that audience** — the result email fires automatically off `sendClosingExamResultEmailIfNeeded()` when an attempt is graded, and this route's `resultEmail` step is recorded as already-done for closing exams so it never sends a second, blander copy.

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` for its own application forms.

**Request:** Path param `id`. No body. Refuses if any matching attempt has `passed` still `null` (not fully reviewed), and is cooled down per-exam (`checkEmailCooldown`).

```bash
curl -sS -X POST -b cookies.txt "$HSR_URL/api/admin/exams/cmeexam1/send-results"
```

**Response:**
```json
{
  "message": "Queued 12 candidate(s). Emails and ranks are being processed in the background.",
  "queued": 12,
  "alreadyQueued": 0,
  "results": [{ "userId": "cmeuser2", "passed": true }]
}
```
Errors: `404` "Exam not found", `400` "No new attempts to send results for" / "`N` attempt(s) not yet fully reviewed", `429` "Cooldown active" with `remainingSeconds`.

### `GET /api/admin/exams/[id]/send-results`

**Why:** Reports admission-queue progress for this exam (per-status counts, and the failed jobs' errors), so the results page can show what work remains after a send.

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` for its own application forms.

**Request:** Path param `id`.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/exams/cmeexam1/send-results"
```

**Response:**
```json
{
  "PENDING": 2,
  "RUNNING": 0,
  "DONE": 9,
  "FAILED": 1,
  "failedJobs": [{ "id": "cmejob1", "userId": "cmeuser3", "lastError": "Discord pull failed: 404", "runs": 5 }]
}
```

### `POST /api/admin/exams/[id]/upload`

**Why:** Uploads an image (for an IMAGE-type question) to Cloudflare R2, returning the public CDN URL to attach to a question.

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` for its own application forms.

**Request:** Path param `id`. Multipart form data: `file` (required — image file, allowed types `image/jpeg`, `image/png`, `image/gif`, `image/webp`; max size 10MB).

```bash
curl -sS -X POST -b cookies.txt \
  -F "file=@question-image.png;type=image/png" \
  "$HSR_URL/api/admin/exams/cmeexam1/upload"
```

**Response:**
```json
{ "url": "https://s3.hsrgames.hu/exam-images/cmeexam1/6e2b....png" }
```
Errors: `500` "R2 storage not configured", `400` "No file provided" / "Invalid file type. Allowed: JPEG, PNG, GIF, WebP" / "File too large. Maximum 10MB".

### `POST /api/admin/exams/[id]/questions`

**Why:** Adds one question to an exam, appended after the current highest `order`.

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` for its own application forms.

**Request:** Path param `id`. JSON body: `type` (`"MULTIPLE_CHOICE"`|`"TEXT"`|`"IMAGE"`, required), `questionText` (string, required), `imageUrl` (string, optional), `options` (required for `MULTIPLE_CHOICE`), `correctAnswer` (required for `MULTIPLE_CHOICE`), `points` (number, optional, default 1), `timeLimitSeconds` (number, optional).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"type":"MULTIPLE_CHOICE","questionText":"Mi a piros jelzés jelentése?","options":["Megállj","Szabad"],"correctAnswer":"Megállj","points":2}' \
  "$HSR_URL/api/admin/exams/cmeexam1/questions"
```

**Response (201):**
```json
{ "id": "cmeq2", "examId": "cmeexam1", "type": "MULTIPLE_CHOICE", "questionText": "Mi a piros jelzés jelentése?", "order": 12, "points": 2 }
```
Errors: `400` "Missing required fields" / "Invalid question type" / "Multiple choice questions require options and correct answer", `404` "Exam not found".

### `PUT /api/admin/exams/[id]/questions/[qid]`

**Why:** Updates one question's fields.

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` for its own application forms.

**Request:** Path params `id`, `qid`. JSON body, all optional: `type`, `questionText`, `imageUrl`, `options`, `correctAnswer`, `points` (parsed int), `order` (parsed int), `timeLimitSeconds`.

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"points":3}' \
  "$HSR_URL/api/admin/exams/cmeexam1/questions/cmeq2"
```

**Response:**
```json
{ "id": "cmeq2", "examId": "cmeexam1", "points": 3 }
```
Error: `404` "Question not found".

### `DELETE /api/admin/exams/[id]/questions/[qid]`

**Why:** Removes one question and re-sequences the `order` of the remaining questions so there is no gap.

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` for its own application forms.

**Request:** Path params `id`, `qid`.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/exams/cmeexam1/questions/cmeq2"
```

**Response:**
```json
{ "success": true }
```
Error: `404` "Question not found".

### `POST /api/admin/exams/[id]/stop/[attemptId]`

**Why:** Force-stops an in-progress attempt from the live monitor (a proctoring action), scoring it 0 and failed, and broadcasting `force_stopped` on the exam's SSE stream.

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` for its own application forms.

**Request:** Path params `id`, `attemptId` (must currently be `IN_PROGRESS`).

```bash
curl -sS -X POST -b cookies.txt "$HSR_URL/api/admin/exams/cmeexam1/stop/cmeattempt1"
```

**Response:**
```json
{ "success": true }
```
Error: `404` "Active attempt not found".

### `POST /api/admin/exams/[id]/results/[attemptId]/review`

**Why:** Manually grades one TEXT/IMAGE answer, recalculates the attempt's total score, and — once every answer on the attempt has been reviewed — sets `passed` and, for a `CLOSING_EXAM`, calls `finalizeClosingExamAttempt()` to fire the result email and queue rank grants. This is the route that closes the loop the automatic MC-scored path handles instantly.

**Auth:** `requireExamAccess(id)` — `MANAGE_EXAMS`, or `MANAGE_APPLICATION_FORMS` for its own application forms.

**Request:** Path params `id`, `attemptId`. JSON body: `questionId` (string, required), `pointsAwarded` (number, required, parsed as float), `isCorrect` (boolean, required).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"questionId":"cmeq3","pointsAwarded":2,"isCorrect":true}' \
  "$HSR_URL/api/admin/exams/cmeexam1/results/cmeattempt1/review"
```

**Response:**
```json
{ "score": 91.7, "allReviewed": true, "passed": true }
```
`passed` is `null` while other TEXT/IMAGE answers on the attempt are still unreviewed. Errors: `400` "Missing required fields", `404` "Attempt not found".

## Admin: Game Stats, Instructors, Locomotive Pool, Maintenance, Perf Stats, Permissions

### `GET /api/admin/game-stats`

**Why:** Feeds the admin game-statistics dashboard: 30-day unique daily player counts, aggregate/average playtime, who's online right now, and the top-15 most active players by playtime.

**Auth:** `canViewAdminPanel()` — note this is the general admin-panel gate, not `VIEW_STATISTICS` (that flag guards `/admin/statistics`, a separate community-stats page). Any admin-panel viewer can read this.

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/game-stats"
```

**Response:**
```json
{
  "dailyPlayers": [{ "date": "2026-08-15", "players": 42 }, "... 30 entries"],
  "playtime": {
    "totalSeconds": 48213200,
    "averageSeconds": 61234,
    "totalPlayers": 785,
    "onlineNow": 12
  },
  "topPlayers": [
    {
      "id": "clx123",
      "name": "SomeRobloxName",
      "displayName": null,
      "image": "https://tr.rbxcdn.com/...",
      "discordRole": "Train driver",
      "isOnline": true,
      "totalPlayTime": 512340,
      "totalSessions": 380,
      "level": 24,
      "xp": 91234
    }
  ]
}
```
Errors: `401 Unauthorized` (no session / user not found), `403 Forbidden`, `500` on failure.

### `GET /api/admin/instructors`

**Why:** Powers `/admin/trainings` → Instructors tab: the live roster of every instructor-rank holder, their weekly credit/session progress against the quota, and which grades they hold — see CLAUDE.md's "Instructor Grades & Weekly Duties" section.

**Auth:** `MANAGE_INSTRUCTORS` (`canManageInstructors()`).

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/instructors"
```

**Response:**
```json
{
  "instructors": [
    {
      "id": "clx1",
      "name": "RobloxName",
      "displayName": "Display Name",
      "discordEmail": "user@example.com",
      "locale": "hu",
      "immune": false,
      "instructorRoleIds": ["1111111111111111"],
      "grades": ["INSTRUCTOR", "TYPE_INSTRUCTOR"],
      "credits": 3,
      "sessions": 2
    }
  ],
  "rules": {
    "requiredCredits": 4,
    "requiredSessions": 2
  }
}
```
Error: `500 "Failed to load instructors"` on failure. Exact shape of each instructor row comes from `buildInstructorOverview()`.

### `GET /api/admin/instructors/tracks`

**Why:** Reads the current on/off state of each supervision branch (driver/dispatcher/etc.) — the per-programme `supervisionEnabled` switch surfaced on `/admin/trainings` → Settings.

**Auth:** `MANAGE_INSTRUCTORS` (`canManageInstructors()`).

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/instructors/tracks"
```

**Response:** shape of `getTrackConfig()`, e.g.
```json
{
  "tracks": [
    { "track": "DRIVER", "enabled": true },
    { "track": "DISPATCHER", "enabled": true }
  ]
}
```

### `POST /api/admin/instructors/tracks`

**Why:** Opens or closes a supervision branch. Turning a branch on is what starts issuing warnings/deadlines for trainees on it; turning it off stops trainees on that branch from submitting supervisions.

**Auth:** `MANAGE_INSTRUCTORS` (`canManageInstructors()`).

**Request:** JSON body: `track` (string, must be one of the keys returned by `getSupervisionTracks()`, required), `enabled` (boolean, required).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"track":"DISPATCHER","enabled":false}' \
  "$HSR_URL/api/admin/instructors/tracks"
```

**Response:** the updated `getTrackConfig()` result, same shape as the GET above. `400` if `track`/`enabled` are missing or invalid: `{"error":"track (one of: DRIVER, DISPATCHER) and enabled (boolean) are required"}`. Writes `SUPERVISION_TRACK_ENABLED`/`SUPERVISION_TRACK_DISABLED` to `ActivityLog`.

### `GET /api/admin/instructors/terminated`

**Why:** Lists instructors who lost their Instructor role for failing the weekly quota, so an admin can review and potentially revoke the termination — the "Terminated" list on `/admin/trainings` → Instructors.

**Auth:** `MANAGE_INSTRUCTORS` (`canManageInstructors()`).

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/instructors/terminated"
```

**Response:**
```json
{
  "rows": [
    {
      "id": "strike_id",
      "userId": "clx1",
      "displayName": "Display Name",
      "weekStart": "2026-09-07T00:00:00.000Z",
      "credits": 1,
      "requiredCredits": 4,
      "sessions": 0,
      "manual": true,
      "terminatedAt": "2026-09-13T09:00:00.000Z",
      "terminatedById": "clxAdmin",
      "removedRoles": ["1111111111111111"],
      "note": "No sessions all week"
    }
  ]
}
```
Error: `500 "Failed to load terminated instructors"`.

### `POST /api/admin/instructors/[id]/revoke-strike`

**Why:** Clears an open (non-terminated) weekly-quota warning — the admin deciding the warning should not have stood, distinct from an appeal approval which stays on record as evidence.

**Auth:** `MANAGE_INSTRUCTORS` (`canManageInstructors()`).

**Request:** Path param `id` (instructor's user id). JSON body: `note` (string, optional).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"note":"Was on approved leave"}' \
  "$HSR_URL/api/admin/instructors/clx1/revoke-strike"
```

**Response:** `{ "success": true }`. `404 "This instructor has no open warning"` if there is no open `InstructorStrike` with `result: "WARNING"`. Clears `terminationDue`/`terminationDueAt`, writes `INSTRUCTOR_STRIKE_REVOKED` to `ActivityLog`, and emails the instructor with template `instructorWarningRevoked` if `discordEmail` is set.

### `POST /api/admin/instructors/[id]/warn`

**Why:** Manual warning issued from the admin page, writing the same `InstructorStrike` row the automatic Monday quota check would write — so the next automatic run escalates from it instead of starting over.

**Auth:** `MANAGE_INSTRUCTORS` (`canManageInstructors()`).

**Request:** Path param `id`. JSON body: `note` (string, optional).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"note":"Missed both sessions this week"}' \
  "$HSR_URL/api/admin/instructors/clx1/warn"
```

**Response:** `{ "success": true, "strikeId": "strike_id" }`. `404 "Instructor not found"` if not currently in `listInstructors()`. `409 "This instructor already has an open warning"` if one already exists. Creates an `InstructorStrike` row (`result: "WARNING"`, `manual: true`), writes `INSTRUCTOR_STRIKE_ISSUED` to `ActivityLog`, and emails template `instructorQuotaWarning` if `discordEmail` is set.

### `POST /api/admin/instructors/[id]/terminate`

**Why:** Removes the Instructor Discord role(s) for an unmet weekly quota. Only instructor roles are touched — driver/dispatcher/staff roles are untouched, since failing the teaching requirement says nothing about the rest of the member's work.

**Auth:** `MANAGE_INSTRUCTORS` (`canManageInstructors()`).

**Request:** Path param `id`. JSON body: `confirm` (boolean, **must be `true`**, required), `note` (string, optional). Refuses `id === session.id` (cannot terminate yourself) and refuses instructors flagged `immune` (site admins are exempt from the quota).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"confirm":true,"note":"3 weeks under quota"}' \
  "$HSR_URL/api/admin/instructors/clx1/terminate"
```

**Response:** `{ "success": true, "strikeId": "strike_id" }`.
Errors: `400 "confirm must be true"`, `400 "You cannot terminate yourself"`, `404 "Instructor not found"`, `400 "Website admins are exempt from the quota and cannot be terminated here"`, `502 "Failed to remove an instructor role on Discord — nothing was changed"` (Discord call fails — nothing is written to the DB in that case, Discord is attempted first). On success: removes every role in `instructor.instructorRoleIds` via Discord, upserts an `InstructorStrike` (`result: "TERMINATION"`), closes any open WARNING strike, logs `INSTRUCTOR_TERMINATED`, and emails template `instructorTermination` if `discordEmail` is set.

### `POST /api/admin/instructors/[id]/revoke-termination`

**Why:** Reverses a termination: grants the Instructor Discord role(s) back and marks the termination strike cleared so `listTerminatedInstructors()` stops surfacing it.

**Auth:** `MANAGE_INSTRUCTORS` (`canManageInstructors()`).

**Request:** Path param `id`. JSON body: `note` (string, optional).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"note":"Appeal accepted"}' \
  "$HSR_URL/api/admin/instructors/clx1/revoke-termination"
```

**Response:** `{ "success": true }`. `404 "This instructor has no active termination to revoke"` if there is no cleared-false `TERMINATION` strike with `terminatedAt` set. `502 "Failed to grant an instructor role back on Discord — nothing was changed"` if the Discord grant fails (Discord attempted before any DB write). Restores exactly the roles recorded on the strike's `removedRoles` (falling back to the bare `INSTRUCTOR_ROLE_ID` for older rows with nothing recorded), clears the strike, logs `INSTRUCTOR_TERMINATION_REVOKED`, and emails template `instructorTerminationRevoked` last, after the role and DB state are already updated.

### `GET /api/admin/permissions/flags`

**Why:** Lists every `PermissionFlag` row grouped by category, for the `/admin/permissions` page to render the full flag table each role/user can be granted against — see CLAUDE.md's Permission System section ("Permission Flags (40 total)").

**Auth:** `MANAGE_PERMISSIONS` (`canManagePermissions()`), checked after `requireAuth()`.

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/permissions/flags"
```

**Response:**
```json
{
  "flags": [
    { "id": "flag1", "name": "MANAGE_TICKETS", "description": "...", "category": "admin" }
  ],
  "grouped": {
    "admin": [{ "id": "flag1", "name": "MANAGE_TICKETS", "description": "...", "category": "admin" }],
    "duty": ["..."]
  }
}
```

### `POST /api/admin/permissions/sync-discord-roles`

**Why:** Manually pulls the guild's current role list (names, colours, `position`, `managed`) into the `DiscordGuildRole` mirror — the "Sync now" action on `/admin/permissions`, used when a role was just renamed/recreated/reordered on Discord and the hierarchy shouldn't wait for the 60s cache or the stale-check to catch up.

**Auth:** `MANAGE_PERMISSIONS` (`canManagePermissions()`).

**Request:** No body.

```bash
curl -sS -X POST -b cookies.txt "$HSR_URL/api/admin/permissions/sync-discord-roles"
```

**Response:** `{ "synced": 47 }` (number of roles synced). `502 "Failed to fetch roles from Discord"` if `syncGuildRolesToDb()` returns null. Logs `SYNC_DISCORD_ROLES` to `ActivityLog`.

### `GET /api/admin/permissions/roles`

**Why:** The main data source for `/admin/permissions`: every Discord guild role with its derived rank priority/category, its permission flags, and every anchor's current resolution (see CLAUDE.md — "Rank order comes from Discord, not from a table in the code" and the anchor/`isRank` mechanics in detail). This is what lets the page flag a broken anchor or a decorative role sitting above a real rank.

**Auth:** `MANAGE_PERMISSIONS` (`canManagePermissions()`).

**Request:** No params. As a side effect, refreshes the guild-role mirror if stale (`syncGuildRolesIfStale()`) and invalidates the cached hierarchy before reading it.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/permissions/roles"
```

**Response:**
```json
{
  "roles": [
    {
      "discordRoleId": "1111111111111111",
      "roleName": "・Commanding Officer",
      "priority": 42,
      "category": "leadership",
      "color": 16711680,
      "secondaryColor": null,
      "tertiaryColor": null,
      "position": 42,
      "isSeparator": false,
      "managed": false,
      "isRankOverride": null,
      "isRank": true,
      "anchors": ["DIRECTOR_GENERAL"],
      "flags": ["MANAGE_USERS", "VIEW_ADMIN_PANEL"]
    }
  ],
  "anchors": [
    {
      "anchor": "DIRECTOR_GENERAL",
      "label": "Director General",
      "discordRoleId": "1111111111111111",
      "roleName": "・Commanding Officer",
      "priority": 42,
      "broken": false
    }
  ],
  "synced": true,
  "rankCount": 38
}
```

### `PUT /api/admin/permissions/roles`

**Why:** Sets the full list of permission flags a Discord role grants — the default flags row on `/admin/permissions` per role.

**Auth:** `MANAGE_PERMISSIONS` (`canManagePermissions()`).

**Request:** JSON body: `discordRoleId` (string, required), `flags` (string array of `PermissionFlag.name` values, required — validated to exist).

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"discordRoleId":"1111111111111111","flags":["MANAGE_TICKETS","VIEW_STATISTICS"]}' \
  "$HSR_URL/api/admin/permissions/roles"
```

**Response:** `{ "success": true }`. `400` if `discordRoleId`/`flags` missing or not an array, or `{"error":"Some flags are invalid"}` if any flag name doesn't exist. Replaces the role's `RolePermission` rows in a transaction, logs `PERMISSION_ROLE_UPDATED`.

### `PATCH /api/admin/permissions/roles`

**Why:** Sets the `isRank` override for a Discord role — including it in, or excluding it from, the rank hierarchy. This is the fix for a decorative role (colour/booster perk) that sits above a real rank and would otherwise be misread as that member's rank; excluding an anchor role here also switches its permission rule off for everyone (see CLAUDE.md's detailed note on this).

**Auth:** `MANAGE_PERMISSIONS` (`canManagePermissions()`).

**Request:** JSON body: `discordRoleId` (string, required), `isRank` (`true` | `false` | `null`, required — `null` returns to the automatic decision).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"discordRoleId":"2222222222222222","isRank":false}' \
  "$HSR_URL/api/admin/permissions/roles"
```

**Response:** `{ "success": true, "isRankOverride": false, "isRank": false }`. `400 "discordRoleId is required"`, `400 "isRank must be true, false or null"`, `404 "Unknown Discord role"`. Updates `DiscordGuildRole.isRank`, invalidates the cached hierarchy, logs `PERMISSION_ROLE_RANK_UPDATED`.

### `POST /api/admin/permissions/roles`

**Why:** Points a rank anchor (e.g. `DIRECTOR_GENERAL`) at a specific Discord role — the repair for a shipped default anchor id going stale when the guild renames or recreates that role. This is exactly the "Anchors are assignable" mechanism from CLAUDE.md's Permission System section.

**Auth:** `MANAGE_PERMISSIONS` (`canManagePermissions()`).

**Request:** JSON body: `anchor` (string, one of `ROLE_ANCHOR_KEYS`, required), `discordRoleId` (string or `null` — `null` resets the anchor to its built-in default role id, required key but nullable value).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"anchor":"DIRECTOR_GENERAL","discordRoleId":"1111111111111111"}' \
  "$HSR_URL/api/admin/permissions/roles"
```

**Response:** `{ "success": true }`. `400 "Unknown anchor"`, `400 "discordRoleId is required"` (when not explicitly `null`), `404 "Unknown Discord role"`, `400 "That role is outside the rank hierarchy — include it first"` (role fails `countsAsRank()` — assigning it would just re-create the priority-0 breakage this endpoint exists to fix). Upserts `RoleAnchorAssignment`, invalidates the hierarchy cache, logs `PERMISSION_ANCHOR_UPDATED`.

### `GET /api/admin/permissions/users`

**Why:** The searchable, paginated user list on `/admin/permissions` showing each member's effective permission flags and any per-user overrides.

**Auth:** `MANAGE_PERMISSIONS` (`canManagePermissions()`).

**Request:** Query params: `search` (string, optional — matches `name`/`displayName`/`discordUsername` via `contains`), `page` (number, default `1`), `limit` (number, default `20`).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/permissions/users?search=kovacs&page=1&limit=20"
```

**Response:**
```json
{
  "users": [
    {
      "id": "clx1",
      "name": "RobloxName",
      "displayName": "Kovács János",
      "image": "https://tr.rbxcdn.com/...",
      "discordRole": "Train driver",
      "role": "USER",
      "effectiveFlags": ["CAN_DRIVE", "SUBMIT_SCHEDULE_DRIVER"],
      "overrides": [{ "flag": "VIEW_TRACK_MAP", "type": "GRANT" }]
    }
  ],
  "total": 812,
  "page": 1,
  "totalPages": 41
}
```

### `GET /api/admin/permissions/users/[userId]`

**Why:** Detail view behind clicking a user on `/admin/permissions` — role-derived flags before overrides, effective flags after, and the override rows themselves (with reason/granter).

**Auth:** `MANAGE_PERMISSIONS` (`canManagePermissions()`).

**Request:** Path param `userId`.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/permissions/users/clx1"
```

**Response:**
```json
{
  "user": {
    "id": "clx1",
    "name": "RobloxName",
    "displayName": "Kovács János",
    "image": "https://tr.rbxcdn.com/...",
    "discordRole": "Train driver",
    "role": "USER"
  },
  "roleFlags": ["CAN_DRIVE"],
  "effectiveFlags": ["CAN_DRIVE", "VIEW_TRACK_MAP"],
  "overrides": [
    { "flag": "VIEW_TRACK_MAP", "type": "GRANT", "reason": "Track mapping volunteer", "grantedById": "clxAdmin" }
  ]
}
```
`404 "User not found"` for an unknown id.

### `PUT /api/admin/permissions/users/[userId]`

**Why:** Replaces a user's permission-override set (per-user GRANT/REVOKE on top of their role's flags) — how `/admin/permissions` grants an exceptional permission (e.g. `VIEW_TRACK_MAP`) or revokes one a rank would otherwise carry.

**Auth:** `MANAGE_PERMISSIONS` (`canManagePermissions()`).

**Request:** Path param `userId`. JSON body: `overrides` (array of `{ flag: string, type: "GRANT" | "REVOKE" }`, required — must be an array, empty array clears all overrides). Refuses targets with legacy `role === "ADMIN"`.

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"overrides":[{"flag":"VIEW_TRACK_MAP","type":"GRANT"}]}' \
  "$HSR_URL/api/admin/permissions/users/clx1"
```

**Response:** `{ "success": true }`. `400 "overrides array is required"`, `404 "User not found"`, `400 "Cannot set permission overrides for admin users"`, `400 "Invalid flag: <name>"` for any unknown flag name. Replaces `UserPermissionOverride` rows in a transaction, logs `PERMISSION_USER_OVERRIDE`.

### `DELETE /api/admin/permissions/users/[userId]`

**Why:** Clears every permission override for a user in one action — the "reset to role defaults" button.

**Auth:** `MANAGE_PERMISSIONS` (`canManagePermissions()`).

**Request:** Path param `userId`. No body.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/permissions/users/clx1"
```

**Response:** `{ "success": true }`. Deletes all `UserPermissionOverride` rows for the user, logs `PERMISSION_USER_OVERRIDE` ("Cleared all permission overrides for user ...").

### `GET /api/admin/locomotive-pool`

**Why:** The single data load behind `/admin/locomotive-pool`: pair groups (both the non-forda-only set for the locomotive/wagon tabs, and the full set including forda pairs for the Inspectors tab), locomotive and wagon chips with their assigned Discord role restrictions, existing pair↔chip assignments, live guild roles for the role picker, and the inspector rank→pair mapping plus the global inspector switches (see CLAUDE.md's Ticket Inspector Duty section).

**Auth:** `MANAGE_LOCOMOTIVE_POOL` (`canManageLocomotivePool()`).

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/locomotive-pool"
```

**Response:**
```json
{
  "pairGroups": [{ "pairGroup": "IC12", "outboundNumber": "912", "outboundType": "IC", "returnNumber": "913", "returnType": "IC" }],
  "chips": [{ "id": "chip1", "number": "V43 1234", "roleIds": ["111"], "imageUrl": "https://s3.hsrgames.hu/...", "matchOverride": null, "operator": "MAV" }],
  "assignments": [{ "id": "a1", "pairGroup": "IC12", "chipId": "chip1", "order": 0 }],
  "discordRoles": [{ "id": "111", "name": "IC Train Driver", "color": 3447003 }],
  "wagonChips": [{ "id": "w1", "number": "Bmpz", "imageUrl": null, "matchOverride": null, "operator": null }],
  "wagonAssignments": [{ "id": "wa1", "pairGroup": "IC12", "wagonId": "w1", "order": 0 }],
  "operators": ["MAV", "GYSEV"],
  "allPairGroups": [{ "pairGroup": "rjx5", "outboundNumber": "5", "outboundType": "RJX", "returnNumber": "6", "returnType": "RJX" }],
  "inspectorRoles": [{ "id": "ir1", "pairGroup": "IC12", "discordRoleId": "222" }],
  "inspectorSettings": { "enabled": true, "maxPerPair": 1 }
}
```

### `POST /api/admin/locomotive-pool/inspector-roles`

**Why:** Assigns an inspector rank to a service (`pairGroup`), or in bulk to every pair whose outbound train matches one of the given `trainTypes` — the drag-a-rank-onto-a-pair / "assign to a whole class of services" actions on the Inspectors tab (see CLAUDE.md's `InspectorPairRole` note: a pair with no rows is open to every inspector).

**Auth:** `MANAGE_LOCOMOTIVE_POOL` (`canManageLocomotivePool()`).

**Request:** JSON body: `discordRoleId` (string, required); either `pairGroup` (string, single pair) or `trainTypes` (string array of valid `TrainType` enum values, bulk mode — takes precedence if non-empty).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"discordRoleId":"222","trainTypes":["IC","RJX"]}' \
  "$HSR_URL/api/admin/locomotive-pool/inspector-roles"
```

**Response:** `201` with `{ "created": [{ "id": "ir2", "pairGroup": "IC12", "discordRoleId": "222" }, "..."] }`, or `{ "created": [], "skipped": 0 }` if no matching pairs. `400 "discordRoleId is required"`, `400 "pairGroup or trainTypes is required"`. Uses `createMany` + `skipDuplicates`, so re-running a bulk assign is a no-op on pairs that already carry the rank. Logs `INSPECTOR_PAIR_ROLE_ASSIGNED`.

### `DELETE /api/admin/locomotive-pool/inspector-roles`

**Why:** Removes one rank↔pair mapping, or every mapping for a given rank at once — undoing an inspector assignment.

**Auth:** `MANAGE_LOCOMOTIVE_POOL` (`canManageLocomotivePool()`).

**Request:** Query params (one required): `id` (string, the `InspectorPairRole` row id — removes that one pair mapping) or `discordRoleId` (string — removes every pair mapping for that rank).

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/locomotive-pool/inspector-roles?id=ir2"
```

**Response:** `{ "ok": true }` (by `id`) or `{ "ok": true, "removed": 4 }` (by `discordRoleId`). `400 "id or discordRoleId is required"` if neither given. Logs `INSPECTOR_PAIR_ROLE_REMOVED`.

### `PATCH /api/admin/locomotive-pool/inspector-roles`

**Why:** Toggles the two global inspector-duty switches: the master enable and `maxPerPair` (how many inspectors may book/ride one service) — CLAUDE.md's `InspectorSettings` singleton.

**Auth:** `MANAGE_LOCOMOTIVE_POOL` (`canManageLocomotivePool()`).

**Request:** JSON body: `enabled` (boolean, optional), `maxPerPair` (number, optional — clamped to 1–10).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"enabled":true,"maxPerPair":2}' \
  "$HSR_URL/api/admin/locomotive-pool/inspector-roles"
```

**Response:** `{ "enabled": true, "maxPerPair": 2 }`. Upserts `InspectorSettings` (id 1); logs `INSPECTOR_SETTINGS_UPDATED` only if at least one field changed.

### `POST /api/admin/locomotive-pool/assignments`

**Why:** Assigns a locomotive chip to a service pair — the "drop event" when an admin drags a chip onto a pair on the locomotive board.

**Auth:** `MANAGE_LOCOMOTIVE_POOL` (`canManageLocomotivePool()`).

**Request:** JSON body: `pairGroup` (string, required), `chipId` (string, required).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"pairGroup":"IC12","chipId":"chip1"}' \
  "$HSR_URL/api/admin/locomotive-pool/assignments"
```

**Response:** `201` with `{ "id": "a2", "pairGroup": "IC12", "chipId": "chip1", "order": 1 }`. `400` if either field missing, `404 "Chip not found"`. Idempotent via `upsert` on the `pairGroup_chipId` unique constraint. Logs `LOCOMOTIVE_ASSIGNMENT_CREATED`.

### `POST /api/admin/locomotive-pool/chips`

**Why:** Creates a new locomotive chip (a locomotive/track number the admin can drag onto pairs).

**Auth:** `MANAGE_LOCOMOTIVE_POOL` (`canManageLocomotivePool()`).

**Request:** JSON body: `number` (string, required, must be unique).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"number":"V43 1234"}' \
  "$HSR_URL/api/admin/locomotive-pool/chips"
```

**Response:** `201` with `{ "id": "chip2", "number": "V43 1234", "roleIds": [], "imageUrl": null }`. `400 "number is required"`, `409 "This locomotive number already exists"`. Logs `LOCOMOTIVE_CHIP_CREATED`.

### `DELETE /api/admin/locomotive-pool/assignments/[id]`

**Why:** Removes a single chip↔pair assignment (does not delete the chip itself) — dragging a chip off a pair.

**Auth:** `MANAGE_LOCOMOTIVE_POOL` (`canManageLocomotivePool()`).

**Request:** Path param `id` (assignment id).

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/locomotive-pool/assignments/a2"
```

**Response:** `{ "success": true }`. `500 "Failed to delete locomotive assignment"` if the assignment id doesn't exist (Prisma `delete` throws). Logs `LOCOMOTIVE_ASSIGNMENT_DELETED` naming the chip number and pair.

### `PATCH /api/admin/locomotive-pool/chips/[id]`

**Why:** Sets/clears a chip's custom in-game match text (`matchOverride`, used when the chip's display number doesn't match the in-game locomotive name) and/or its operator (MÁV/GYSEV badge).

**Auth:** `MANAGE_LOCOMOTIVE_POOL` (`canManageLocomotivePool()`).

**Request:** Path param `id`. JSON body (both optional, only applied `if "<field>" in body`): `matchOverride` (string or `null` — empty/whitespace-only coerces to `null`), `operator` (must be exactly `"MAV"` or `"GYSEV"`, anything else coerces to `null`).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"matchOverride":"V43-1234","operator":"MAV"}' \
  "$HSR_URL/api/admin/locomotive-pool/chips/chip1"
```

**Response:** `{ "id": "chip1", "matchOverride": "V43-1234", "operator": "MAV" }`. `500` if the chip id doesn't exist. Logs `LOCOMOTIVE_CHIP_UPDATED`.

### `DELETE /api/admin/locomotive-pool/chips/[id]`

**Why:** Removes a chip entirely from the pool, cascading its pair assignments and role restrictions.

**Auth:** `MANAGE_LOCOMOTIVE_POOL` (`canManageLocomotivePool()`).

**Request:** Path param `id`.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/locomotive-pool/chips/chip1"
```

**Response:** `{ "success": true }`. `500` if the chip doesn't exist. Logs `LOCOMOTIVE_CHIP_DELETED`.

### `PUT /api/admin/locomotive-pool/chips/[id]/roles`

**Why:** Overwrites the full set of Discord roles required to drive a chip's locomotive — an empty array means no restriction (anyone may draw it).

**Auth:** `MANAGE_LOCOMOTIVE_POOL` (`canManageLocomotivePool()`).

**Request:** Path param `id`. JSON body: `roleIds` (string array, required — non-string entries filtered out, missing/invalid treated as `[]`).

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"roleIds":["111","222"]}' \
  "$HSR_URL/api/admin/locomotive-pool/chips/chip1/roles"
```

**Response:** `{ "id": "chip1", "roleIds": ["111", "222"] }`. `404 "Chip not found"`. Deletes and recreates `LocomotiveChipRole` rows in a transaction. Logs `LOCOMOTIVE_CHIP_ROLES_UPDATED`.

### `POST /api/admin/locomotive-pool/chips/[id]/image`

**Why:** Uploads/replaces the photo shown on a locomotive pool chip, stored in Cloudflare R2.

**Auth:** `MANAGE_LOCOMOTIVE_POOL` (`canManageLocomotivePool()`).

**Request:** Path param `id`. `multipart/form-data` with field `file` (required) — allowed types `image/jpeg`, `image/png`, `image/gif`, `image/webp`; max 10MB.

```bash
curl -sS -X POST -b cookies.txt \
  -F "file=@locomotive.jpg;type=image/jpeg" \
  "$HSR_URL/api/admin/locomotive-pool/chips/chip1/image"
```

**Response:** `{ "imageUrl": "https://s3.hsrgames.hu/locomotive-pool/chip1/<uuid>.jpg" }`. `500 "R2 storage not configured"` if R2 env vars aren't set, `404 "Chip not found"`, `400 "No file provided"`, `400 "Invalid file type. Allowed: JPEG, PNG, GIF, WebP"`, `400 "File too large. Maximum 10MB"`. Writes the object to R2 under a random UUID key, updates `LocomotiveChip.imageUrl`, logs `LOCOMOTIVE_CHIP_IMAGE_UPDATED`.

### `GET /api/admin/maintenance`

**Why:** Reads the current site-wide maintenance-mode state (on/off + message) for the admin toggle UI.

**Auth:** `canViewAdminPanel()` — any admin-panel viewer may read the state (lower bar than toggling it).

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/maintenance"
```

**Response:** `{ "enabled": false, "message": null }`.

### `PUT /api/admin/maintenance`

**Why:** Toggles site-wide maintenance mode, deliberately gated higher than the `MAINTENANCE_BYPASS` flag itself — only site admins may flip the switch (bypassing maintenance once it's on is the separate, lower-bar flag).

**Auth:** `isSiteAdmin()` — a stricter check than a permission flag; note this is not `MAINTENANCE_BYPASS`.

**Request:** JSON body: `enabled` (boolean, coerced), `message` (string, optional — trimmed, empty string becomes `null`).

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"enabled":true,"message":"Deploying a database migration, back in 10 minutes."}' \
  "$HSR_URL/api/admin/maintenance"
```

**Response:** `{ "enabled": true, "message": "Deploying a database migration, back in 10 minutes." }`. Upserts `MaintenanceSettings` (id 1), logs `MAINTENANCE_MODE_TOGGLED`. `500 "Failed to update maintenance settings"` on failure.

### `GET /api/admin/perf-stats/stream`

**Why:** Live SSE feed backing `/admin/diagnostics`/perf-monitor views: in-memory player perf snapshots, aggregate stats, per-server stats, and a rolling history window, pushed every 3 seconds.

**Auth:** `VIEW_PERF_MONITOR` (`canViewPerfMonitor()`). Auth is checked manually here (not via `requirePermission()` helper) — `getSession()` then a direct `prisma.user.findUnique` + `createPermissionManager`.

**Request:** No query params.

```bash
curl -sS -N -b cookies.txt "$HSR_URL/api/admin/perf-stats/stream"
```

**Response:** SSE stream built with `createSSEResponse()` (route name `admin/perf-stats`, 20s heartbeat). Each event is a JSON payload sent immediately and then every 3000ms:
```json
{
  "players": [{ "robloxId": "123", "playerName": "...", "fps": 58, "ping": 42, "speed": 12.3, "team": "Train driver" }],
  "aggregate": { "avgFps": 57.2, "avgPing": 51, "playerCount": 34 },
  "server": { "serverId": "job-abc", "serverFps": 59, "heartbeatMs": 16.8, "serverMemoryMB": 1024 },
  "servers": [{ "serverId": "job-abc", "..." : "..." }],
  "history": [{ "timestamp": "2026-09-13T10:00:00.000Z", "avgFps": 57.1, "...": "..." }]
}
```
Errors: `401 Unauthorized` (no session / user not found), `403 Forbidden` if not `canViewPerfMonitor()`.

### `GET /api/admin/perf-stats/history`

**Why:** Historical perf-stat query behind the diagnostics/track-mapper charts — either a single player's timeline or the server-wide aggregate snapshots, optionally filtered to one Roblox server instance.

**Auth:** `VIEW_PERF_MONITOR` (`canViewPerfMonitor()`), checked the same manual way as the stream route.

**Request:** Query params: `from` (ISO date string, optional, default now-24h), `to` (ISO date string, optional, default now), `robloxId` (string, optional — switches to per-player mode), `serverId` (string, optional — filters aggregate mode to one `game.JobId`). The window is clamped to a maximum of 24 hours (if `to - from > 24h`, `from` is pulled forward to `to - 24h`).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/perf-stats/history?from=2026-09-12T00:00:00Z&to=2026-09-13T00:00:00Z&serverId=job-abc"
```

**Response (aggregate mode, no `robloxId`):**
```json
{
  "data": [
    {
      "timestamp": "2026-09-12T00:00:10.000Z",
      "serverId": "job-abc",
      "avgFps": 58.4,
      "avgPing": 47,
      "avgSpeed": 22.1,
      "playerCount": 12,
      "serverFps": 60,
      "heartbeatMs": 16.7,
      "serverMemoryMB": 980,
      "highPingCount": 1,
      "isLagSpike": false
    }
  ]
}
```
**Response (per-player mode, `robloxId` given):**
```json
{
  "data": [
    { "timestamp": "2026-09-12T00:00:10.000Z", "fps": 60, "ping": 38, "speed": 15.2, "team": "Train driver", "playerName": "SomeName" }
  ]
}
```
Both are capped at 8640 rows (max 1 day at 10s intervals). Errors: `401 Unauthorized`, `403 Forbidden`.

## Admin: Schedule Requests, Shop, Stations, Stops

### `GET /api/admin/schedule-request`

**Why:** Lists timetable schedule requests (driver/dispatcher bookings) for the admin approval board, filterable by date/month/status/type, with each DRIVER request's paired return train attached so the admin sees both legs of the service.

**Auth:** `requireAuth()` then a manual check: `MANAGE_SCHEDULE_MVK` (via `canManageScheduleRequestsMVK()`) or `MANAGE_SCHEDULE_FSZT` (via `canManageScheduleRequestsFSZT()`), or `canViewAdminPanel()`. A caller with only MVK is silently restricted to `requestType: "DRIVER"`, and FSZT-only to `"DISPATCHER"`, unless `?type=` is explicit or the caller can view the whole admin panel.

**Request:** Query params, all optional: `status` (`PENDING`/`APPROVED`/`REJECTED`), `type` (`DRIVER`/`DISPATCHER`), `date` (`YYYY-MM-DD`, exact-day filter), `year`/`month` (defaults to current month when `date` is absent).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/schedule-request?status=PENDING&date=2026-09-14"
```

**Response:**
```json
[
  {
    "id": "clx...",
    "requestType": "DRIVER",
    "status": "PENDING",
    "date": "2026-09-14T00:00:00.000Z",
    "trainId": "clt...",
    "trackNumber": null,
    "user": { "id": "u1", "displayName": "Kovács Béla", "name": "kovacsb", "discordRole": "Driver", "image": null },
    "train": { "id": "clt...", "trainNumber": "9014", "trainType": "IC", "direction": 0, "fordaGroup": null, "pairGroup": "IC90" },
    "station": null,
    "returnTrain": { "id": "clt2...", "trainNumber": "9015", "trainType": "IC", "direction": 1, "pairGroup": "IC90" }
  }
]
```
Error: `403 Forbidden`; `500` on query failure.

### `PUT /api/admin/schedule-request`

**Why:** Approve or reject a driver/dispatcher booking. Approving a DRIVER request against a train re-runs the same conflict checks the booking form already applied (a train claimed by another approved driver, or a departure-time clash within the driver's other approved bookings), auto-rejects any other still-PENDING requests for that same train/date, and notifies everyone affected.

**Auth:** `requireAuth()`; the check is per the *request's own* type — `MANAGE_SCHEDULE_MVK` for a `DRIVER` request, `MANAGE_SCHEDULE_FSZT` for `DISPATCHER` — or `canViewAdminPanel()`.

**Request:** JSON body: `requestId` (string, required), `status` (`"APPROVED"` | `"REJECTED"`, required), `responseNote` (string, optional), `trackNumber` (string, **required when approving a DRIVER request tied to a train**).

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"requestId":"clx...","status":"APPROVED","trackNumber":"3"}' \
  "$HSR_URL/api/admin/schedule-request"
```

**Response:** the updated `ScheduleRequest` with `user`/`train`/`station` included.
```json
{ "id": "clx...", "status": "APPROVED", "trackNumber": "3", "approvedById": "admin1", "approvedAt": "2026-09-13T10:00:00.000Z", "responseNote": null, "user": {"id":"u1","displayName":"Kovács Béla","name":"kovacsb","discordRole":"Driver"}, "train": {"id":"clt...","trainNumber":"9014","trainType":"IC","direction":0} }
```
Errors: `400` missing `requestId`/`status`, invalid `status` value, missing `trackNumber` on a DRIVER approval, or a time/train conflict (`{"error": "Time conflict: this driver already has train 9012 departing at 08:15 on this date", "conflicts": [...]}` / `"This train is already assigned to another driver for this date"`); `403` wrong-branch permission; `404` request not found.

### `DELETE /api/admin/schedule-request?id=<requestId>`

**Why:** Remove a schedule request entirely (not a status change) — used to clear out a mistaken or stale booking.

**Auth:** `requireAuth()`; permission is checked against the request's own `requestType` the same way as `PUT` (`MANAGE_SCHEDULE_MVK`/`MANAGE_SCHEDULE_FSZT`), or `canViewAdminPanel()`.

**Request:** Query param `id` (string, required).

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/schedule-request?id=clx..."
```

**Response:** `{ "success": true }`. Errors: `400` missing `id`; `403` forbidden; `404` not found (either lookup or the delete helper itself).

### `GET /api/admin/schedule-request/missed-assignment-penalty-setting`

**Why:** Reads the tunables for the proportional throttle applied to a driver's auto-assign weight after a missed assignment (recency-weighted, decaying penalty rather than a flat ban).

**Auth:** `requireAuth()` + `canManageScheduleRequests()` (either `MANAGE_SCHEDULE_MVK` or `MANAGE_SCHEDULE_FSZT`) or `canViewAdminPanel()`.

**Request:** none.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/schedule-request/missed-assignment-penalty-setting"
```

**Response:**
```json
{ "enabled": true, "windowDays": 14, "halfLifeDays": 7, "fullWeight": 1.0, "pendingAppealWeight": 0.0, "approvedAppealWeight": 0.0, "maxReductionRatio": 0.75, "applyToDispatcher": false }
```
Falls back to this exact shape (defaults) when no `MissedAssignmentPenaltySettings` row exists yet. `403` if unauthorized; `500` on failure.

### `PUT /api/admin/schedule-request/missed-assignment-penalty-setting`

**Why:** Update the penalty tunables (window length, decay half-life, appeal weights, dispatcher opt-in).

**Auth:** `requireAuth()` + `canManageScheduleRequests()` or `canViewAdminPanel()`.

**Request:** JSON body, all optional (unsupplied numeric fields fall back to defaults, not to the stored row): `enabled` (bool, default `true` if omitted — an old client omitting the field never silently disables the penalty), `windowDays` (number, clamped 1–60), `halfLifeDays` (number, clamped 0.5–60), `fullWeight`/`pendingAppealWeight`/`approvedAppealWeight`/`maxReductionRatio` (number, clamped 0–1), `applyToDispatcher` (bool).

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"windowDays":21,"halfLifeDays":10,"maxReductionRatio":0.5}' \
  "$HSR_URL/api/admin/schedule-request/missed-assignment-penalty-setting"
```

**Response:** the upserted settings row, same shape as GET. `403` forbidden; `500` on failure.

### `GET /api/admin/schedule-request/auto-assign-setting`

**Why:** Reads the same-day auto-assign master switch (whether the system may automatically assign drivers/dispatchers to today's trains).

**Auth:** `requireAuth()` + `canManageScheduleRequests()` or `canViewAdminPanel()`.

**Request:** none.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/schedule-request/auto-assign-setting"
```

**Response:** `{ "enabled": false }` (reads `ScheduleSettings.autoAssignEnabled`, defaulting to `false` if the singleton row doesn't exist). `403`/`500` on failure.

### `PUT /api/admin/schedule-request/auto-assign-setting`

**Why:** Toggle the same-day auto-assign switch on/off.

**Auth:** `requireAuth()` + `canManageScheduleRequests()` or `canViewAdminPanel()`.

**Request:** JSON body: `enabled` (boolean).

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"enabled":true}' "$HSR_URL/api/admin/schedule-request/auto-assign-setting"
```

**Response:** `{ "enabled": true }`. Writes `SCHEDULE_AUTO_ASSIGN_TOGGLED` to the activity log. `403`/`500` on failure.

### `POST /api/admin/schedule-request/auto-assign`

**Why:** Runs the auto-assignment algorithm for a given date — assigns drivers and/or dispatchers to unbooked trains, optionally weighting against recent missed-assignment history. Supports a dry run so an admin can preview before committing.

**Auth:** `requireAuth()`; scope-gated — `DRIVER` scope needs `canManageScheduleRequestsMVK()` (or `canViewAdminPanel()`), `DISPATCHER` scope needs `canManageScheduleRequestsFSZT()` (or `canViewAdminPanel()`); `BOTH` needs both.

**Request:** JSON body: `date` (string, required), `considerMissedAssignments` (bool, default `false`), `dryRun` (bool, default `true`), `scope` (`"DRIVER"` | `"DISPATCHER"` | `"BOTH"`, default `"BOTH"`, case-insensitive), `seed` (number, optional — for reproducible randomized assignment).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"date":"2026-09-14","scope":"DRIVER","dryRun":true,"considerMissedAssignments":true}' \
  "$HSR_URL/api/admin/schedule-request/auto-assign"
```

**Response:** whatever `runScheduleAutoAssign()` returns (assignment plan/results per train, keyed by scope) — passed straight through as JSON. Errors: `400` missing `date` or invalid `scope`; `403` insufficient scope permission; `500` on failure.

### `POST /api/admin/schedule-request/update-email`

**Why:** Manually trigger the "your schedule changed" notification emails for a given date — used when an admin edits assignments outside the normal approve/reject flow and still wants affected drivers/dispatchers notified.

**Auth:** `requireAuth()` + `canManageScheduleRequests()` or `canViewAdminPanel()`.

**Request:** JSON body: `date` (string, required).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"date":"2026-09-14"}' "$HSR_URL/api/admin/schedule-request/update-email"
```

**Response:** whatever `sendScheduleUpdateEmails(date, actorId)` returns (e.g. counts of emails queued) passed through as JSON. `400` missing `date`; `403` forbidden; `500` on failure.

### `GET /api/admin/stations`

**Why:** Lists every timetable station in line order, for the Timetable Editor's station list and the stops grid's column headers.

**Auth:** None — no `requirePermission`/`requireAuth` call in this handler; it is open to any caller who can reach the route (the page itself is gated client-side by `MANAGE_TRAINS`, but the API route is unauthenticated).

**Request:** none.

```bash
curl -sS "$HSR_URL/api/admin/stations"
```

**Response:**
```json
[ { "id": "cls1", "name": "Budapest-Nyugati", "km": 0, "order": 0 }, { "id": "cls2", "name": "Vác", "km": 34, "order": 1 } ]
```
`500` on query failure.

### `POST /api/admin/stations`

**Why:** Add a new station to the line.

**Auth:** `MANAGE_TRAINS`.

**Request:** JSON body: `name` (string, required), `km` (number, required — distance marker), `order` (number, required — must be unique; the editor normally derives it via `PATCH` reorder rather than this raw field).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"name":"Vác","km":34,"order":1}' "$HSR_URL/api/admin/stations"
```

**Response:** `201` with the created station. `400` if `name`/`km`/`order` missing; `500` on failure (including a unique-`order` collision surfaced as a generic 500).

### `PUT /api/admin/stations`

**Why:** Edit a station's name/km/order directly (rarely used for `order` — the drag list uses `PATCH` instead, since a raw `order` write can collide with `Station.order`'s uniqueness).

**Auth:** `MANAGE_TRAINS`.

**Request:** JSON body: `id` (string, required), `name`, `km`, `order` (all optional — whatever is supplied overwrites).

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"id":"cls2","name":"Vác állomás"}' "$HSR_URL/api/admin/stations"
```

**Response:** the updated station row. `400` missing `id`; `500` on failure.

### `DELETE /api/admin/stations?id=<stationId>`

**Why:** Remove a station from the line.

**Auth:** `MANAGE_TRAINS`.

**Request:** Query param `id` (required).

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/stations?id=cls2"
```

**Response:** `{ "success": true }`. `400` missing `id`; `500` on failure (e.g. FK constraint from existing `ScheduleStop` rows referencing it — surfaces as a generic 500, not a specific error message).

### `PATCH /api/admin/stations`

**Why:** Reorder the whole line in one call from the drag-and-drop station list. Because `Station.order` is a unique column, a naive per-row update loop would collide with whichever row still holds the target position; this writes every station to a negative placeholder position first, then to its final index, all inside one transaction.

**Auth:** `MANAGE_TRAINS`.

**Request:** JSON body: `order` (array of station id strings, required — must list *every* existing station exactly once, no duplicates, no unknown ids).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"order":["cls2","cls1","cls3"]}' "$HSR_URL/api/admin/stations"
```

**Response:** the full station list, re-ordered.
```json
[ { "id": "cls2", "name": "Vác", "km": 34, "order": 0 }, { "id": "cls1", "name": "Budapest-Nyugati", "km": 0, "order": 1 } ]
```
Errors: `400` if `order` is not an array of strings, or doesn't list every station exactly once (missing, duplicate, or unknown id); `500` on transaction failure.

### `POST /api/admin/stops`

**Why:** Create or update a single train's stop at a single station (arrival/departure time, pass-through flag, note). Superseded in the main editor UI by the whole-column `PUT`, but still used for one-off edits.

**Auth:** `MANAGE_TRAINS`.

**Request:** JSON body: `trainId` (string, required), `stationId` (string, required), `arrivalTime`/`departureTime` (string, optional, raw — not normalized here), `passes` (bool, optional, default `false`), `note` (string, optional).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"trainId":"clt1","stationId":"cls2","arrivalTime":"08:34","departureTime":"08:36"}' \
  "$HSR_URL/api/admin/stops"
```

**Response:** `201` with the upserted `ScheduleStop` row.
```json
{ "trainId": "clt1", "stationId": "cls2", "arrivalTime": "08:34", "departureTime": "08:36", "passes": false, "note": null }
```
`400` missing `trainId`/`stationId`; `500` on failure.

### `DELETE /api/admin/stops?trainId=<id>&stationId=<id>`

**Why:** Remove a single stop row (train does not call at this station).

**Auth:** `MANAGE_TRAINS`.

**Request:** Query params `trainId`, `stationId` (both required).

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/stops?trainId=clt1&stationId=cls2"
```

**Response:** `{ "success": true }`. `400` missing either param; `500` on failure (e.g. row not found — surfaces as generic 500 since Prisma's `delete` throws on a missing record).

### `PUT /api/admin/stops`

**Why:** Save a whole train's timetable column in one transaction — the Timetable Editor's Save button. An emptied row (no times, not marked passing, no note) is deleted rather than written as nulls, so "does this train call here" stays answerable from row existence rather than needing a null-check. Times are normalized server-side (`normalizeTime`) as a second line of defense beyond the grid's own formatting.

**Auth:** `MANAGE_TRAINS`.

**Request:** JSON body: `trainId` (string, required), `stops` (array, required) — each entry: `stationId` (string, required, duplicates within the array are silently deduped keeping the first), `arrivalTime`/`departureTime` (string, optional — ignored/nulled when `passes` is true), `passes` (bool, optional), `note` (string, optional, trimmed; empty string becomes `null`).

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"trainId":"clt1","stops":[{"stationId":"cls1","departureTime":"800"},{"stationId":"cls2","arrivalTime":"834","departureTime":"836"},{"stationId":"cls3","passes":true}]}' \
  "$HSR_URL/api/admin/stops"
```

**Response:** the full saved list of stops for the train, including station data.
```json
[ { "trainId": "clt1", "stationId": "cls1", "arrivalTime": null, "departureTime": "08:00", "passes": false, "note": null, "station": { "id": "cls1", "name": "Budapest-Nyugati" } } ]
```
Errors: `400` if `trainId` is not a string or `stops` is not an array; `404` train not found; `500` on transaction failure.

### `GET /api/admin/shop/settings`

**Why:** Read the shop's global settings (whether the shop is enabled at all, and the fallback shipping fee used when Printful can't quote one) — upserts the singleton row (`id: 1`) on first read so the page never has to handle "no settings exist yet".

**Auth:** `MANAGE_SHOP`.

**Request:** none.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/shop/settings"
```

**Response:** `{ "id": 1, "shopEnabled": true, "fallbackShippingFeeHuf": 1500 }`. `500` on failure.

### `PATCH /api/admin/shop/settings`

**Why:** Toggle the shop on/off or update the fallback shipping fee.

**Auth:** `MANAGE_SHOP`.

**Request:** JSON body, both optional: `shopEnabled` (bool), `fallbackShippingFeeHuf` (number).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"shopEnabled":false}' "$HSR_URL/api/admin/shop/settings"
```

**Response:** the upserted settings row. `500` on failure.

### `GET /api/admin/shop/products`

**Why:** List shop products for the admin catalogue view, with images, filterable by active state, physical/digital, and a name search.

**Auth:** `MANAGE_SHOP`.

**Request:** Query params, all optional: `active` (`"true"`/`"false"`), `isPhysical` (`"true"`/`"false"`), `search` (matches `nameHu`/`nameEn` via `contains`).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/shop/products?active=true&search=badge"
```

**Response:**
```json
[ { "id": "shp1", "nameHu": "HSR Kitűző", "nameEn": "HSR Badge", "priceHuf": 2500, "stockMode": "LIMITED", "stock": 40, "isPhysical": true, "active": true, "images": [ { "id": "img1", "url": "https://s3.hsrgames.hu/shop-images/shp1/a.jpg", "sortOrder": 0, "source": "upload" } ] } ]
```
`500` on failure.

### `POST /api/admin/shop/products`

**Why:** Create a new shop product, physical (Printful/manual fulfillment) or digital (e.g. `APPLE_PASS`).

**Auth:** `MANAGE_SHOP`.

**Request:** JSON body: `nameHu`, `nameEn` (string, required), `priceHuf` (number, required), `stockMode` (string, required, e.g. `"UNLIMITED"`/`"LIMITED"`), `descHu`/`descEn` (string, optional, default `""`), `stock` (number, optional, default `0`), `isPhysical` (bool, optional, default `false`), `printfulSyncProductId`/`printfulVariantId` (string|number, optional, normalized to a trimmed string or `null`), `fulfillmentType` (string, **required unless `isPhysical` is true**, default `"MANUAL"`), `fulfillmentData` (JSON, optional), `active` (bool, optional, default `true`).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"nameHu":"HSR Kitűző","nameEn":"HSR Badge","priceHuf":2500,"stockMode":"LIMITED","stock":40,"isPhysical":true}' \
  "$HSR_URL/api/admin/shop/products"
```

**Response:** `201` with the created product. `400` if any required field is missing (including `fulfillmentType` for a non-physical product with none supplied); `500` on failure.

### `GET /api/admin/shop/products/[id]`

**Why:** Fetch one product's full detail (images, and a count of how many order items reference it — used to decide whether deleting is a hard delete or a soft deactivate).

**Auth:** `MANAGE_SHOP`.

**Request:** path param `id`.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/shop/products/shp1"
```

**Response:** the product with `images` and `_count.orderItems`.
```json
{ "id": "shp1", "nameEn": "HSR Badge", "priceHuf": 2500, "images": [], "_count": { "orderItems": 3 } }
```
`404` if not found; `500` on failure.

### `PATCH /api/admin/shop/products/[id]`

**Why:** Edit a product's fields. Uses an explicit allowlist so a client can never write to fields outside the update surface (e.g. `id`, `createdAt`).

**Auth:** `MANAGE_SHOP`.

**Request:** JSON body — only these keys are read, anything else is ignored: `nameHu`, `nameEn`, `descHu`, `descEn`, `priceHuf`, `stockMode`, `stock`, `isPhysical`, `printfulSyncProductId`, `printfulVariantId`, `printfulSyncVariantId` (the last three normalized to trimmed string or `null`), `fulfillmentType`, `fulfillmentData`, `active`. All optional — only supplied keys are written.

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"priceHuf":2900,"active":false}' "$HSR_URL/api/admin/shop/products/shp1"
```

**Response:** the updated product row. `500` on failure (no explicit 404 check before the update — a missing id surfaces as a generic 500 from Prisma).

### `DELETE /api/admin/shop/products/[id]`

**Why:** Remove a product. A product with existing order history is soft-deleted (deactivated) instead of removed, since deleting it would corrupt the order history's product reference; a product with no orders is hard-deleted along with its uploaded R2 images (Printful-sourced image URLs are left alone, since nothing was uploaded to R2 for them).

**Auth:** `MANAGE_SHOP`.

**Request:** path param `id`.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/shop/products/shp1"
```

**Response:** `{ "ok": true, "softDeleted": true }` (has order history) or `{ "ok": true, "softDeleted": false }` (hard-deleted). `404` if not found; `500` on failure (R2 delete failures for individual images are caught and logged, not fatal to the request).

### `POST /api/admin/shop/products/[id]/images`

**Why:** Upload a product image to Cloudflare R2 and register it as a `ShopProductImage` row, appended after the current max `sortOrder`.

**Auth:** `MANAGE_SHOP`.

**Request:** multipart/form-data: `file` (required — image file, one of `image/jpeg`/`image/png`/`image/webp`/`image/gif`, max 10MB).

```bash
curl -sS -X POST -b cookies.txt -F "file=@badge.png" "$HSR_URL/api/admin/shop/products/shp1/images"
```

**Response:** `201` `{ "id": "img2", "url": "https://s3.hsrgames.hu/shop-images/shp1/<uuid>.png" }`. Errors: `404` product not found; `503` R2 not configured; `400` no file / invalid type / too large; `500` on upload failure.

### `DELETE /api/admin/shop/products/[id]/images/[imgId]`

**Why:** Remove a single product image — deletes the R2 object first (if it's an uploaded one, not a Printful-hotlinked one) then the DB row.

**Auth:** `MANAGE_SHOP`.

**Request:** path params `id` (product), `imgId` (image).

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/shop/products/shp1/images/img2"
```

**Response:** `{ "ok": true }`. `404` image not found for that product; `500` on failure (R2 deletion failure is caught/logged, not fatal).

### `POST /api/admin/shop/products/[id]/images/import-printful`

**Why:** Pull the preview images Printful generated for the product's linked sync variant straight into the product's image list, replacing whatever Printful-sourced images were there before (uploaded images are untouched).

**Auth:** `MANAGE_SHOP`.

**Request:** path param `id` (product) — must already have `printfulSyncProductId` set; no body.

```bash
curl -sS -X POST -b cookies.txt "$HSR_URL/api/admin/shop/products/shp1/images/import-printful"
```

**Response:** `{ "imported": 3 }` — count of images created. Errors: `404` product not found; `400` no `printfulSyncProductId` linked, an unparseable id, or no matching variant found on Printful's side; `500` on failure. **External API call**: `getPrintfulSyncProduct()` hits the Printful API — a Printful outage or an invalid/revoked API token surfaces here as a `500 Internal server error` (the route does not distinguish Printful failures from any other exception).

### `GET /api/admin/shop/printful/sync-products`

**Why:** Browse the Printful account's sync products (paginated) so an admin can pick one to link to a local `ShopProduct`.

**Auth:** `MANAGE_SHOP`.

**Request:** Query params, optional: `offset` (number, default `0`), `limit` (number, default `20`).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/shop/printful/sync-products?offset=0&limit=20"
```

**Response:** whatever `listPrintfulSyncProducts()` returns — the Printful sync-products list payload (ids, names, thumbnails) — passed through as JSON. `500` on failure. **External API call**: this proxies directly to Printful; a Printful outage, rate limit, or bad API key returns `500 Internal server error` here with no distinguishing detail.

### `GET /api/admin/shop/printful/sync-products/[id]`

**Why:** Fetch one Printful sync product's full detail (its variants and their preview files) — used both by the linking UI and internally by the image-import route.

**Auth:** `MANAGE_SHOP`.

**Request:** path param `id` (Printful numeric sync product id, as a string, parsed with `parseInt`).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/shop/printful/sync-products/123456"
```

**Response:** whatever `getPrintfulSyncProduct()` returns (sync product + `sync_variants[]`, each with `files[]` carrying `visible`/`preview_url`). `500` on failure. **External API call**: same as above — a Printful-side failure (down, rate-limited, invalid id) surfaces as a generic `500`.

### `GET /api/admin/shop/orders`

**Why:** List recent shop orders (capped at 100, newest first) for the admin fulfillment queue, with buyer and line-item/product info attached.

**Auth:** `MANAGE_SHOP`.

**Request:** Query params, optional: `status` (a `ShopOrderStatus` enum value, e.g. `PENDING`/`PAID`/`SHIPPED`/`DELIVERED`/`CANCELLED`), `search` (matches order `id`, `trackingNumber`, or the buyer's `name`, all via `contains`).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/shop/orders?status=PAID"
```

**Response:**
```json
[ { "id": "ord1", "status": "PAID", "createdAt": "2026-09-10T12:00:00.000Z", "user": { "id": "u1", "name": "kovacsb", "displayName": "Kovács Béla", "image": null }, "items": [ { "id": "it1", "quantity": 1, "product": { "id": "shp1", "nameHu": "HSR Kitűző", "nameEn": "HSR Badge" } } ] } ]
```
`500` on failure.

### `GET /api/admin/shop/orders/[id]`

**Why:** Fetch one order's full detail, including the buyer's Discord contact info and each item's product images, for the order detail panel.

**Auth:** `MANAGE_SHOP`.

**Request:** path param `id`.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/shop/orders/ord1"
```

**Response:** the order with `user` (incl. `discordId`, `discordEmail`) and `items[].product.images` (first image only, by `sortOrder`). `404` not found; `500` on failure.

### `PATCH /api/admin/shop/orders/[id]`

**Why:** Update an order's fulfillment state — status, tracking info, notes, or the linked Printful order id. A status transition to `SHIPPED`/`DELIVERED`/`CANCELLED` stamps the matching timestamp the first time it's reached, and `SHIPPED`/`CANCELLED` transitions email the buyer (if they have a `discordEmail`) with `shopOrderShipped`/`shopOrderCancelled`.

**Auth:** `MANAGE_SHOP`.

**Request:** JSON body, all optional: `status` (`ShopOrderStatus`), `trackingNumber`, `trackingCarrier`, `trackingUrl`, `notes` (strings), `printfulOrderId` (string).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"status":"SHIPPED","trackingNumber":"HU123456789","trackingCarrier":"Posta"}' \
  "$HSR_URL/api/admin/shop/orders/ord1"
```

**Response:** the updated order, same include shape as GET. `404` not found; `500` on failure (email send failures are caught/logged, not fatal to the request — the order still updates).

### `DELETE /api/admin/shop/orders/[id]`

**Why:** Permanently delete an order and its line items (e.g. a test/duplicate order) — unlike products, there's no soft-delete path here.

**Auth:** `MANAGE_SHOP`.

**Request:** path param `id`.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/shop/orders/ord1"
```

**Response:** `{ "success": true }`. `404` not found; `500` on failure.

### `PATCH /api/admin/shop/orders/[id]/items/[itemId]`

**Why:** Mark a single line item within an order as fulfilled (or add a fulfillment note) — used for multi-item orders where some items ship separately or need manual fulfillment tracking distinct from the order's own status.

**Auth:** `MANAGE_SHOP`.

**Request:** path params `id` (order), `itemId`; JSON body, both optional: `fulfilledAt` (ISO date string or `null` to clear), `fulfillmentNote` (string).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"fulfilledAt":"2026-09-13T10:00:00.000Z","fulfillmentNote":"Handed over in person"}' \
  "$HSR_URL/api/admin/shop/orders/ord1/items/it1"
```

**Response:** the updated `ShopOrderItem`. `404` if the item doesn't belong to that order; `500` on failure.

### `POST /api/admin/shop/orders/[id]/resubmit-printful`

**Why:** Retry sending an order to Printful for fulfillment after a prior submission failed (network error, Printful-side rejection, etc.) — the manual counterpart to the `shop-printful-retry` cron.

**Auth:** `MANAGE_SHOP`.

**Request:** path param `id` (order); no body.

```bash
curl -sS -X POST -b cookies.txt "$HSR_URL/api/admin/shop/orders/ord1/resubmit-printful"
```

**Response:** `{ "ok": true }`. **External API call**: `submitOrderToPrintful()` calls the Printful order-creation API — any failure (down, rejected line item, invalid address, rate limit) is caught and returned as `{ "error": "<thrown message>" }` with status `500`, so unlike most routes here the error text can carry Printful's own rejection reason.

## Admin: Statistics, Supervision Rules, Terms, Discord Tickets, Track Map

### `GET /api/admin/statistics`

**Why:** The overview tab of `/admin/statistics` — headline numbers for both populations the project has (the Discord guild mirror and the site's own `User` table) plus the funnel between them, operations counters and trend history.

**Auth:** `VIEW_STATISTICS` (`requirePermission((p) => p.canViewStatistics())`).

**Request:** Query params, all optional:
- `days` — one of `7`, `30`, `90`, `365`; anything else falls back to `30`.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/statistics?days=30"
```

**Response:** 200 with a large object. Notable fields: `lastSyncedAt`, `syncStale` (mirror older than 6h), `neverSynced` (no sync row exists at all), plus `discord`, `activity`, `web`, `funnel`, `operations` and `trends` sections built from `DiscordGuildMember`/`CommunityDailyStat`.

```json
{
  "generatedAt": "2026-09-13T10:00:00.000Z",
  "rangeDays": 30,
  "lastSyncedAt": "2026-09-13T09:58:00.000Z",
  "syncStale": false,
  "neverSynced": false,
  "discord": {
    "totalMembers": 1240,
    "humanMembers": 1190,
    "botMembers": 50,
    "boosters": 12,
    "pendingMembers": 3,
    "timedOutMembers": 1,
    "staffMembers": 58,
    "rankedMembers": 340,
    "unrankedMembers": 850,
    "joins": 22,
    "leaves": 9,
    "netGrowth": 13,
    "roleDistribution": [{ "role": "Passenger", "category": "member", "priority": 0, "count": 850 }],
    "categoryDistribution": [{ "category": "member", "count": 850 }],
    "ageBuckets": [{ "label": "< 30 days", "count": 40 }],
    "tenureBuckets": [{ "label": "< 7 days", "count": 12 }]
  },
  "activity": {
    "windowDays": 30,
    "active7d": 400,
    "active30d": 620,
    "inactive": 570,
    "activeRate": 0.5210,
    "neverActive": 210,
    "neverPosted": 180,
    "messages": 15230,
    "reactions": 3400,
    "voiceMinutes": 22000
  },
  "web": {
    "totalUsers": 900,
    "newUsers": 30,
    "confirmedUsers": 870,
    "unconfirmedUsers": 30,
    "bannedUsers": 4,
    "activeInGame": 12,
    "dau": 45,
    "wau": 210,
    "mau": 480,
    "loginBreakdown": [{ "action": "LOGIN", "count": 500 }]
  },
  "funnel": {
    "guildMembers": 1190,
    "linkedMembers": 850,
    "unlinkedMembers": 340,
    "webUsers": 900,
    "webUsersWithDiscord": 850,
    "webUsersWithoutDiscord": 50,
    "departedLinkedUsers": 6,
    "coverageRate": 0.7142,
    "linkRate": 0.9444
  },
  "operations": {
    "duties": 300,
    "activeDuties": 4,
    "dutyHours": 512.3,
    "examAttempts": 20,
    "examsPassed": 15,
    "examPassRate": 0.75,
    "openTickets": 5,
    "closedTickets": 40,
    "automodEvents": 18,
    "warnings": 3,
    "totalPlayHours": 88210.4,
    "totalDistanceKm": 940213,
    "totalGameSessions": 51234
  },
  "trends": [{ "date": "2026-09-01", "totalMembers": 1230, "joins": 2, "leaves": 1 }]
}
```
Errors: 401/403 from `requirePermission`; 500 on unexpected error.

### `GET /api/admin/statistics/activity`

**Why:** Backs the Activity tab — is the server large but quiet? Breaks activity down by role, join-month cohort, a quietest-members ranking, and daily volume, with a configurable "active" threshold.

**Auth:** `VIEW_STATISTICS`.

**Request:** Query params, all optional:
- `threshold` — one of `7`, `14`, `30`, `90`; invalid/missing falls back to `ACTIVE_WINDOW_DAYS` (30).
- `days` — trend window for the daily volume chart, clamped to `[7, 365]`, default `30`.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/statistics/activity?threshold=30&days=30"
```

**Response:**
```json
{
  "threshold": 30,
  "trendDays": 30,
  "summary": {
    "totalMembers": 1190,
    "activeMembers": 620,
    "inactiveMembers": 570,
    "activeRate": 0.5210,
    "neverActive": 210,
    "neverPosted": 180,
    "voiceOnly": 25,
    "medianSilenceDays": 14
  },
  "byRole": [{ "role": "Member", "category": "member", "priority": 0, "members": 850, "active": 400, "neverPosted": 150, "activeRate": 0.4705 }],
  "byCohort": [{ "month": "2026-06", "joined": 40, "active": 22, "neverPosted": 10, "activeRate": 0.55 }],
  "leastActive": [
    {
      "discordId": "1234567890",
      "name": "somebody",
      "username": "somebody",
      "avatar": null,
      "highestRole": "Passenger",
      "category": "member",
      "joinedAt": "2025-01-01T00:00:00.000Z",
      "daysInServer": 620,
      "lastActivityAt": null,
      "daysSinceActivity": null,
      "messageCount": 0,
      "reactionCount": 0,
      "voiceMinutes": 0
    }
  ],
  "daily": [{ "date": "2026-09-01", "messages": 500, "reactions": 100, "voiceMinutes": 300, "activeMembers": 150 }],
  "backfill": { "status": "COMPLETED", "coversFrom": "2026-06-15T00:00:00.000Z" },
  "historyCoversFrom": "2026-06-15T00:00:00.000Z"
}
```
`leastActive` is capped at 50 rows. Errors: 401/403; 500.

### `GET /api/admin/statistics/backfill`

**Why:** Reports the state of the activity-history crawl (`DiscordActivityBackfill`), so the page can show a live progress bar instead of a blind "Start" button.

**Auth:** `VIEW_STATISTICS`.

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/statistics/backfill"
```

**Response:**
```json
{ "progress": { "status": "RUNNING", "completedChannels": 12, "cursorChannel": "998877", "cursorBefore": "1120334455", "coversFrom": null } }
```
`progress` may be `null` if a crawl has never been started.

### `POST /api/admin/statistics/backfill`

**Why:** Starts (or resumes) the message-history crawl that makes "never posted" mean something more than "not since the bot started counting".

**Auth:** `VIEW_STATISTICS`.

**Request:** JSON body, optional (missing/invalid body silently falls back to the default):
- `days` (number) — one of `30`, `90`, `180`, `365`; default `90`.
- `resume` (boolean) — resume from the saved cursor instead of restarting from zero; default `false`.

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"days":90,"resume":false}' \
  "$HSR_URL/api/admin/statistics/backfill"
```

**Response:** 200 `{ "started": true, "days": 90, "resumed": false }`. 409 `{ "error": "..." }` if a crawl is already running or otherwise refuses to start (`startBackfill()` reason).

### `GET /api/admin/statistics/members`

**Why:** The searchable, filterable, paginated per-member table behind the statistics page — the drill-down from the headline numbers to an actual list of accounts.

**Auth:** `VIEW_STATISTICS`.

**Request:** Query params, all optional:
- `q` — free-text search against username/globalName/nickname/discordId (`contains`).
- `status` — one of `all` (default) | `linked` | `unlinked` | `left` | `staff` | `bots` | `boosters` | `inactive` | `silent` | `active`.
- `role` — exact `highestRole` filter.
- `sort` — one of `joined` (default) | `name` | `role` | `created` | `activity` | `messages`.
- `page` — integer, default `1`, floored at `1`.
- `pageSize` — integer, default `25`, capped at `100`.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/statistics/members?status=inactive&sort=activity&page=1&pageSize=25"
```

**Response:**
```json
{
  "rows": [
    {
      "discordId": "1234567890",
      "username": "somebody",
      "globalName": "Somebody",
      "nickname": null,
      "avatar": null,
      "isBot": false,
      "highestRole": "Passenger",
      "roleCategory": "member",
      "rolePriority": 0,
      "roleCount": 1,
      "joinedAt": "2025-01-01T00:00:00.000Z",
      "accountCreatedAt": "2021-05-01T00:00:00.000Z",
      "premiumSince": null,
      "pending": false,
      "timeoutUntil": null,
      "leftAt": null,
      "lastActivityAt": null,
      "lastMessageAt": null,
      "messageCount": 0,
      "reactionCount": 0,
      "voiceMinutes": 0,
      "webAccount": {
        "id": "cly...",
        "name": "Somebody",
        "image": null,
        "robloxId": "998877",
        "isAdmin": false,
        "banned": false,
        "confirmed": true,
        "isActiveInGame": false,
        "lastLoginAt": "2026-09-01T00:00:00.000Z",
        "registeredAt": "2025-01-02T00:00:00.000Z",
        "playTimeSeconds": 120000,
        "level": 12,
        "duties": 30
      }
    }
  ],
  "page": 1,
  "pageSize": 25,
  "total": 570,
  "totalPages": 23,
  "roleOptions": [{ "role": "Passenger", "count": 850 }]
}
```
Errors: 401/403; 500.

### `POST /api/admin/statistics/sync`

**Why:** The "Sync now" button — pulls the live Discord guild member list into the `DiscordGuildMember` mirror on demand, for "I just changed roles, show me" rather than waiting for the cron.

**Auth:** `VIEW_STATISTICS`.

**Request:** No body.

```bash
curl -sS -X POST -b cookies.txt "$HSR_URL/api/admin/statistics/sync"
```

**Response:** 200 `{ "success": true, ... }` (spreads `runCommunityStatsSync()`'s result). **429** `{ "error": "Sync cooldown active", "retryInSeconds": 87 }` if fired within **2 minutes** of the last sync. **502** `{ "error": "Discord is unreachable — the mirror was left untouched" }` if Discord could not be reached. 500 on unexpected error.

### `GET /api/admin/supervision/fault-types`

**Why:** Lists the supervision rulebook (`SupervisionFaultType`) — the catalogue the evaluation sheet marks faults from and the settings page edits.

**Auth:** `MANAGE_SUPERVISION_RULES`.

**Request:** Query params, optional:
- `track` — `DRIVER` or `DISPATCHER`; anything else is ignored (no track filter).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/supervision/fault-types?track=DRIVER"
```

**Response:**
```json
{
  "faultTypes": [
    {
      "id": "clx...",
      "code": "RED_SIGNAL",
      "label": "Passed a red signal",
      "description": null,
      "kind": "INSTANT_FAIL",
      "points": 0,
      "scalable": false,
      "maxPoints": 0,
      "track": "DRIVER",
      "category": null,
      "active": true,
      "sortOrder": 10
    }
  ]
}
```

### `POST /api/admin/supervision/fault-types`

**Why:** Adds a new rulebook entry when the Head of Education decides on a new fault to grade against.

**Auth:** `MANAGE_SUPERVISION_RULES`.

**Request:** JSON body:
- `label` (string, required, trimmed non-empty).
- `code` (string, optional) — derived from `label` if omitted; normalized (NFD-stripped, uppercased, non-alnum → `_`, trimmed, max 40 chars); de-duplicated with a numeric suffix (`_2`, `_3`, …) if it collides.
- `kind` (`"INSTANT_FAIL"` | `"FAULT_POINT"`, default `"FAULT_POINT"`).
- `scalable` (boolean, only honored when `kind === "FAULT_POINT"`).
- `points` (number, clamped `0–20`; forced to `0` for `INSTANT_FAIL`).
- `maxPoints` (number; `Math.max(points, clampPoints(maxPoints, points))` when `scalable`, else equals `points`).
- `track` (`"DRIVER"` | `"DISPATCHER"` | otherwise `null`).
- `description` (string, optional).
- `category` (string, optional).
- `active` (boolean, default `true`).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"label":"Kürt használata elmulasztva","kind":"FAULT_POINT","points":2,"track":"DRIVER"}' \
  "$HSR_URL/api/admin/supervision/fault-types"
```

**Response:** 200 `{ "faultType": { "id": "...", "code": "KURT_HASZNALATA_ELMULASZTVA", "label": "...", ... } }`. 400 `{ "error": "Invalid JSON body" }` or `{ "error": "label is required" }`. 500 on failure.

### `PATCH /api/admin/supervision/fault-types/[id]`

**Why:** Edits an existing rulebook entry — re-pricing a fault, retiring it, changing its track — without a deploy.

**Auth:** `MANAGE_SUPERVISION_RULES`.

**Request:** JSON body, all fields optional (only present keys are applied):
- `label` (string, non-empty after trim to take effect).
- `description` (string or `null`, present via `"description" in body`).
- `category` (string or `null`, same pattern).
- `kind` (`"INSTANT_FAIL"` | `"FAULT_POINT"`).
- `track` (`"DRIVER"` | `"DISPATCHER"` | `null`, via `"track" in body`).
- `active` (boolean).
- `scalable` (boolean).
- `points` (number, clamped `0–20`, via `"points" in body`).
- `maxPoints` (number, clamped `0–20`, via `"maxPoints" in body`).
- `sortOrder` (number, via `"sortOrder" in body`).

Note: switching `kind` to `INSTANT_FAIL` forces `points`/`maxPoints` to `0` and `scalable` to `false`; otherwise `maxPoints` is recomputed as `Math.max(points, maxPoints)` when `scalable`, else equal to `points`.

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"points":3,"active":true}' \
  "$HSR_URL/api/admin/supervision/fault-types/clx123"
```

**Response:** 200 `{ "faultType": { ...updated } }`. 400 invalid JSON. 404 `{ "error": "Fault type not found" }`. 500 on failure.

### `DELETE /api/admin/supervision/fault-types/[id]`

**Why:** Retires a rulebook entry. Preserves history: an entry already used by a marked `SupervisionFault` is deactivated rather than deleted, so past sessions and reports still name it.

**Auth:** `MANAGE_SUPERVISION_RULES`.

**Request:** No body; `id` from the path.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/supervision/fault-types/clx123"
```

**Response:** 200 `{ "success": true, "retained": true }` (`retained: true` if it was deactivated rather than hard-deleted, i.e. `SupervisionFault` rows reference it). 404 if not found. 500 on failure.

### `GET /api/admin/supervision/members`

**Why:** A narrow member-search endpoint for picking who to open a probation/observation session against — deliberately separate from `/api/admin/users`, so it can be gated on the supervisory permission instead of `MANAGE_USERS`.

**Auth:** `MANAGE_SUPERVISION_REQUESTS` (`p.canManageSupervisionRequests()`).

**Request:** Query params:
- `q` (string) — search term; must be at least 2 characters or an empty list is returned immediately.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/supervision/members?q=kov"
```

**Response:**
```json
{
  "members": [
    { "id": "clx...", "displayName": "Kovács Péter", "name": "PeterK", "discordUsername": "peterk", "discordRole": "Trainee Driver" }
  ]
}
```
Capped at 10 results, banned users excluded.

### `GET /api/admin/supervision/reports`

**Why:** The Head of Education's cross-trainee report — which catalogue items fire most, and for whom — grouped the same way the trainee's own summary is.

**Auth:** `MANAGE_SUPERVISION_RULES`.

**Request:** Query params:
- `days` (number, optional, default `30`) — falls back to `30` if not finite.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/supervision/reports?days=30"
```

**Response:** Shape comes from `buildSupervisionReport()` (not read here); it is a JSON object with fault/frequency aggregates and per-member breakdowns. 500 `{ "error": "Failed to build report" }` on failure.

### `POST /api/admin/supervision/sessions`

**Why:** Opens a `PROBATION` (a candidate supervisor demonstrating on a HoE member) or `OBSERVATION` (report-only sit-in) session — the two kinds of `SupervisorRequest` that are opened by staff against a chosen member rather than requested by the trainee.

**Auth:** `MANAGE_INSTRUCTORS` for `kind: "PROBATION"`, `MANAGE_SUPERVISION_REQUESTS` for `kind: "OBSERVATION"` (checked via `requirePermission` with a conditional predicate on `kind`).

**Request:** JSON body:
- `kind` (`"PROBATION"` | `"OBSERVATION"`, required).
- `userId` (string, required) — the member being observed/tested.
- `track` (string, optional) — must be one of `getSupervisionTracks()`; defaults to `"DRIVER"` if invalid/absent.
- `note` (string, optional).

Also refused (400) if the relevant switch (`probationEnabled` / `observationEnabled` in `SupervisionRulesSettings`) is off, or if the target already has a `PENDING`/`IN_PROGRESS` session open.

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"kind":"OBSERVATION","userId":"clxUser1","track":"DRIVER","note":"Routine sit-in"}' \
  "$HSR_URL/api/admin/supervision/sessions"
```

**Response:** 200 `{ "request": { "id": "...", "userId": "clxUser1", "kind": "OBSERVATION", "track": "DRIVER", "status": "IN_PROGRESS", ... } }`. 400 for missing/invalid `kind`/`userId`, disabled switch, or an already-open session (`{"error": "This member already has a <kind> session open"}`). 404 `{ "error": "User not found" }`. 500 on failure.

### `GET /api/admin/supervision/settings`

**Why:** Reads the supervision rulebook's global thresholds (`SupervisionRulesSettings`, singleton id 1) plus their defaults/limits, for the `/admin/supervision-rules` settings form.

**Auth:** `MANAGE_SUPERVISION_RULES`.

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/supervision/settings"
```

**Response:**
```json
{
  "rules": {
    "enabled": true,
    "failPointThreshold": 4,
    "requiredLaps": 2,
    "pointsFromLap": 2,
    "instantFailFirstLapWarning": true,
    "requireLapDocumentation": true,
    "probationEnabled": true,
    "probationRequiredLaps": 2,
    "observationEnabled": true,
    "traineeInstructorRoleId": null,
    "instructorRoleIdOnPass": null,
    "retrainingRequiredFrom": null
  },
  "defaults": { "enabled": true, "failPointThreshold": 4, "requiredLaps": 2, "pointsFromLap": 2, "instantFailFirstLapWarning": true, "requireLapDocumentation": true, "probationEnabled": true, "probationRequiredLaps": 2, "observationEnabled": true, "traineeInstructorRoleId": null, "instructorRoleIdOnPass": null, "retrainingRequiredFrom": null },
  "limits": { "failPointThreshold": { "min": 1, "max": 50 }, "requiredLaps": { "min": 1, "max": 10 }, "pointsFromLap": { "min": 1, "max": 10 }, "probationRequiredLaps": { "min": 1, "max": 10 } }
}
```

### `PATCH /api/admin/supervision/settings`

**Why:** Edits the rulebook's thresholds — re-tuning the fail point bar, the required laps, or switching probation/observation off — without a deploy.

**Auth:** `MANAGE_SUPERVISION_RULES`.

**Request:** JSON body, validated by `validateRulesPatch()`; any subset of the fields listed in the GET response's `rules` object. Numeric fields (`failPointThreshold`, `requiredLaps`, `pointsFromLap`, `probationRequiredLaps`) are clamped to `RULE_LIMITS`. `traineeInstructorRoleId` / `instructorRoleIdOnPass` must be `null`/`""` (→ `null`) or a Discord snowflake matching `/^\d{5,25}$/`. `retrainingRequiredFrom` is `null`/`""` or a parseable date.

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"failPointThreshold":5,"observationEnabled":false}' \
  "$HSR_URL/api/admin/supervision/settings"
```

**Response:** 200 `{ "rules": { ...updated } }`. 400 `{ "error": "Invalid JSON body" }` or the field-specific validation error (e.g. `"traineeInstructorRoleId must be a Discord role id or empty"`). 500 on failure.

### `POST /api/admin/terms/notify`

**Why:** Sends a one-off "Terms of Service updated" email to every member with a known address, and records whether the terms text has actually changed since the last send so an accidental re-click doesn't spam everyone.

**Auth:** `MANAGE_USERS` (`p.hasFlag("MANAGE_USERS")`).

**Request:** JSON body:
- `force` (boolean, optional) — send even if the computed hash of `terms*` strings in `src/locales/en.json` matches the last recorded hash.

The route hashes (SHA-256) the sorted `key:value` pairs of every `en.json` key starting with `terms`, compares it against `data/terms-notified.json`'s `lastHash`, and refuses (unless `force`) if unchanged.

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"force":false}' \
  "$HSR_URL/api/admin/terms/notify"
```

**Response:** 200 `{ "message": "Sent terms update notification to 842 users" }`. 400 `{ "error": "No terms change detected" }` if the hash is unchanged and `force` was not set. Recipients are every `User` with a non-null `discordEmail`; each gets the `termsUpdated` template in their own `locale`. Writes `data/terms-notified.json` (`lastHash`, `lastNotifiedAt`, `lastNotifiedBy`) and an `ActivityLog` row (`TERMS_NOTIFY`).

### `GET /api/admin/tickets/discord`

**Why:** The Discord-ticket admin list — search, filter, and the summary stats card (open/closed/unclaimed/rating average) shown above the table.

**Auth:** `MANAGE_TICKETS` (via `requireTicketAccess()`, which is `canManageTickets()`).

**Request:** Query params, all optional:
- `status` — `"OPEN"` | `"CLOSED"`.
- `category` — exact `categoryKey`.
- `rated` — `"yes"` (has a rating) | `"no"` (no rating).
- `search` — matched against `discordUsername`, `discordUserId`, ticket `id` (exact), `categoryLabel`, `claimedByName` (all `contains` except `id`).
- `page` — default `1`, floored at `1`.
- `limit` — default `20`, capped at `50`.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/tickets/discord?status=OPEN&page=1&limit=20"
```

**Response:**
```json
{
  "tickets": [
    {
      "id": "clxTicket1",
      "status": "OPEN",
      "discordUsername": "somebody",
      "discordUserId": "1234567890",
      "categoryKey": "general",
      "categoryLabel": "Általános",
      "createdAt": "2026-09-12T10:00:00.000Z",
      "user": { "id": "clxUser1", "name": "somebody", "displayName": "Somebody", "image": null },
      "rating": null,
      "_count": { "messages": 4 }
    }
  ],
  "total": 5,
  "page": 1,
  "limit": 20,
  "stats": { "open": 5, "closed": 40, "unclaimed": 2, "rated": 30, "unrated": 10, "averageRating": 4.6 },
  "categories": [{ "key": "general", "label": "Általános", "count": 12 }]
}
```

### `GET /api/admin/tickets/discord/[id]`

**Why:** Loads one Discord ticket's full transcript for the admin panel's ticket detail view.

**Auth:** `MANAGE_TICKETS`.

**Request:** No body; `id` from the path.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/tickets/discord/clxTicket1"
```

**Response:** 200 `{ "ticket": { "id": "...", "messages": [...], "user": {...}, "rating": {...} | null } }`. 404 `{ "error": "Not found" }`.

### `PATCH /api/admin/tickets/discord/[id]`

**Why:** The three admin-panel actions on a Discord ticket: close it, reopen it, or claim it — mirroring what the Discord ticket buttons do, so closing from the panel behaves identically (same notification, same transcript, same channel teardown).

**Auth:** `MANAGE_TICKETS`.

**Request:** JSON body:
- `action` (`"close"` | `"reopen"` | `"claim"`, required).

`close`: generates a transcript, stamps `closedAt`/`closedBy`/`closedByUsername`, clears `channelId`, fires `notifyTicketClosed()`, posts a closing embed to the Discord channel and deletes it after 5s, writes `TICKET_CLOSED` to `ActivityLog`. Refuses (400) if already `CLOSED`.
`reopen`: sets status back to `OPEN`, clears close fields. Refuses (400) if already `OPEN`.
`claim`: sets `claimedByUserId`/`claimedByName` to the caller; if the ticket has a live channel, upserts `SupportConversation` to `ESCALATED` (silences the AI support bot).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"action":"claim"}' \
  "$HSR_URL/api/admin/tickets/discord/clxTicket1"
```

**Response:** 200 `{ "ticket": { ...updated, "messages": [...], "user": {...}, "rating": {...} | null } }`. 400 `{ "error": "Already closed" }` / `{ "error": "Already open" }` / `{ "error": "Unknown action" }`. 404 `{ "error": "Not found" }`.

### `DELETE /api/admin/tickets/discord/[id]`

**Why:** Permanently removes a Discord ticket record from the admin panel (distinct from closing — this deletes the row entirely).

**Auth:** `MANAGE_TICKETS`.

**Request:** No body; `id` from the path.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/tickets/discord/clxTicket1"
```

**Response:** 200 `{ "success": true }`. 404 `{ "error": "Not found" }`. Writes `TICKET_DELETED` to `ActivityLog`.

### `GET /api/admin/tickets/discord/count`

**Why:** The sidebar's open/unclaimed ticket badge. Deliberately its own lightweight endpoint — separate from the listing route, which also runs a paged query, a rating aggregate and a group-by — so a once-a-minute poll from every open admin tab stays cheap.

**Auth:** `MANAGE_TICKETS`.

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/tickets/discord/count"
```

**Response:** 200 `{ "open": 5, "unclaimed": 2 }`.

### `GET /api/admin/track-map`

**Why:** Loads the track mapper's settings, aggregate stats and the list of drivers who have contributed traces, for the `/admin/track-map` page's control panel.

**Auth:** `VIEW_PERF_MONITOR` (`requireTrackMapAccess()` → `permissions.canViewTrackMap()`, which is granted by the `VIEW_PERF_MONITOR` flag — the track mapper shares the perf-monitor population).

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/track-map"
```

**Response:**
```json
{
  "settings": {
    "id": 1,
    "enabled": true,
    "cellSize": 8,
    "minSpeed": 2,
    "minMoveStuds": 4,
    "retentionDays": 21,
    "includeAllTeams": false,
    "flipX": false,
    "flipZ": false,
    "rotationDeg": 0,
    "studsPerKm": 1000,
    "panX": 0,
    "panY": 0
  },
  "stats": { "cellCount": 84213, "traceCount": 512, "pointCount": 210034, "maxHits": 9820, "lastSampleAt": "2026-09-13T09:40:00.000Z" },
  "drivers": [{ "robloxId": "998877", "playerName": "somebody" }]
}
```

### `PATCH /api/admin/track-map`

**Why:** Edits the track mapper's calibration and behavior switches (`TrackMapSettings`, singleton id 1) — rotation, mirroring, studs-per-km, minimum-speed noise filter, etc. — stored rather than re-guessed each time the map is opened.

**Auth:** `VIEW_PERF_MONITOR`.

**Request:** JSON body, any subset of:
- Booleans: `enabled`, `includeAllTeams`, `flipX`, `flipZ`.
- Numbers, clamped to their limits: `cellSize` (0.5–64), `minSpeed` (0–200), `minMoveStuds` (0–200), `retentionDays` (0–365, rounded to integer), `rotationDeg` (-360–360), `studsPerKm` (0–1,000,000).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"rotationDeg":15,"includeAllTeams":false}' \
  "$HSR_URL/api/admin/track-map"
```

**Response:** 200 `{ "settings": { ...updated } }`. 400 `{ "error": "Invalid body" }` or `{ "error": "Invalid value for <key>" }` or `{ "error": "Nothing to update" }` (empty patch).

### `GET /api/admin/track-map/data`

**Why:** The actual drawing data for the canvas: averaged track cells, driver traces (decimated point lists) and station pins — capped hard so one impatient click cannot pull the whole table.

**Auth:** `VIEW_PERF_MONITOR`.

**Request:** Query params, all optional:
- `minHits` (number, default `1`, floored at `1`) — only cells with at least this many hits.
- `traceLimit` (number, default `200`, capped at `600` / `MAX_TRACES`).
- `pointsPerTrace` (number, default `2000`, clamped `50–4000`) — decimation target per trace.
- `robloxId` (string) — filter traces to one driver.
- `from` / `to` (ISO date strings) — time filter (`lastSeenAt` for cells, `endedAt` for traces).
- `traces` (`"0"` disables trace inclusion).
- `cells` (`"0"` disables cell inclusion).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/track-map/data?minHits=2&traceLimit=200&pointsPerTrace=2000"
```

**Response:** Cells are capped at **120,000** rows (`MAX_CELLS`), traces at **600** (`MAX_TRACES`), and the total decimated points across all traces at **400,000** (`MAX_TOTAL_POINTS`).
```json
{
  "settings": { "id": 1, "enabled": true, "cellSize": 8, "rotationDeg": 0, "studsPerKm": 1000 },
  "cells": { "x": [120.5], "z": [-30.2], "hits": [842], "avgSpeed": [22.4], "size": [8] },
  "traces": [
    { "id": "clxTrace1", "playerName": "somebody", "robloxId": "998877", "endedAt": "2026-09-13T09:40:00.000Z", "pts": [120.5, -30.2, 121.0, -29.8] }
  ],
  "stations": [{ "id": "clxMarker1", "stationId": "clxStation1", "name": "Budapest-Nyugati", "km": 0, "x": 0, "z": 0 }],
  "truncated": { "cells": false, "traces": false }
}
```
`truncated.cells` is `true` when the cell query hit `MAX_CELLS`; `truncated.traces` is `true` when the trace query returned at least `traceLimit` rows (and `traceLimit > 0`).

### `POST /api/admin/track-map/purge`

**Why:** The one irreversible button on the page — deletes accumulated track-map data. Each scope names exactly what it removes; nothing removes more than it names.

**Auth:** `VIEW_PERF_MONITOR`.

**Request:** JSON body:
- `scope` (required): `"all"` | `"traces"` | `"noise"` | `"older"` | `"driver"`.
- `minHits` (number, used only for `scope: "noise"`, default `2`, floored at `1`) — deletes cells with `hits < minHits`.
- `olderThanDays` (number, used only for `scope: "older"`, default `7`, floored at `0`) — deletes traces with `endedAt` older than the cutoff.
- `robloxId` (string, required for `scope: "driver"`) — deletes all traces for that driver.

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"scope":"noise","minHits":2}' \
  "$HSR_URL/api/admin/track-map/purge"
```

**Response varies by scope:**
- `all`: `{ "deleted": { "cells": 84213, "traces": 512 } }` (deletes every `TrackMapCell` and `TrackMapTrace`).
- `traces`: `{ "deleted": { "traces": 512 } }`.
- `noise`: `{ "deleted": { "cells": 1200 } }`.
- `older`: `{ "deleted": { "traces": 80 } }`.
- `driver`: `{ "deleted": { "traces": 12 } }`.

400 `{ "error": "Missing robloxId" }` (scope `driver` with no id) or `{ "error": "Unknown scope" }`. Every purge writes an `ActivityLog` row (`TRACK_MAP_PURGED`) describing what was removed.

### `GET /api/admin/track-map/stations`

**Why:** Loads every `Station` alongside its hand-marked pin position (if any) plus free-text custom markers (yards, junctions with no `Station` row) — the data behind the mapper's "click to mark a station" tool.

**Auth:** `VIEW_PERF_MONITOR`.

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/track-map/stations"
```

**Response:**
```json
{
  "stations": [
    { "id": "clxStation1", "name": "Budapest-Nyugati", "km": 0, "order": 1, "marker": { "id": "clxMarker1", "x": 10.5, "z": -4.2 } },
    { "id": "clxStation2", "name": "Kelenföld", "km": 8, "order": 2, "marker": null }
  ],
  "customMarkers": [{ "id": "clxMarker2", "label": "Ferencváros yard", "x": 40.1, "z": 12.0 }]
}
```

### `POST /api/admin/track-map/stations`

**Why:** Marks (or moves) where a `Station` sits on the drawn map, or drops a free-text pin for a yard/junction with no `Station` row. Re-marking a station moves its pin instead of stacking a second one.

**Auth:** `VIEW_PERF_MONITOR`.

**Request:** JSON body:
- `x`, `z` (numbers, required, must be finite).
- `stationId` (string, optional) — links the marker to a `Station`; upserts by `stationId` (one marker per station).
- `label` (string, optional, truncated to 80 chars) — used instead of `stationId` for a free-text pin.
- Either `stationId` or `label` is required.

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"stationId":"clxStation1","x":10.5,"z":-4.2}' \
  "$HSR_URL/api/admin/track-map/stations"
```

**Response:** 200 `{ "marker": { "id": "clxMarker1", "stationId": "clxStation1", "x": 10.5, "z": -4.2 } }`. 400 `{ "error": "Invalid coordinates" }` or `{ "error": "A station or a label is required" }`. 404 `{ "error": "Unknown station" }` if `stationId` does not exist.

### `DELETE /api/admin/track-map/stations`

**Why:** Removes a station marker or a custom pin from the map.

**Auth:** `VIEW_PERF_MONITOR`.

**Request:** Query params (one required):
- `id` — the marker's own id.
- `stationId` — deletes whatever marker is attached to that station.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/track-map/stations?id=clxMarker1"
```

**Response:** 200 `{ "ok": true }`. 400 `{ "error": "Missing id" }` if neither `id` nor `stationId` is given.

## Admin: Training Programmes & Content, Trains

### `GET /api/admin/trains`

**Why:** Feeds the timetable editor's train list (both directions), each with its stops and stations, so the editor can render the sortable per-direction lists and the stop-times grid.

**Auth:** None — no `requirePermission`/`requireAuth` call in this handler.

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/trains"
```

**Response:**
```json
[
  {
    "id": "cktr123",
    "trainNumber": "9412",
    "trainType": "IC",
    "locomotiveNum": "480 021",
    "locomotiveType": "Traxx",
    "direction": 0,
    "hasAC": true,
    "order": 3,
    "pairGroup": "9412-9413",
    "fordaGroup": null,
    "operator": "MAV",
    "composition": "6 kocsi",
    "specialNotes": null,
    "isTransfer": false,
    "linkedPairGroup": null,
    "active": true,
    "stops": [
      { "id": "cksp1", "stationId": "ckst1", "arrival": "08:34", "departure": "08:36", "station": { "id": "ckst1", "name": "Budapest-Keleti" } }
    ]
  }
]
```

### `POST /api/admin/trains`

**Why:** Creates a train row for the timetable — a service that can then be given stops, a pair group and a place in duty booking.

**Auth:** `MANAGE_TRAINS` flag (`requirePermission((p) => p.hasFlag('MANAGE_TRAINS'))`).

**Request:** JSON body. Required: `trainNumber`, `trainType`, `direction` (0/1), `order` (number). Optional: `locomotiveNum`, `locomotiveType`, `hasAC` (bool, default false), `pairGroup`, `fordaGroup`, `operator` (normalized via `normalizeTrainOperator()`), `composition`, `specialNotes`, `isTransfer` (bool, default false), `linkedPairGroup`.

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"trainNumber":"9412","trainType":"IC","direction":0,"order":3,"pairGroup":"9412-9413"}' \
  "$HSR_URL/api/admin/trains"
```

**Response:** `201` with the created `Train` row (same shape as the GET list items, minus `stops`). `400` if `trainNumber`/`trainType`/`direction`/`order` are missing. `500` on failure.

### `PUT /api/admin/trains`

**Why:** Edits an existing train's fields — locomotive, composition, pairing, active flag — from the admin form.

**Auth:** `MANAGE_TRAINS` flag.

**Request:** JSON body, `id` required. All other fields from the POST body are accepted and, when present, overwrite the corresponding column (`undefined` fields are left untouched by Prisma; empty-string values for optional string fields are coerced to `null`). `active` only applied when it is strictly boolean.

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"id":"cktr123","composition":"8 kocsi","active":true}' \
  "$HSR_URL/api/admin/trains"
```

**Response:** Updated `Train` row. `400` if `id` missing. `500` on failure (e.g. unknown id).

### `DELETE /api/admin/trains`

**Why:** Removes a train row entirely — for a service that never should have existed rather than one being retired (retiring is `PATCH action: "set-active"`).

**Auth:** `MANAGE_TRAINS` flag.

**Request:** Query param `id` (string, required).

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/trains?id=cktr123"
```

**Response:** `{ "success": true }`. `400` if `id` missing. `500` on failure.

### `PATCH /api/admin/trains`

**Why:** The two edits the timetable editor's train list makes without opening the full form: reordering trains within a direction by drag, and retiring/restoring a train without deleting it. Reordering is scoped to a direction because that is the column order the public timetable prints, and the two directions lay out independently (`Train.order` is not unique, unlike `Station.order`, so it can be written in one pass rather than the two-phase negative-order dance stations need).

**Auth:** `MANAGE_TRAINS` flag.

**Request:** JSON body with `action`:
- `"reorder"`: `direction` (0 or 1), `ids` (string array — must contain every train id of that direction exactly once, no duplicates, no unknowns).
- `"set-active"`: `id` (string), `active` (boolean).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"action":"reorder","direction":0,"ids":["cktr1","cktr2","cktr3"]}' \
  "$HSR_URL/api/admin/trains"

curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"action":"set-active","id":"cktr123","active":false}' \
  "$HSR_URL/api/admin/trains"
```

**Response:** Reorder: `{ "success": true }`, writes `order: 0..n-1` in the array's order inside one transaction. Set-active: the updated `Train` row. `400` for a bad `direction`, a non-array/non-string `ids`, or an `ids` array that doesn't list every train of the direction exactly once; `400` for `"Unknown action"` on anything else; `500` on failure.

### `PUT /api/admin/training/reorder`

**Why:** Drag-and-drop reordering for the training material editor's module and section lists. A section carrying `moduleId` in its item is how dragging a section across modules is expressed — the same write as a plain reorder, just landing in a different parent list, so a cross-module move and an in-place reorder don't need two endpoints.

**Auth:** `MANAGE_WIKI` flag.

**Request:** JSON body: `type` (`"modules"` or `"sections"`), `items`: array of `{ id: string, order: number, moduleId?: string }`.

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"type":"sections","items":[{"id":"cksec1","order":0},{"id":"cksec2","order":1,"moduleId":"ckmod2"}]}' \
  "$HSR_URL/api/admin/training/reorder"
```

**Response:** `{ "ok": true }`. `400` if `items` is not an array or `type` is not `"modules"`/`"sections"`.

### `GET /api/admin/training/trainees`

**Why:** The instructor-facing trainee roster. Each `TrainingEnrollment` row (one per member per programme) becomes one row here — reporting the two different questions that matter at each phase: attempts remaining before the closing exam, and supervisions completed after it. `phase` distinguishes `EXAM` (not yet passed), `SUPERVISION` (passed, programme runs supervisions), and `AWAITING_PROMOTION` (passed, programme has no supervision run — waiting on a human).

**Auth:** `canHostTraining()` (`HOST_TRAINING` flag) — checked after `requireAuth()`, returns `403` otherwise (not the standard `requirePermission` helper).

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/training/trainees"
```

**Response:**
```json
{
  "trainees": [
    {
      "id": "ckuser1",
      "branch": "DRIVER",
      "branchName": "Train Driver",
      "displayName": "János Kovács",
      "name": "janos_kovacs",
      "image": "https://...",
      "traineeSince": "2026-08-01T00:00:00.000Z",
      "daysLeft": 12,
      "minutesLeft": 1800,
      "minutesUsed": 1200,
      "hasMaterial": true,
      "isDeadlinePassed": false,
      "isHoursExhausted": false,
      "materialCompleted": true,
      "examPassed": true,
      "examPassedAt": "2026-09-01T10:00:00.000Z",
      "phase": "SUPERVISION",
      "examAttemptsUsed": 1,
      "examAttemptsLeft": 2,
      "examAttemptsMax": 3,
      "supervisionEnabled": true,
      "supervisionsCompleted": 2,
      "supervisionsRequired": 4,
      "supervisionSince": "2026-09-01T10:00:00.000Z",
      "supervisionDeadlineAt": "2026-10-01T10:00:00.000Z",
      "supervisionDaysLeft": 18,
      "supervisionDeadlineDays": 30,
      "isSupervisionDeadlinePassed": false,
      "lastSupervisionFailedAt": null,
      "canAutoPromote": true
    }
  ],
  "total": 1,
  "maxExamAttempts": 3,
  "programs": [{ "key": "DRIVER", "name": "Train Driver" }]
}
```

### `GET /api/admin/training/terminations`

**Why:** The review queue for `/admin/trainings` → Trainees' removal history. Only shows terminations still inside the retention window and not yet revoked — the same cutoff the revoke route enforces and the purge cron uses, so the page never offers a row that revoke would then refuse.

**Auth:** `canHostTraining()` (`HOST_TRAINING` flag), `403` otherwise.

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/training/terminations"
```

**Response:**
```json
{
  "terminations": [
    {
      "id": "cktm1",
      "userId": "ckuser1",
      "displayName": "János Kovács",
      "name": "janos_kovacs",
      "image": "https://...",
      "branch": "DRIVER",
      "reason": "DEADLINE",
      "note": null,
      "removedAt": "2026-09-05T00:00:00.000Z",
      "traineeSince": "2026-08-01T00:00:00.000Z",
      "minutesUsed": 900,
      "materialCompleted": false,
      "expiresInDays": 26,
      "reapplyInDays": 5,
      "dataPurgedAt": null
    }
  ],
  "retentionDays": 30,
  "grantDays": 7,
  "reapplyCooldownDays": 7
}
```

### `GET /api/admin/training/programs`

**Why:** Feeds `/admin/trainings` → Settings: the full list of `TrainingProgram` rows plus the guild's roles to pick from (ranks ship together with the programme data so a settings form never renders with empty rank selects). Also reports whether the list is a fallback to the built-in defaults (DB unreadable), and per-programme usage counts (skipped when falling back, since a real zero and "couldn't count" must not look the same).

**Auth:** `canManageInstructors()` (`MANAGE_INSTRUCTORS` flag).

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/training/programs"
```

**Response:**
```json
{
  "programs": [
    {
      "key": "DRIVER",
      "name": "Train Driver",
      "enabled": true,
      "hasMaterial": true,
      "supervisionEnabled": true,
      "traineeRoleId": "111",
      "fullRoleId": "222",
      "closingExamRoleId": "333",
      "qualifiedRoleIds": ["444"],
      "instructorRoleIds": ["555"],
      "promotionRoleIds": ["222"],
      "requiredSupervisions": 4,
      "deadlineDays": 30,
      "maxMinutes": 3000,
      "warningBeforeDays": 7,
      "supervisionDeadlineDays": 30,
      "traineeTeam": "TRAIN_DRIVER",
      "fullTeam": "TRAIN_DRIVER",
      "order": 0
    }
  ],
  "usage": { "DRIVER": { "enrollments": 12, "trainingProgress": 12, "examAttempts": 20 } },
  "fallback": false,
  "roles": [
    { "id": "111", "name": "Mozdonyvezető gyakornok", "rawName": "・Mozdonyvezető gyakornok", "color": 0, "position": 10, "countsAsRank": true }
  ]
}
```

### `POST /api/admin/training/programs`

**Why:** Creates a new career path (a `TrainingProgram` row) — what used to require a schema change and a deploy is now a form submission. The `key` is permanent: every `branch`/`track`/`applicationFormType` column across the schema stores it, so it is normalized once at creation and never editable afterwards.

**Auth:** `canManageInstructors()` (`MANAGE_INSTRUCTORS` flag).

**Request:** JSON body. `name` (string, required, trimmed). `key` (optional string — normalized via `normalizeProgramKey()`; derived from `name` if omitted). Any other `ProgramInput` fields (`description`, `enabled`, `hasMaterial`, `supervisionEnabled`, role ids, `requiredSupervisions`, `deadlineDays`, `maxMinutes`, `warningBeforeDays`, `supervisionDeadlineDays`, `traineeTeam`, `fullTeam`, `order`) are passed through to `createTrainingProgram()`.

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"name":"Ticket Inspector","hasMaterial":false}' \
  "$HSR_URL/api/admin/training/programs"
```

**Response:** `{ "program": { "key": "TICKET_INSPECTOR", "name": "Ticket Inspector", ... } }`. `400` if `name` is empty or the derived `key` contains no letter/digit. `409` if a programme with that key already exists.

### `PATCH /api/admin/training/programs/[key]`

**Why:** The whole edit surface for one programme on `/admin/trainings` → Settings — renaming, toggling `enabled`/`hasMaterial`/`supervisionEnabled`, re-pointing every rank, and re-tuning every numeric limit — all without a deploy, since rank ids and limits are guild/HR decisions rather than code.

**Auth:** `canManageInstructors()` (`MANAGE_INSTRUCTORS` flag).

**Request:** JSON body, all fields optional (only present ones are patched): `name` (non-empty string), `description` (string or null), `enabled`/`hasMaterial`/`supervisionEnabled` (boolean), `order` (number), `traineeRoleId`/`fullRoleId`/`closingExamRoleId` (string, or `""`/`null` to clear), `qualifiedRoleIds`/`instructorRoleIds`/`promotionRoleIds` (string arrays, deduplicated), `requiredSupervisions` (1–100), `deadlineDays` (1–3650), `maxMinutes` (1–100000), `warningBeforeDays` (integer ≥0 and < the effective `deadlineDays`), `supervisionDeadlineDays` (1–3650), `traineeTeam`/`fullTeam` (string or null).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"requiredSupervisions":5,"supervisionEnabled":true}' \
  "$HSR_URL/api/admin/training/programs/DRIVER"
```

**Response:** `{ "program": { ...updated row... } }`. `404` if the programme key doesn't resolve. `409` specifically when `traineeRoleId` is being cleared (set to `null`) on a programme that still has active enrollments — the response names the enrollment count and says to pick a different rank or disable the programme first instead.

### `DELETE /api/admin/training/programs/[key]`

**Why:** Deleting a programme with any history would orphan every row that stores its key as plain text (training progress, supervision records, exam rows) — those rows resolve to nothing and the trainees behind them become invisible with unreachable progress. Switching a programme off does everything a delete is usually wanted for while keeping the history readable.

**Auth:** `canManageInstructors()` (`MANAGE_INSTRUCTORS` flag).

**Request:** No body — `key` from the path.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/training/programs/TICKET_INSPECTOR"
```

**Response:** `{ "success": true }`. `404` if the programme doesn't exist. `409` — **refuses to delete a programme key with history** — if `trainingProgramUsage(key)` reports any nonzero count (enrollments, progress rows, exam attempts, etc.); the error names each blocking count and points at switching the programme off instead, and returns the `usage` object.

### `GET /api/admin/training/exam-votes`

**Why:** Read-only record for `/admin/trainings` → Exam votes: who was asked whether they want to sit today's closing exam, who wants to, the per-hour tally with the voters behind each bar, the winning hour and the slot it produced — for today or any day back.

**Auth:** `canHostTraining()` (`HOST_TRAINING` flag), `403` otherwise.

**Request:** Query param `date` (optional, `YYYY-MM-DD`; defaults to today in Budapest time). `400` if the date string doesn't parse.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/training/exam-votes?date=2026-09-12"
```

**Response:**
```json
{
  "date": "2026-09-12",
  "isToday": false,
  "phase": "DECIDED",
  "hours": { "daySend": 6, "dayClose": 12, "hourClose": 14 },
  "programs": [
    {
      "key": "DRIVER",
      "name": "Train Driver",
      "asked": 5,
      "yes": 3,
      "no": 1,
      "pending": 1,
      "invited": 3,
      "hourVotesCast": 3,
      "tally": [
        { "hour": 14, "count": 2, "voters": ["János Kovács", "Anna Tóth"] },
        { "hour": 16, "count": 1, "voters": ["Béla Szabó"] }
      ],
      "decidedAt": "2026-09-12T14:00:00.000Z",
      "decidedHour": 14,
      "slot": {
        "id": "ckslot1",
        "examId": "ckexam1",
        "examTitle": "Driver Closing Exam",
        "scheduledAt": "2026-09-12T14:00:00.000Z",
        "status": "COMPLETED",
        "registrations": 2
      }
    }
  ],
  "voters": [
    {
      "userId": "ckuser1",
      "name": "János Kovács",
      "image": "https://...",
      "response": true,
      "respondedAt": "2026-09-12T07:15:00.000Z",
      "programNames": ["Train Driver"],
      "hours": [{ "programName": "Train Driver", "hour": 14 }]
    }
  ]
}
```

### `GET /api/admin/training/modules`

**Why:** Loads one branch's whole training material tree (modules → sections → blocks) for the content editor.

**Auth:** `MANAGE_WIKI` flag.

**Request:** Query param `branch` (optional; parsed by `parseBranch()`, defaults to `"DRIVER"` if missing or unrecognized).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/training/modules?branch=DISPATCHER"
```

**Response:**
```json
[
  {
    "id": "ckmod1",
    "title": "Signalling Basics",
    "branch": "DISPATCHER",
    "order": 0,
    "sections": [
      {
        "id": "cksec1",
        "title": "Block Sections",
        "order": 0,
        "isCompletionGate": false,
        "blocks": [
          { "id": "ckblk1", "order": 0, "text": "...", "images": [], "videoUrl": null }
        ]
      }
    ]
  }
]
```

### `POST /api/admin/training/modules`

**Why:** Creates a new module (a top-level chapter) for a programme's training material.

**Auth:** `MANAGE_WIKI` flag.

**Request:** JSON body: `title` (string, required, trimmed), `branch` (optional; parsed by `parseBranch()`, defaults to `"DRIVER"`). New module is appended (`order` = last module's order in that branch + 1 — order is per branch, so the two materials number independently).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"title":"Emergency Procedures","branch":"DRIVER"}' \
  "$HSR_URL/api/admin/training/modules"
```

**Response:** `201` with the created `TrainingModule` row. `400` if `title` is empty.

### `POST /api/admin/training/upload`

**Why:** Uploads an image or short video for a training material block to Cloudflare R2, returning the public URL to embed.

**Auth:** `MANAGE_WIKI` flag.

**Request:** `multipart/form-data` with a single field `file`. Accepted MIME types (or inferred by extension if the browser didn't set one): images `image/jpeg`, `image/png`, `image/gif`, `image/webp`; videos `video/mp4`, `video/webm`.

```bash
curl -sS -X POST -b cookies.txt -F "file=@brake-test.png" \
  "$HSR_URL/api/admin/training/upload"
```

**Response:** `{ "url": "https://s3.hsrgames.hu/training-images/<uuid>.png", "type": "image" }`. `500` if R2 is not configured. `400` if no file or an unsupported type.

### `PUT /api/admin/training/blocks/[id]`

**Why:** Saves a content block's text/images/video inside the training editor, and cleans up any R2 files that were removed from the block in the same edit (URLs present in the old row but absent from the new payload are deleted from R2).

**Auth:** `MANAGE_WIKI` flag.

**Request:** JSON body: `text` (string or null), `images` (array of URL strings, max 5, or null), `videoUrl` (string or null).

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"text":"Check the brake pressure gauge.","images":["https://s3.hsrgames.hu/training-images/a.png"],"videoUrl":null}' \
  "$HSR_URL/api/admin/training/blocks/ckblk1"
```

**Response:** Updated `TrainingSectionBlock` row. `400` if `images` has more than 5 entries.

### `DELETE /api/admin/training/blocks/[id]`

**Why:** Deletes a content block and any R2 files (images/video) it referenced.

**Auth:** `MANAGE_WIKI` flag.

**Request:** No body — `id` from the path.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/training/blocks/ckblk1"
```

**Response:** `{ "ok": true }`.

### `PUT /api/admin/training/sections/[id]`

**Why:** Renames a section and/or sets it as the branch's completion gate. Only one gate is allowed per training branch (progress completion — `allSectionsCompleted` — is counted per branch), so setting a gate here clears any other section in the same branch that was previously marked as one. A trainee who had already passed this section before it became the gate has their completion backfilled (`backfillGateCompletion()`) rather than being forced to revisit it.

**Auth:** `MANAGE_WIKI` flag.

**Request:** JSON body: `title` (string, required, trimmed), `isCompletionGate` (boolean, optional, coerced with `!!`).

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"title":"Final Assessment","isCompletionGate":true}' \
  "$HSR_URL/api/admin/training/sections/cksec1"
```

**Response:** Updated `TrainingSection` row with its `blocks`. `400` if `title` is empty.

### `DELETE /api/admin/training/sections/[id]`

**Why:** Deletes a section and every image/video its blocks referenced in R2.

**Auth:** `MANAGE_WIKI` flag.

**Request:** No body — `id` from the path.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/training/sections/cksec1"
```

**Response:** `{ "ok": true }`.

### `POST /api/admin/training/sections/[id]/blocks`

**Why:** Appends a new content block to the end of a section.

**Auth:** `MANAGE_WIKI` flag.

**Request:** JSON body: `text` (string or null), `images` (array, max 5, or null), `videoUrl` (string or null). `sectionId` comes from the path.

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"text":"New paragraph."}' \
  "$HSR_URL/api/admin/training/sections/cksec1/blocks"
```

**Response:** `201` with the created `TrainingSectionBlock` row (`order` = last block's order in the section + 1). `400` if `images` has more than 5 entries.

### `PUT /api/admin/training/modules/[id]`

**Why:** Renames a module and/or moves it to another programme's branch. Moving carries the sections and blocks with it (they hang off the module) but re-derives `order` (appended to the end of the target branch's list) since the old position would otherwise collide with whatever already occupies it there.

**Auth:** `MANAGE_WIKI` flag.

**Request:** JSON body, both fields optional but at least one must produce a change: `title` (non-empty string if provided), `branch` (parsed by `parseBranch()`; only triggers a move if it differs from the module's current branch).

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"branch":"DISPATCHER"}' \
  "$HSR_URL/api/admin/training/modules/ckmod1"
```

**Response:** Updated `TrainingModule` row. `400` for an unrecognized `branch` value, an empty `title`, or if neither field results in any data to update. `404` if the module doesn't exist (only checked when a branch move is requested).

### `DELETE /api/admin/training/modules/[id]`

**Why:** Deletes a module and, cascading through Prisma relations, its sections and blocks; cleans up R2 files that are no longer referenced by anything else afterward — deliberately deferred until after the delete, since a duplicated module shares its original's media URLs and deleting the copy must not delete the original's files.

**Auth:** `MANAGE_WIKI` flag.

**Request:** No body — `id` from the path.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/training/modules/ckmod1"
```

**Response:** `{ "ok": true }`.

### `POST /api/admin/training/modules/[id]/duplicate`

**Why:** Copies a whole module (its sections and blocks) either in place within the same programme (e.g. "start a variant from this module") or into a different programme's branch (e.g. driver material that dispatchers also need). One endpoint serves both because they are the same operation with one optional argument. The copy references the same R2 files as the original rather than re-uploading them — deletion elsewhere only removes files nothing else points at.

**Auth:** `MANAGE_WIKI` flag.

**Request:** JSON body, both optional: `branch` (parsed by `parseBranch()`; defaults to the source module's own branch, i.e. duplicate in place), `title` (string; defaults to `"<source title> (másolat)"` when duplicating in place, or the source's own title unchanged when moving to another branch).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"branch":"DISPATCHER"}' \
  "$HSR_URL/api/admin/training/modules/ckmod1/duplicate"
```

**Response:** `201` with the newly created `TrainingModule` row (sections/blocks copied inside one transaction — a module that ended up with only half its sections would look like a finished copy). `404` if the source module doesn't exist. `400` for an unrecognized `branch`.

### `POST /api/admin/training/modules/[id]/sections`

**Why:** Appends a new section to the end of a module, optionally marking it as the branch's completion gate (which, like the sibling section-level route, clears any other section in the same branch previously holding that flag).

**Auth:** `MANAGE_WIKI` flag.

**Request:** JSON body: `title` (string, required, trimmed), `isCompletionGate` (boolean, optional). `moduleId` comes from the path.

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"title":"Signal Aspects"}' \
  "$HSR_URL/api/admin/training/modules/ckmod1/sections"
```

**Response:** `201` with the created `TrainingSection` row including `blocks` (empty array). `400` if `title` is empty.

### `POST /api/admin/training/terminations/[id]/revoke`

**Why:** Reverses a training removal within its review window, restoring the trainee's progress snapshot and handing back exactly `TRAINING_REVOKE_GRANT_DAYS` (7) of deadline rather than a full fresh run — undoing a mistaken or since-resolved removal (missed deadline, role sync glitch) without erasing the fact that it happened.

**Auth:** `canHostTraining()` (`HOST_TRAINING` flag), `403` otherwise.

**Request:** JSON body: `note` (string, optional, trimmed — appended to the termination's existing note). `id` (termination id) comes from the path.

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"note":"Deadline missed due to a documented illness."}' \
  "$HSR_URL/api/admin/training/terminations/cktm1/revoke"
```

**Response:** `{ "success": true, "grantDays": 7 }`. `404` if the termination doesn't exist. `409` if already revoked, if the user is already an active trainee of that programme, or if the programme no longer exists. `409` also if the programme has no `traineeRoleId` configured (nothing to grant back). `410` if the termination is past the retention/review window and can no longer be revoked. `502` if granting the Discord role back fails (nothing is changed in that case — Discord is updated before the database, so a role grant that fails must not be recorded as a successful revoke).

## Admin: Type Trainings, Users, Wagon Pool, Wrapped

### `GET /api/admin/users`

**Why:** The admin member list/search that backs `/admin/users` — finding a member to ban, suspend, pull into Discord or otherwise manage.

**Auth:** `MANAGE_USERS`.

**Request:** Query params, all optional: `search` (matches `name`, `displayName`, `robloxId`, `discordUsername` via `contains`), `page` (default `1`), `limit` (default `20`).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/users?search=janos&page=1&limit=20"
```

**Response:**
```json
{
  "users": [
    {
      "id": "clx123",
      "robloxId": "123456789",
      "name": "JanosKovacs",
      "displayName": "Janos",
      "image": "https://tr.rbxcdn.com/...",
      "discordId": "987654321",
      "discordUsername": "janos.k",
      "discordRole": "Train driver",
      "role": "USER",
      "banned": false,
      "isActiveInGame": false,
      "createdAt": "2024-05-01T12:00:00.000Z",
      "lastLoginAt": "2026-09-12T08:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "limit": 20, "total": 812, "pages": 41 }
}
```
Error: `500` on failure.

---

### `GET /api/admin/users/[id]`

**Why:** Single-user lookup for the admin user detail page — the same shape as the list row.

**Auth:** `MANAGE_USERS`.

**Request:** Path param `id` (User id). No body.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/users/clx123"
```

**Response:**
```json
{
  "user": {
    "id": "clx123",
    "robloxId": "123456789",
    "name": "JanosKovacs",
    "displayName": "Janos",
    "image": "https://tr.rbxcdn.com/...",
    "discordId": "987654321",
    "discordUsername": "janos.k",
    "discordRole": "Train driver",
    "role": "USER",
    "banned": false,
    "isActiveInGame": false,
    "createdAt": "2024-05-01T12:00:00.000Z",
    "lastLoginAt": "2026-09-12T08:00:00.000Z"
  }
}
```
Errors: `404` User not found, `500` on failure.

---

### `POST /api/admin/users/ban`

**Why:** Takes the whole account, not just one service branch — the blunt instrument the Service Suspensions feature was built to replace for cases that don't fit "not this branch, until Friday". Invalidates every session so a banned member is logged out everywhere immediately.

**Auth:** `MANAGE_USERS`. Rank-hierarchy guard: cannot ban yourself (`userId === session.id`), cannot ban another `role: "ADMIN"` account, and cannot ban someone whose Discord rank (resolved through `getRoleHierarchy()`, never the stored role name) has priority `>=` the actor's own priority.

**Request:** JSON body: `userId` (string, required), `reason` (string, optional).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"userId":"clx123","reason":"Repeated automod violations"}' \
  "$HSR_URL/api/admin/users/ban"
```

**Response:**
```json
{ "success": true }
```
Errors: `400` "User ID is required" / "You cannot ban yourself", `403` "You cannot ban another admin" / "You cannot ban someone with the same or higher rank", `404` User not found, `500` on failure. Effects: `banned=true`, `bannedAt`, `bannedReason`, `bannedBy`, `sessionVersion` incremented; writes `BAN_USER` (actor) and `BANNED` (target) to `ActivityLog`.

---

### `POST /api/admin/users/unban`

**Why:** Reverses a ban — clears the ban fields so the account can sign in again.

**Auth:** `MANAGE_USERS`. No rank guard (lifting a ban is not treated as the same risk as issuing one).

**Request:** JSON body: `userId` (string, required).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"userId":"clx123"}' \
  "$HSR_URL/api/admin/users/unban"
```

**Response:**
```json
{ "success": true }
```
Errors: `400` "User ID is required", `500` on failure. Effects: `banned=false`, `bannedAt/bannedReason/bannedBy` cleared; writes `UNBAN_USER` (actor) and `UNBANNED` (target) to `ActivityLog`.

---

### `POST /api/admin/users/logout`

**Why:** Force-logout without a ban — revokes the member's web session validity and mobile refresh tokens, e.g. after a password/account-security concern that doesn't warrant banning.

**Auth:** `MANAGE_USERS`. No rank guard.

**Request:** JSON body: `userId` (string, required).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"userId":"clx123"}' \
  "$HSR_URL/api/admin/users/logout"
```

**Response:**
```json
{ "success": true }
```
Errors: `400` "User ID is required", `500` on failure. Effects: `sessionVersion` incremented (invalidates all existing web JWTs), all of the user's non-revoked `MobileSession` rows get `revokedAt` set; writes `FORCE_LOGOUT` to `ActivityLog` for both actor and target.

---

### `POST /api/admin/users/sync-roles`

**Why:** Full-guild Discord role resync — refreshes the `DiscordGuildRole` hierarchy mirror and every linked user's `discordRoles`/`discordRole`/admin flag/duty-role timers/programme enrollments in one pass, e.g. after ranks were reordered on Discord or the sync drifted.

**Auth:** `MANAGE_USERS`.

**Request:** No body. Requires `DISCORD_BOT_TOKEN` and `DISCORD_GUILD_ID` env vars to be configured.

```bash
curl -sS -X POST -b cookies.txt "$HSR_URL/api/admin/users/sync-roles"
```

**Response:**
```json
{ "success": true, "synced": 640, "notInGuild": 12, "ranks": 38 }
```
Errors: `500` "Discord bot not configured" (missing env), `502` "Could not read the full Discord member list — nothing was changed." (a partial crawl aborts entirely, rather than silently demoting members it never saw), `500` generic failure. Writes `SYNC_ROLES` to `ActivityLog`.

---

### `POST /api/admin/users/sync-dnr`

**Why:** Batched refresh of display-and-recognition data (Roblox name/displayName/avatar, Discord username/avatar) for users, walked in pages of 100 from the client so a run can resume after a dropped request instead of restarting from user 0.

**Auth:** `MANAGE_USERS`.

**Request:** JSON body, all optional: `cursor` (string, the last processed user id, to continue a run), `syncedSoFar` / `errorsSoFar` (numbers, running totals the client carries forward for the final activity-log line).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"cursor":null,"syncedSoFar":0,"errorsSoFar":0}' \
  "$HSR_URL/api/admin/users/sync-dnr"
```

**Response:**
```json
{
  "total": 812,
  "processed": 100,
  "synced": 94,
  "errors": 0,
  "nextCursor": "clx456",
  "done": false,
  "lastUsername": "JanosKovacs"
}
```
`done: true` (batch smaller than 100) triggers the `SYNC_DNR` `ActivityLog` write with cumulative totals. No auth-specific errors beyond the standard `requirePermission` 401/403.

---

### `POST /api/admin/users/pull-discord`

**Why:** Bulk-adds every linked user who has a stored Discord OAuth refresh token into the Discord guild via the OAuth2 "add guild member" API, walked in batches of 25 with a cursor so a proxy timeout on thousands of users doesn't restart the whole run.

**Auth:** `MANAGE_USERS`.

**Request:** JSON body, optional: `cursor` (string, last processed user id).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"cursor":null}' \
  "$HSR_URL/api/admin/users/pull-discord"
```

**Response:**
```json
{
  "total": 300,
  "processed": 25,
  "pulled": 20,
  "alreadyIn": 4,
  "failed": 1,
  "nextCursor": "clx789",
  "done": false,
  "lastUsername": "JanosKovacs"
}
```
Writes `USER_DISCORD_PULLED` to `ActivityLog` per batch when `pulled > 0`. Only users with both `discordId` and `discordRefreshToken` set are ever candidates, so `NOT_LINKED`/`NO_TOKEN` results cannot occur here.

---

### `POST /api/admin/users/[id]/pull-discord`

**Why:** Single-user version of the bulk pull, for a "pull into Discord" button on one member's admin page.

**Auth:** `MANAGE_USERS`.

**Request:** Path param `id`. No body.

```bash
curl -sS -X POST -b cookies.txt "$HSR_URL/api/admin/users/clx123/pull-discord"
```

**Response:**
```json
{ "result": "PULLED" }
```
Errors: `404` User not found, `400` "This user has no Discord account linked" (`result: "NOT_LINKED"`) / "This user has not logged in with Discord, so there is no grant to add them with" (`result: "NO_TOKEN"`), `502` "Discord refused the request" (`result: "FAILED"`). Writes `USER_DISCORD_PULLED` to `ActivityLog` only on `PULLED`.

---

### `GET /api/admin/users/[id]/suspensions`

**Why:** Lists every suspension (active, lifted, and expired) for a member's Driver/Dispatcher/Inspector service branches — see the Service Suspensions section: this is the record that answers "has this member been off the seat before," which is why rows are never deleted.

**Auth:** `MANAGE_USERS`.

**Request:** Path param `id` (User id). No body.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/users/clx123/suspensions"
```

**Response:**
```json
{
  "suspensions": [
    {
      "id": "susp_1",
      "scope": "DRIVER",
      "reason": "Ignored a red signal during supervision",
      "expiresAt": "2026-09-20T06:00:00.000Z",
      "liftedAt": null,
      "liftReason": null,
      "createdAt": "2026-09-13T09:00:00.000Z",
      "active": true,
      "issuedBy": "Head of Education",
      "liftedBy": null
    }
  ]
}
```

---

### `POST /api/admin/users/[id]/suspensions`

**Why:** Issues a branch-scoped service suspension (`DRIVER`/`DISPATCHER`/`INSPECTOR`) — the alternative to a full ban or a silent permission REVOKE described in the Service Suspensions section. Ends the member's active duties in that branch only (the other branch is left running) and pushes a notification.

**Auth:** `MANAGE_USERS`. Rank-hierarchy guard (`assertMaySuspend`): cannot suspend yourself, cannot suspend a `role: "ADMIN"` account, and cannot suspend someone whose Discord rank priority (via `getRoleHierarchy()`) is `>=` the actor's own.

**Request:** Path param `id`. JSON body: `scope` (`"DRIVER" | "DISPATCHER" | "INSPECTOR"`, required), `reason` (string, required, trimmed non-empty), and one of: `expiresAt` (ISO date string, must be in the future within a 1-minute grace and no more than 10 years out) or `minutes` (positive number) or legacy `days` (positive number, converted to minutes) — omitting all three means open-ended (`expiresAt: null`).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"scope":"DRIVER","reason":"Ignored a red signal during supervision","minutes":10080}' \
  "$HSR_URL/api/admin/users/clx123/suspensions"
```

**Response:**
```json
{
  "success": true,
  "suspension": {
    "id": "susp_1",
    "scope": "DRIVER",
    "reason": "Ignored a red signal during supervision",
    "expiresAt": "2026-09-20T09:00:00.000Z",
    "createdAt": "2026-09-13T09:00:00.000Z"
  },
  "endedDuties": 1
}
```
Errors: `400` "You cannot suspend yourself" / invalid scope / missing reason / invalid or too-far/past expiry, `403` "You cannot suspend another admin" / "You cannot suspend someone with the same or higher rank", `409` "This user is already suspended from {branch} service." Writes `SUSPENSION_ISSUED` (actor) and `SUSPENDED` (target) to `ActivityLog`; pushes a notification to the target (fire-and-forget).

---

### `DELETE /api/admin/users/[id]/suspensions/[suspensionId]`

**Why:** Lifts a suspension early. Updates the row rather than deleting it — the reason field is the whole point, per the Service Suspensions section.

**Auth:** `MANAGE_USERS`. Same `assertMaySuspend` rank-hierarchy guard as issuing.

**Request:** Path params `id` (User id), `suspensionId`. JSON body, optional: `reason` (string, the lift reason).

```bash
curl -sS -X DELETE -b cookies.txt -H "Content-Type: application/json" \
  -d '{"reason":"Re-briefed on signal procedure"}' \
  "$HSR_URL/api/admin/users/clx123/suspensions/susp_1"
```

**Response:**
```json
{ "success": true }
```
Errors: `400` "You cannot suspend yourself" (guard reused for lift), `403` rank guard failures, `404` "Suspension not found" (also scoped to `userId` in the path, so a suspension id from elsewhere can't be lifted through a member the actor outranks), `409` "This suspension has already been lifted". Writes `SUSPENSION_LIFTED` to `ActivityLog` for both actor and target; pushes a notification to the target.

---

### `GET /api/admin/wrapped`

**Why:** Backs the `/admin/wrapped` dashboard — delivery counts for a year's HSR Wrapped run plus how many members are currently eligible to receive one. See the HSR Wrapped section.

**Auth:** `MANAGE_WRAPPED`.

**Request:** Query param `year` (integer, optional; must be between 2020 and next calendar year, else defaults to the current UTC year).

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/wrapped?year=2025"
```

**Response:**
```json
{
  "year": 2025,
  "total": 640,
  "sent": 590,
  "pending": 40,
  "failed": 10,
  "eligible": 645
}
```
(exact fields of `wrappedStatus()` besides `eligible` depend on `src/lib/profile/wrapped-delivery.ts`; shown here as returned via spread.)

---

### `POST /api/admin/wrapped`

**Why:** Drives the three Wrapped admin actions: preview a single member's card without sending, queue `WrappedDelivery` rows for a year (the durable send queue described in the HSR Wrapped section), or queue-and-drain a batch of 40 sends immediately (the rest is finished by the `/api/cron/wrapped` cron).

**Auth:** `MANAGE_WRAPPED`.

**Request:** JSON body: `action` (`"preview" | "queue" | "send"`, required), `year` (integer, optional, defaults to current UTC year), `userId` (string, optional, only used by `preview`; defaults to the requesting admin).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"action":"preview","year":2025,"userId":"clx123"}' \
  "$HSR_URL/api/admin/wrapped"

curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"action":"queue","year":2025}' \
  "$HSR_URL/api/admin/wrapped"

curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"action":"send","year":2025}' \
  "$HSR_URL/api/admin/wrapped"
```

**Response (`preview`):**
```json
{
  "subject": "A Te 2025-ös HSR Wrapped összefoglalód",
  "html": "<html>...</html>",
  "data": { "distanceKm": 4210, "hours": 132, "badgesEarned": 3, "praiseReceived": 7 }
}
```
**Response (`queue`):**
```json
{ "ok": true, "total": 645, "queued": 40, "year": 2025, "sent": 590, "pending": 40, "failed": 10, "eligible": 645 }
```
**Response (`send`):**
```json
{ "ok": true, "queued": 0, "sent": 40, "remaining": 0, "year": 2025, "total": 645, "eligible": 645 }
```
Errors: `400` "Invalid body" (bad JSON), `404` "This member had no activity that year." (preview with no data), `400` "Unknown action". `queue` writes `WRAPPED_QUEUED`, `send` writes `WRAPPED_SENT` to `ActivityLog`.

---

### `POST /api/admin/wagon-pool/chips`

**Why:** Creates a wagon chip — a draggable wagon identifier in the `/admin/locomotive-pool` wagon board, the wagon-side equivalent of a locomotive chip.

**Auth:** `MANAGE_LOCOMOTIVE_POOL`.

**Request:** JSON body: `number` (string, required, trimmed non-empty, must be unique).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"number":"Bpmz-1234"}' \
  "$HSR_URL/api/admin/wagon-pool/chips"
```

**Response (`201`):**
```json
{ "id": "wc_1", "number": "Bpmz-1234", "imageUrl": null }
```
Errors: `400` "number is required", `409` "This wagon number already exists", `500` on failure. Writes `WAGON_CHIP_CREATED` to `ActivityLog`.

---

### `PATCH /api/admin/wagon-pool/chips/[id]`

**Why:** Sets or clears the in-game match override text and/or operator (MÁV/GYSEV) on a wagon chip, so the same physical wagon can be re-labelled without deleting and recreating it.

**Auth:** `MANAGE_LOCOMOTIVE_POOL`.

**Request:** Path param `id`. JSON body, both optional (only applied if the key is present): `matchOverride` (string or `null` — empty/whitespace string is stored as `null`), `operator` (`"MAV" | "GYSEV"` or anything else, which is stored as `null`).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"matchOverride":"Bpmz 1234","operator":"MAV"}' \
  "$HSR_URL/api/admin/wagon-pool/chips/wc_1"
```

**Response:**
```json
{ "id": "wc_1", "matchOverride": "Bpmz 1234", "operator": "MAV" }
```
Errors: `500` on failure (including a not-found id, since `prisma.update` throws).

---

### `DELETE /api/admin/wagon-pool/chips/[id]`

**Why:** Removes a wagon chip from the pool entirely, cascading its pair assignments — used when a wagon is retired from the roster.

**Auth:** `MANAGE_LOCOMOTIVE_POOL`.

**Request:** Path param `id`. No body.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/wagon-pool/chips/wc_1"
```

**Response:**
```json
{ "success": true }
```
Errors: `500` on failure. Writes `WAGON_CHIP_DELETED` to `ActivityLog`.

---

### `POST /api/admin/wagon-pool/chips/[id]/image`

**Why:** Uploads/replaces the reference photo for a wagon chip, stored in Cloudflare R2 — the visual identifier shown on the chip in the pool board.

**Auth:** `MANAGE_LOCOMOTIVE_POOL`.

**Request:** Path param `id`. `multipart/form-data` with a `file` field: JPEG/PNG/GIF/WebP only, max 10MB. Requires R2 to be configured.

```bash
curl -sS -X POST -b cookies.txt \
  -F "file=@wagon-photo.jpg;type=image/jpeg" \
  "$HSR_URL/api/admin/wagon-pool/chips/wc_1/image"
```

**Response:**
```json
{ "imageUrl": "https://s3.hsrgames.hu/wagon-pool/wc_1/3f2a9c7e-....jpg" }
```
Errors: `500` "R2 storage not configured", `404` "Chip not found", `400` "No file provided" / "Invalid file type. Allowed: JPEG, PNG, GIF, WebP" / "File too large. Maximum 10MB".

---

### `POST /api/admin/wagon-pool/assignments`

**Why:** Drop event — appends a wagon chip onto the end of a train pair's fixed consist sequence. Always creates a new position rather than upserting, because the same wagon type may legitimately repeat within one consist.

**Auth:** `MANAGE_LOCOMOTIVE_POOL`.

**Request:** JSON body: `pairGroup` (string, required, trimmed non-empty — the `Train.pairGroup` identifier), `wagonId` (string, required — a `WagonChip` id).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"pairGroup":"IC-101/102","wagonId":"wc_1"}' \
  "$HSR_URL/api/admin/wagon-pool/assignments"
```

**Response (`201`):**
```json
{ "id": "pwa_1", "pairGroup": "IC-101/102", "wagonId": "wc_1", "order": 3 }
```
Errors: `400` "pairGroup and wagonId are required", `404` "Chip not found", `500` on failure. Writes `WAGON_ASSIGNMENT_CREATED` to `ActivityLog`.

---

### `PATCH /api/admin/wagon-pool/assignments`

**Why:** Reorders the fixed consist sequence for a pair — `order` on each `PairWagonAssignment` row *is* the expected wagon position, so this is how a drag-and-drop reorder in the admin UI is persisted.

**Auth:** `MANAGE_LOCOMOTIVE_POOL`.

**Request:** JSON body: `pairGroup` (string, required, trimmed non-empty), `orderedAssignmentIds` (array of strings, required non-empty — the assignment ids in their new order; written in one transaction as `order = index`).

```bash
curl -sS -X PATCH -b cookies.txt -H "Content-Type: application/json" \
  -d '{"pairGroup":"IC-101/102","orderedAssignmentIds":["pwa_3","pwa_1","pwa_2"]}' \
  "$HSR_URL/api/admin/wagon-pool/assignments"
```

**Response:**
```json
{ "success": true }
```
Errors: `400` "pairGroup and orderedAssignmentIds are required", `500` on failure.

---

### `DELETE /api/admin/wagon-pool/assignments/[id]`

**Why:** Removes a single wagon-to-pair assignment (not the chip itself) and closes the resulting gap in the ordering so the remaining positions stay a contiguous `0..n-1` sequence.

**Auth:** `MANAGE_LOCOMOTIVE_POOL`.

**Request:** Path param `id` (assignment id). No body.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/wagon-pool/assignments/pwa_1"
```

**Response:**
```json
{ "success": true }
```
Errors: `500` on failure (including not-found, since `prisma.delete` throws). Writes `WAGON_ASSIGNMENT_DELETED` to `ActivityLog`.

---

### `GET /api/admin/type-trainings`

**Why:** Lists every locomotive type training session (with registration counts) for the `/admin/trainings` type-training tab.

**Auth:** `HOST_TRAINING`.

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/type-trainings"
```

**Response:**
```json
[
  {
    "id": "tt_1",
    "title": "M41 típusoktatás",
    "roleIds": ["1111111111"],
    "scheduledAt": "2026-09-20T17:00:00.000Z",
    "channelId": "1464334153863397663",
    "discordMessageId": "1464999999999999999",
    "hostId": "clxHost",
    "status": "SCHEDULED",
    "_count": { "registrations": 6 }
  }
]
```

---

### `POST /api/admin/type-trainings`

**Why:** Creates and announces a new type training session — the host is recorded as the announcer, which is what the Type Instructor weekly "at least one training" quota (Instructor Grades section) is counted from. Posts a Discord embed with a registration link.

**Auth:** `HOST_TRAINING`.

**Request:** JSON body: `title` (string, required), `roleIds` (non-empty array of non-empty strings, required — the Discord ranks granted to attendees on close), `scheduledAt` (ISO date string, required, must parse to a valid date).

```bash
curl -sS -X POST -b cookies.txt -H "Content-Type: application/json" \
  -d '{"title":"M41 típusoktatás","roleIds":["1111111111"],"scheduledAt":"2026-09-20T17:00:00.000Z"}' \
  "$HSR_URL/api/admin/type-trainings"
```

**Response (`201`):**
```json
{
  "id": "tt_1",
  "title": "M41 típusoktatás",
  "roleIds": ["1111111111"],
  "scheduledAt": "2026-09-20T17:00:00.000Z",
  "channelId": "1464334153863397663",
  "discordMessageId": "1464999999999999999",
  "hostId": "clxHost",
  "status": "SCHEDULED"
}
```
Errors: `400` "title, roleIds (non-empty array) and scheduledAt required" / "roleIds must be an array of non-empty strings" / "Invalid scheduledAt". Writes `TYPE_TRAINING_CREATED` to `ActivityLog`. The Discord announcement send is best-effort — if it fails, `discordMessageId` is simply not set (no error returned to the caller).

---

### `GET /api/admin/type-trainings/role-options`

**Why:** Lists the Discord ranks offerable as a type-training reward — restricted to roles already used to gate locomotives in the locomotive pool (`LocomotiveChipRole`), so the two role sets stay in sync rather than admins picking arbitrary Discord ranks.

**Auth:** `HOST_TRAINING`.

**Request:** No params.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/type-trainings/role-options"
```

**Response:**
```json
[
  { "id": "1111111111", "name": "M41 mozdonyvezető" },
  { "id": "2222222222", "name": "M62 mozdonyvezető" }
]
```

---

### `GET /api/admin/type-trainings/[id]`

**Why:** Detail view of one type training, including its registrations and each registrant's basic profile — backs the attendance-marking screen.

**Auth:** `HOST_TRAINING`.

**Request:** Path param `id`. No body.

```bash
curl -sS -b cookies.txt "$HSR_URL/api/admin/type-trainings/tt_1"
```

**Response:**
```json
{
  "id": "tt_1",
  "title": "M41 típusoktatás",
  "roleIds": ["1111111111"],
  "scheduledAt": "2026-09-20T17:00:00.000Z",
  "status": "SCHEDULED",
  "registrations": [
    {
      "id": "reg_1",
      "trainingId": "tt_1",
      "userId": "clx123",
      "attended": null,
      "user": { "id": "clx123", "displayName": "Janos", "name": "JanosKovacs", "image": "https://tr.rbxcdn.com/..." }
    }
  ]
}
```
Errors: `404` "Not found".

---

### `DELETE /api/admin/type-trainings/[id]`

**Why:** Deletes a type training outright — also deletes the Discord announcement message if one was posted, so a cancelled session doesn't leave a stale "sign up here" post.

**Auth:** `HOST_TRAINING`.

**Request:** Path param `id`. No body.

```bash
curl -sS -X DELETE -b cookies.txt "$HSR_URL/api/admin/type-trainings/tt_1"
```

**Response:**
```json
{ "ok": true }
```
Errors: `404` "Not found". Writes `TYPE_TRAINING_DELETED` to `ActivityLog`.

---

### `PUT /api/admin/type-trainings/[id]/attendance`

**Why:** Records who actually showed up to a type training, ahead of closing it — attendance is what decides who receives the reward rank on close.

**Auth:** `HOST_TRAINING`.

**Request:** Path param `id`. JSON body: `attendance` (object, required — map of `userId` → `boolean`).

```bash
curl -sS -X PUT -b cookies.txt -H "Content-Type: application/json" \
  -d '{"attendance":{"clx123":true,"clx456":false}}' \
  "$HSR_URL/api/admin/type-trainings/tt_1/attendance"
```

**Response:**
```json
{ "ok": true }
```
Errors: `400` "attendance object required". Each entry does an `updateMany` scoped to `trainingId` + `userId`, so an id not actually registered on this training is silently a no-op.

---

### `POST /api/admin/type-trainings/[id]/close`

**Why:** Finishes a type training: deletes its Discord announcement, marks it `COMPLETED`, and grants every `roleIds` rank to every registrant marked `attended`. This is the step that turns attendance into an actual Discord promotion.

**Auth:** `HOST_TRAINING`.

**Request:** Path param `id`. No body. The training must currently be `status: "SCHEDULED"`.

```bash
curl -sS -X POST -b cookies.txt "$HSR_URL/api/admin/type-trainings/tt_1/close"
```

**Response:**
```json
{ "ok": true, "granted": 6 }
```
Errors: `404` "Not found", `400` "Training is not open" (already closed). Writes `TYPE_TRAINING_CLOSED` to `ActivityLog`. Role grants to attendees without a `discordId` are skipped (filtered out before the grant loop).

## Cron Jobs

All routes below require the `CRON_API_KEY` (or, for one legacy route, `CRON_SECRET`) shared secret and are meant to be called only by the server's own cron scheduler (`scripts/crontab.txt`), never by a browser or a mobile/game client. They are listed in the same order as the "Cron Jobs" table in `CLAUDE.md`.

### `POST /api/cron/missed-assignments`

**Why:** Finds yesterday's APPROVED driver `ScheduleRequest`s that never turned into a FULFILLED duty, records a `MissedAssignment` (with an appeal token) for each, and emails both the driver and every `MANAGE_DUTY` admin. This has to run once, after the day is over and fulfillment is known, rather than in-request — nothing in the request lifecycle observes "a booked driver never showed up".

**Auth:** `x-api-key` header must equal `process.env.CRON_API_KEY`, checked directly in the handler. Server-to-server only.

**Request:** No query parameters — always processes "yesterday" relative to the moment it runs.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/missed-assignments"
```

**Response:**
```json
{
  "message": "Missed assignments emails sent",
  "missedCount": 3,
  "adminCount": 5,
  "sent": 5,
  "userCount": 3,
  "userSent": 3
}
```
If there were no bookings at all yesterday: `{ "message": "No schedules for today", "sent": 0 }` (note: this message is emitted even though the check is for yesterday's bookings feeding the missed-assignment computation). If there were bookings but none missed: `{ "message": "No missed assignments yesterday", "sent": 0 }`. Bad key → 401 `{ "error": "Unauthorized" }`. Failure → 500 `{ "error": "Failed to send missed assignments emails" }`.

### `POST /api/cron/daily-schedule`

**Why:** Emails every driver/dispatcher with an APPROVED `ScheduleRequest` for today a summary of their day's assignments, grouped per user. A schedule that changes throughout the evening before needs one authoritative morning-of digest rather than a page nobody opens.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`. Server-to-server only.

**Request:** No query parameters — always "today" (local server midnight to next midnight).

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/daily-schedule"
```

**Response:**
```json
{ "message": "Daily schedule emails sent", "sent": 12, "skipped": 1 }
```
No schedules today → `{ "message": "No schedules for today", "sent": 0 }`. Bad key → 401. Failure → 500 `{ "error": "Failed to send daily schedule emails" }`.

### `POST /api/cron/weekly-driver`

**Why:** Posts the weekly driver summary announcement (Discord embed + related bookkeeping) via `sendWeeklyDriverAnnouncement()`, gated by `canRunWeeklyDriverCron()` so it only actually fires once per week even if the cron ticks more often. A weekly rollup has no single request to hang off of.

**Auth:** This route is the one exception in the table — it does **not** read `x-api-key`/`CRON_API_KEY`. It checks the `Authorization` header for an exact `Bearer $CRON_SECRET` match against `process.env.CRON_SECRET`. Server-to-server only; do not send `CRON_API_KEY` here, it will not match.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "Authorization: Bearer $CRON_SECRET" \
  "$HSR_URL/api/cron/weekly-driver"
```

**Response:**
```json
{ "success": true, "posted": true, "driverId": "clx...", "week": "2026-09-07" }
```
(exact fields come from `sendWeeklyDriverAnnouncement()`). When it isn't the scheduled day/time yet: `{ "success": true, "skipped": true, "reason": "not the scheduled time" }`. Bad/missing bearer token → 401 `{ "error": "Unauthorized" }`. Failure → 500 `{ "error": "Failed to run weekly driver cron" }`.

### `POST /api/cron/live-activity-update`

**Why:** Pushes an APNs Live Activity content-state update (train position, ETA, badge image) to every iOS device with a `LiveActivityToken` on an ACTIVE duty. Live Activities need periodic pushes to refresh their on-screen state; nothing about an ongoing duty otherwise triggers a push on its own, so this has to be polled on an interval (see the Push Notifications section, iOS Live Activity).

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY` (read once into a module constant `CRON_API_KEY`). Server-to-server only.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/live-activity-update"
```

**Response:**
```json
{ "success": true, "total": 8, "succeeded": 7, "failed": 1, "removed": 1 }
```
`removed` counts push tokens deleted after APNs returned 410/400 (device unregistered / bad token). Bad key → 401 `{ "error": "Unauthorized" }`. Failure → 500 `{ "error": "Failed to process Live Activity updates" }`.

### `GET /api/cron/live-activity-update`

**Why:** Identical body to the `POST` handler above — `GET` simply calls `POST(request)` internally. Exists so a cron trigger or health-check that can only issue `GET` requests can still run the same job; the underlying job is the same batch APNs push described above.

**Auth:** Same as `POST` — `x-api-key` header must equal `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X GET \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/live-activity-update"
```

**Response:** Identical shape to `POST` above — `{ "success": true, "total": ..., "succeeded": ..., "failed": ..., "removed": ... }`. Same 401/500 error cases.

### `POST /api/cron/weekly-duty-check`

**Why:** Runs the weekly duty compliance check (`runWeeklyDutyCheck()` — strikes for members who missed their weekly duty obligation, gated by its own admin on/off toggle) and, independently, the weekly instructor teaching-quota check (`runWeeklyInstructorCheck()`, see Instructor Grades). Both are "once a week, look back at the whole week" jobs with no single triggering request.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/weekly-duty-check"
```

**Response:**
```json
{
  "success": true,
  "skipped": false,
  "strikesIssued": 4,
  "instructors": { "warned": 2, "graded": 9 }
}
```
(fields depend on `runWeeklyDutyCheck()`'s own return shape, spread into the response; `instructors` is `null` if that sub-check itself threw, which is caught and logged without failing the whole request). Bad key → 401. Failure of the outer check → 500 `{ "error": "Internal server error" }`.

### `POST /api/cron/training-deadline-check`

**Why:** The full daily sweep for the Training Programmes deadline machinery: removes trainees past their material deadline or their post-closing-exam supervision deadline, sends warning emails ahead of each deadline, clears `TrainingEnrollment`/`TrainingProgress`/`TrainingAccessSession` rows for removed trainees, records a `TrainingTermination` snapshot before every removal, emails a training-movement report to `MANAGE_INSTRUCTORS` admins (only when something actually moved), purges expired `TrainingTermination` rows past their 30-day review window, and purges the exam-side data of removals whose 7-day re-application cooldown has elapsed. None of this can be request-triggered — deadlines are calendar-driven, not action-driven.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/training-deadline-check"
```

**Response:**
```json
{
  "ok": true,
  "deadlineExpired": 1,
  "supervisionExpired": 0,
  "roleRemoved": 2,
  "warningsSent": 3,
  "supervisionWarningsSent": 1,
  "terminationsPurged": 0,
  "reportRecipients": 4,
  "dataPurged": { "terminations": 0, "attempts": 2, "registrations": 1, "admissionJobs": 1 }
}
```
Bad key → 401 `{ "error": "Unauthorized" }`. This route has no try/catch around the body, so an unhandled exception surfaces as Next.js's default 500 response rather than a custom JSON error.

### `POST /api/cron/training-session-cleanup`

**Why:** Closes any `TrainingAccessSession` whose `expiresAt` has passed but was never explicitly ended, stamping `endedAt`/`minutesConsumed` and rolling the consumed minutes into `TrainingProgress.totalMinutesUsed`. A session a member simply closed the tab on (or lost connectivity during) otherwise never gets its minutes accounted for, and the hour-budget logic depends on that being accurate.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/training-session-cleanup"
```

**Response:**
```json
{ "ok": true, "closed": 6 }
```
Bad key → 401 `{ "error": "Unauthorized" }`.

### `POST /api/cron/exam-cleanup`

**Why:** Finds every `IN_PROGRESS` `ExamAttempt` that has exceeded its time limit, gone inactive for 10 minutes, or overrun its question count, and resolves it. An ordinary exam attempt is simply deleted (the trainee can retry with no record); a `CLOSING_EXAM` attempt is instead graded and marked `TIMED_OUT`/failed and run through `finalizeClosingExamAttempt()`, because it is tied to a slot registration and a 3-failure removal rule that erasing the row would silently defeat. Nothing in the exam-taking UI itself can notice "the tab was closed and never came back" — that has to be swept.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/exam-cleanup"
```

**Response:**
```json
{ "success": true, "checked": 14, "timedOut": 2 }
```
Bad key → 401 `{ "error": "Unauthorized" }`. No explicit try/catch — an unhandled error surfaces as Next.js's default 500.

### `POST /api/cron/exam-slot-close`

**Why:** Runs `closeExamSlot()` on every `ACTIVE` `ExamTimeSlot` whose `EXAM_SLOT_WINDOW_MINUTES`-long (60 min) window has elapsed — deciding `attended` per registrant, applying the no-show deadline penalty, sending result emails, granting/removing programme ranks, and archiving decided attempts. Being DB-driven (as opposed to an in-memory timer per slot) is exactly the point: a restart between ticks must not lose track of which slots are due to close (see the Closing Exam Slots section).

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/exam-slot-close"
```

**Response:**
```json
{ "success": true, "checked": 2, "closed": 2 }
```
`closed` only counts slots that were not already closed by another path (e.g., an admin's manual "End" button) between the query and the call. Bad key → 401 `{ "error": "Unauthorized" }`.

### `POST /api/cron/closing-exam-vote`

**Why:** Runs whichever phase of the Daily Closing-Exam Vote the current Budapest time falls into — the 06:00 "do you want to sit today?" invitations, the 12:00 per-programme hour question, the 14:00 tally and slot creation, and (at any hour) opening a slot the vote scheduled once its hour arrives — all in one call via `runClosingExamVoteCron()`. Each phase is idempotent (guarded by its own stamped row/column), so a tick firing twice, late, or for the first time after a restart cannot double-send letters or double-schedule an exam; that idempotency is exactly why one cron line can safely cover every phase.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/closing-exam-vote"
```

**Response:**
```json
{ "success": true, "dayInvitesSent": 5, "hourInvitesSent": 3, "programsDecided": 1, "slotsOpened": 1 }
```
(exact fields come from `runClosingExamVoteCron()`'s summary, spread into the JSON). Bad key → 401 `{ "error": "Unauthorized" }`. Failure → 500 `{ "error": "Failed" }`.

### `POST /api/cron/discord-ticket-cleanup`

**Why:** Auto-closes `DiscordTicket` rows with no activity for 24 hours (generating and storing a transcript, resolving the linked support conversation, and deleting the Discord channel), and sends a one-time "still there?" warning 4 hours ahead of that deadline. Support tickets left open forever are an operational leak with no natural end-of-request to trigger a close.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/discord-ticket-cleanup"
```

**Response:**
```json
{ "closed": 3, "warned": 2 }
```
If nothing was due to close (regardless of warnings): `{ "closed": 0, "warned": 1 }`. Bad key → 401 `{ "error": "Unauthorized" }`. No explicit try/catch around the body — an unhandled error surfaces as Next.js's default 500.

### `POST /api/cron/perf-cleanup`

**Why:** Deletes `GamePerfPlayerSnapshot` and `GamePerfSnapshot` rows older than 24 hours in one transaction. Game performance telemetry is high-volume and only useful briefly; without a periodic prune it grows unbounded (see the Track Mapper section for the related, separately-retained `TrackMapTrace` data, which this route does not touch).

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/perf-cleanup"
```

**Response:**
```json
{ "success": true, "deletedPlayerSnapshots": 15234, "deletedSnapshots": 890 }
```
Bad key → 401 `{ "error": "Unauthorized" }`. No explicit try/catch — an unhandled error surfaces as Next.js's default 500.

### `POST /api/cron/news-sync`

**Why:** Pulls new messages from the configured Discord news channel into `NewsPost` for the homepage feed, and separately syncs the changelog channel into the same table (see the Changelog Popup section) — one more `channelId`, not a second sync pipeline. The changelog sync failing is deliberately non-fatal to the news sync, since an unreadable changelog channel must not stop the homepage news from refreshing.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** Optional `?fetchLimit=` (1-50, default 20) — Discord messages fetched per sync; `?keepRecent=` (50-2000, default 400) — how many `NewsPost` rows are kept after pruning. Both are clamped and default silently if missing/non-numeric.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/news-sync?fetchLimit=20&keepRecent=400"
```

**Response:**
```json
{
  "success": true,
  "fetched": 5,
  "created": 2,
  "updated": 1,
  "pruned": 0,
  "changelog": { "fetched": 3, "created": 1, "updated": 0, "pruned": 0 },
  "changelogError": null
}
```
(`fetched`/`created`/`updated`/`pruned` come from `syncDiscordNewsToDatabase()`'s stats, spread at the top level for the news channel). If the changelog sync itself throws, `changelog` is `null` and `changelogError` carries the message, while the overall response is still 200. Bad key → 401 `{ "error": "Unauthorized" }`. Failure of the main news sync → 500 `{ "error": "Failed to sync news" }`.

### `POST /api/cron/cargo-generate`

**Why:** Generates the day's pool of 20 cargo orders (unique order numbers, wagon counts, destination "Mecsekháza") for the cargo-driver game feature, and expires yesterday's still-`AVAILABLE` orders. This has to run once per calendar day to hand every driver the same shared order pool, rather than being generated per-request.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/cargo-generate"
```

**Response:**
```json
{ "success": true, "generated": 20 }
```
Already generated today (idempotent, checked via a `>= 20` count for today's UTC date) → `{ "skipped": true, "reason": "already generated" }`. Bad key → 401 `{ "error": "Unauthorized" }`. Failure → 500 `{ "error": "Internal server error" }`.

### `POST /api/cron/dispatcher-peak-update`

**Why:** Refreshes the cached dispatcher peak-hour data via `updateDispatcherPeakCache()` — a computed cache over historical dispatch activity that is too expensive to recompute per page view, so it is refreshed on a schedule instead.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/dispatcher-peak-update"
```

**Response:**
```json
{ "success": true }
```
Bad key → 401 `{ "error": "Unauthorized" }`. Failure → 500 `{ "error": "Internal server error" }`.

### `POST /api/cron/schedule-auto-assign-tomorrow`

**Why:** Runs at 23:59 daily to batch auto-assign/approve every still-PENDING schedule request for **tomorrow's** date (both driver and dispatcher, `scope: "BOTH"`), so departures start the next day already assigned rather than waiting on same-day auto-assign to react request-by-request. Gated behind `ScheduleSettings.autoAssignEnabled` — the same toggle the admin panel's auto-assign switch controls — so disabling it there disables this cron too.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters (the target date, tomorrow in Europe/Budapest, is computed internally).

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/schedule-auto-assign-tomorrow"
```

**Response:**
```json
{
  "date": "2026-09-14",
  "actorUserId": "clx...",
  "assigned": 18,
  "skipped": 2
}
```
(the `assigned`/`skipped`-style fields come from `runScheduleAutoAssign()`'s own return shape, spread alongside `date`/`actorUserId`). Auto-assign disabled → `{ "message": "Auto-assign disabled, skipped", "skipped": true }`. No eligible approver found → 500 `{ "error": "No Director General+ user found to act as approver" }`. Bad key → 401 `{ "error": "Unauthorized" }`. Other failure → 500 `{ "error": "Failed to auto-assign tomorrow's schedule" }`.

### `POST /api/cron/monthly-leaderboard`

**Why:** Sends the monthly leaderboard announcement (Discord embed) once the month has actually rolled over, gated by `canRunMonthlyLeaderboardCron()` so repeated ticks within the same month are no-ops. Rolling the monthly leaderboard is a calendar event, not something any single user request causes.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/monthly-leaderboard"
```

**Response:**
```json
{ "success": true, "posted": true, "month": "2026-09" }
```
(fields from `sendMonthlyLeaderboardAnnouncement()`'s return, spread in). Not yet time to run → `{ "success": true, "skipped": true, "reason": "already ran this month" }`. Bad key → 401 `{ "error": "Unauthorized" }`. Failure → 500 `{ "error": "Failed to run monthly leaderboard cron" }`.

### `POST /api/cron/team-reset`

**Why:** Runs the nightly Roblox team reset check (`runTeamResetCheck()`) — moving players back to their default team assignment on a schedule the game itself has no notion of running on its own.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/team-reset"
```

**Response:**
```json
{ "success": true, "reset": 42 }
```
(fields come from `runTeamResetCheck()`'s own return shape). Bad key → 401 `{ "error": "Unauthorized" }`. Failure → 500 `{ "error": "Internal server error" }`.

### `POST /api/cron/discord-member-sync`

**Why:** Mirrors the Discord guild's member list into `DiscordGuildMember` (`syncGuildMembers()`) and writes the day's `CommunityDailyStat` snapshot (`recordDailySnapshot()`), via `runCommunityStatsSync()`. `User` cannot describe someone who is in the guild but never touched the site, so this mirror — and its daily trend row — has to be built from a scheduled Discord API crawl, not from anything the web app's own requests observe (see Community Statistics).

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/discord-member-sync"
```

**Response:**
```json
{ "success": true, "sync": { "added": 3, "updated": 210, "left": 1 }, "snapshot": { "date": "2026-09-13" } }
```
(the `sync`/`snapshot` shape comes from `runCommunityStatsSync()`'s own return value). If the Discord guild member sync itself failed, no daily snapshot is written and the response is 502: `{ "error": "Discord guild member sync failed — snapshot skipped" }` — this is a deliberate gap-not-cliff design so a bad day never reads as a mass exodus. Bad key → 401 `{ "error": "Unauthorized" }`. Other failure → 500 `{ "error": "Internal server error" }`.

### `POST /api/cron/admission-queue`

**Why:** Drains the `AdmissionJob` queue (`processAdmissionJobs()`) — sending result emails, pulling accepted applicants into the Discord guild, granting ranks — in up to 10 bounded passes per invocation, stopping early once a pass processes zero jobs. This is the safety net for the admission pipeline described in the Admission Queue section: the request that queues jobs also kicks the worker in the background, so this route mainly exists to pick the queue back up after a restart or to run a job whose retry backoff had not yet expired.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/admission-queue"
```

**Response:**
```json
{ "processed": 7 }
```
This route has no try/catch — an unhandled error surfaces as Next.js's default 500. Bad key → 401 `{ "error": "Unauthorized" }`.

### `POST /api/cron/shop-apple-pass`

**Why:** Retries Apple Wallet membership pass deliveries left unfinished at payment time (`retryPendingApplePasses()`) — a restart mid-flight, a momentary SMTP failure, an avatar host that was briefly down. Pass generation shells out to `openssl` and fetches an avatar over the network, so it deliberately runs outside the payment transaction (see the Apple Wallet Membership Pass section) and needs this cron to finish what didn't complete inline.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/shop-apple-pass"
```

**Response:**
```json
{ "ok": true, "delivered": 2, "failed": 0 }
```
Bad key → 401 `{ "error": "Unauthorized" }`. Failure → 500 `{ "error": "Internal server error" }`.

### `POST /api/cron/wrapped`

**Why:** Two jobs behind one route, chosen by the `queue` param: with it, `queueWrappedForYear()` writes a `WrappedDelivery` row per eligible member for the year (the once-a-year crontab line, run 31 December); without it, `processWrappedQueue()` just drains whatever is `PENDING` — which is what finishes a run interrupted mid-flight and retries a card whose send failed (see the HSR Wrapped section). Both being one route keeps the "queue" and "drain" halves of the same feature next to each other.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** `?queue=1` — also queues the year's rows before draining (omit to only drain). `?year=` — the year to queue, defaults to the current UTC year (only meaningful with `queue=1`). `?limit=` — how many outstanding rows to drain this call, defaults to 40.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/wrapped?queue=1&year=2026&limit=40"
```

**Response:**
```json
{ "ok": true, "year": 2026, "queued": 512, "total": 512, "sent": 40, "failed": 0, "remaining": 472 }
```
(the `queued`/`total` fields come from `queueWrappedForYear()` and are `{ "queued": 0, "total": 0 }` when `queue` was omitted; `sent`/`failed`/`remaining`-style fields come from `processWrappedQueue()`). Bad key → 401 `{ "error": "Unauthorized" }`. Failure → 500 `{ "error": "Internal server error" }`.

### `POST /api/cron/shop-printful-retry`

**Why:** Retries shop orders whose Printful print-on-demand submission previously failed (`retryFailedPrintfulOrders()`) — a third-party fulfillment API call that can fail transiently and has no user-facing action to hang a retry off of.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/shop-printful-retry"
```

**Response:**
```json
{ "ok": true, "retried": 1, "failed": 0 }
```
Bad key → 401 `{ "error": "Unauthorized" }`. Failure → 500 `{ "error": "Internal server error" }`.

### `POST /api/cron/shop-expire-orders`

**Why:** Expires shop orders left `PENDING` past their payment window (`expirePendingShopOrders()`) — an order nobody paid for and nobody explicitly cancelled has no other trigger to close it out.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/shop-expire-orders"
```

**Response:**
```json
{ "ok": true, "expired": 3 }
```
Bad key → 401 `{ "error": "Unauthorized" }`. Failure → 500 `{ "error": "Internal server error" }`.

### `POST /api/cron/radio-cleanup`

**Why:** Deletes in-game radio (Telephone tool) `RadioMessage` rows older than 30 days (`RETENTION_DAYS`). In-game radio chatter has no legitimate long-term retention need and would otherwise grow unbounded.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/radio-cleanup"
```

**Response:**
```json
{ "deleted": 128 }
```
Bad key → 401 `{ "error": "Unauthorized" }`. No explicit try/catch — an unhandled error surfaces as Next.js's default 500.

### `POST /api/cron/duty-day-report`

**Why:** Two steps in one route: `queueDutyDayReports(date)` writes a `DutyDayReport` row per member who completed a duty on the given operating day (default: yesterday, UTC), then `processDutyDayQueue()` sends/drains whatever is outstanding — including anything a previous run queued but could not send. Running it hourly is safe because queueing is idempotent (one row per member per day) and a sent row is never re-sent (see the Daily Duty Card section).

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** `?date=YYYY-MM-DD` — re-queues a specific operating day instead of "yesterday" (for a run that was missed). An invalid or missing value falls back to yesterday (UTC midnight, matching how `Duty.date` is written).

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/duty-day-report?date=2026-09-12"
```

**Response:**
```json
{
  "success": true,
  "date": "2026-09-12",
  "queued": 9,
  "sent": 9,
  "failed": 0,
  "remaining": 0
}
```
(the `queued`-style field comes from `queueDutyDayReports()` and the `sent`/`failed`/`remaining`-style fields come from `processDutyDayQueue()`, both spread into the response alongside `date`). Bad key → 401 `{ "error": "Unauthorized" }`. Failure → 500 `{ "error": "Failed to run duty day reports" }`.

### `POST /api/cron/document-reminders`

**Why:** Re-resolves the required signers of every PENDING signable document right now (not at submit time) and writes to whichever of them have not signed, via `remindPendingSigners()` — picking up a signer whose Discord role cache was stale at submit time, or someone promoted into the signer set afterwards (see the Document Signature Reminders section). This has to be a scheduled re-check because role membership drifts continuously and nothing about a pending document re-evaluates its own audience.

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/document-reminders"
```

**Response:**
```json
{ "documentsChecked": 4, "remindersSent": 3, "signersDue": 3 }
```
(exact fields come from `remindPendingSigners()`'s own return shape). Bad key → 401 `{ "error": "Unauthorized" }`. Failure → 500 `{ "error": "Failed to send document reminders" }`.

### `POST /api/cron/email-queue`

**Why:** Drains the `EmailOutbox` (`drainEmailOutbox()`) and reports its current stats (`emailOutboxStats()`). The app already kicks a drain the moment it queues a letter, so in the ordinary case this finds nothing to do; it exists for letters still in line when the process stopped, a retry whose backoff expired long after the request that queued it returned, and a letter left `SENDING` by a container killed mid-send (see The Email Outbox section).

**Auth:** `x-api-key` header compared to `process.env.CRON_API_KEY`.

**Request:** No query parameters.

```bash
curl -sS -X POST \
  -H "x-api-key: $CRON_API_KEY" \
  "$HSR_URL/api/cron/email-queue"
```

**Response:**
```json
{
  "sent": 2,
  "failed": 0,
  "retried": 1,
  "stats": { "pending": 0, "sending": 0, "failed": 1, "sent": 4820 }
}
```
(the `sent`/`failed`/`retried`-style fields come from `drainEmailOutbox()`, and `stats` from `emailOutboxStats()` — both spread/nested into the response). Bad key → 401 `{ "error": "Unauthorized" }`. Failure → 500 `{ "error": "Failed to drain the email queue" }`.

## Cargo, Changelog, Closing-Exam Vote, Dispatch Orders, Documents, Duty

### `GET /api/changelog`

**Why:** Feeds the site-wide "what's new" popup with the changelog entries a member has not seen yet, mirrored from the Discord changelog channel.

**Auth:** Session cookie or bearer (`getSessionWithBearer()`). A signed-out visitor is answered `{ entries: [], total: 0, seenUpTo: null }` with status 200 rather than 401 — the popup is mounted on public pages too.

**Request:** No parameters.

```bash
curl -sS "$HSR_URL/api/changelog" -b cookies.txt
```

**Response:**

```json
{
  "entries": [
    { "id": "abc123", "title": "New feature", "content": "...", "publishedAt": "2026-09-10T12:00:00.000Z" }
  ],
  "total": 1,
  "seenUpTo": "2026-09-05T00:00:00.000Z"
}
```
On any internal failure it still returns 200 with the empty shape.

---

### `POST /api/changelog`

**Why:** Dismisses the popup, marking everything up to the newest entry the client actually rendered as seen (never marks an entry posted while the dialog was open).

**Auth:** Session cookie or bearer. 401 for a signed-out caller.

**Request:** JSON body: `seenUpTo` (string, required — an ISO date parseable by `new Date()`).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"seenUpTo":"2026-09-10T12:00:00.000Z"}' \
  "$HSR_URL/api/changelog"
```

**Response:**

```json
{ "success": true }
```
Errors: 400 `{"error":"Invalid body"}` (unparsable JSON) or `{"error":"seenUpTo must be a date"}` (missing/invalid date); 500 `{"error":"Failed to save"}`.

---

### `GET /api/cargo/orders`

**Why:** Lists today's still-available cargo shipments for a member qualified to haul freight, so they can pick one before starting a `CARGO_DRIVER` duty.

**Auth:** Session cookie or bearer, plus `permissions.isCargoGroup()` (the cargo-qualified Discord role group). 403 otherwise.

**Request:** No parameters — always today's date (UTC midnight).

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/cargo/orders"
```

**Response:**

```json
[
  {
    "id": "clx1",
    "date": "2026-09-13T00:00:00.000Z",
    "orderNumber": 12,
    "status": "AVAILABLE",
    "wagonCount": 8,
    "totalWeightT": 640,
    "takenByUserId": null
  }
]
```
500 `{"error":"Internal server error"}` on failure.

---

### `GET /api/closing-exam-vote/[token]`

**Why:** Renders the public voting page a candidate reaches from the 06:00 "do you want to sit today?" letter or the 12:00 hour-pick letter, without requiring login (the letter is opened on a phone that may not be signed in).

**Auth:** Public — the `token` in the URL is the only authority; it resolves to exactly one `ClosingExamDayVote` or `ClosingExamHourVote` row.

**Request:** Path param `token` (string). No body.

```bash
curl -sS "$HSR_URL/api/closing-exam-vote/$TOKEN"
```

**Response (day vote):**

```json
{
  "kind": "DAY",
  "open": true,
  "date": "2026-09-13",
  "response": null,
  "programNames": ["Train Driver"],
  "deadlineHour": 12
}
```
**Response (hour vote):**

```json
{
  "kind": "HOUR",
  "open": true,
  "date": "2026-09-13",
  "hour": null,
  "programName": "Train Driver",
  "hours": [14, 15, 16, 17, 18, 19, 20, 21, 22],
  "deadlineHour": 14
}
```
404 `{"error":"Not found"}` for an unknown token.

---

### `POST /api/closing-exam-vote/[token]`

**Why:** Casts the vote — Yes/No for the day question, or the chosen hour for the hour question. Re-checks the deadline server-side so a late submission cannot change a tally the exam was already scheduled on.

**Auth:** Public, token-authenticated via the URL (same as GET). The letter's buttons only pre-select `?answer=`; this call is what actually commits it.

**Request:** JSON body — for a DAY-kind token: `response` (boolean, required); for an HOUR-kind token: `hour` (number, required, must be one of `EXAM_HOURS`, i.e. one of 14–22).

```bash
curl -sS -X POST -H "Content-Type: application/json" \
  -d '{"response": true}' \
  "$HSR_URL/api/closing-exam-vote/$TOKEN"
```

**Response:**

```json
{ "ok": true, "response": true }
```
or `{ "ok": true, "hour": 18 }`. Errors: 404 `{"error":"Not found"}`; 409 `{"error":"Voting closed","code":"closed"}` once the deadline passed; 400 `{"error":"response must be a boolean"}` / `{"error":"hour out of range"}`.

---

### `POST /api/dispatch-orders`

**Why:** Lets a dispatcher on active duty draft a written dispatch order ("írásbeli rendelkezés") addressed to a specific driver who is also on active duty.

**Auth:** Session cookie or bearer, plus `permissions.canUseDispatchOrders()`, plus an active `TRAIN_DISPATCHER` duty (`getActiveDispatcherDuty`). 403 without either.

**Request:** JSON body: `content` (string, required, trimmed non-empty), `recipientId` (string, required — must be a user currently on an active `TRAIN_DRIVER` duty).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content":"Kérem várjon a 3-as vágányon.","recipientId":"user_123"}' \
  "$HSR_URL/api/dispatch-orders"
```

**Response:**

```json
{
  "id": "ord_1",
  "status": "DRAFT",
  "content": "Kérem várjon a 3-as vágányon.",
  "issuerId": "user_dispatcher",
  "recipientId": "user_123",
  "createdAt": "2026-09-13T10:00:00.000Z"
}
```
Errors: 403 `{"error":"Nincs jogosultság az írásbeli rendelkezésekhez"}`, 403 `{"error":"Nincs aktív forgalmi szolgálat"}`, 400 `{"error":"A rendelkezés tartalma kötelező"}` / `{"error":"Címzett megadása kötelező"}`, 400 `{"error":"A címzett nincs aktív mozdonyvezetői szolgálatban"}`.

---

### `GET /api/dispatch-orders`

**Why:** Lists every dispatch order the caller is party to — everything they issued, plus everything addressed to them once it has left DRAFT.

**Auth:** Session cookie or bearer.

**Request:** No parameters.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/dispatch-orders"
```

**Response:**

```json
[
  {
    "id": "ord_1",
    "status": "FINALIZED",
    "content": "Kérem várjon a 3-as vágányon.",
    "issuerId": "user_dispatcher",
    "recipientId": "user_123",
    "revision": 0,
    "createdAt": "2026-09-13T10:00:00.000Z"
  }
]
```

---

### `GET /api/documents`

**Why:** A member's own drafted/submitted signed documents (signers use the admin surface for everyone else's).

**Auth:** Session cookie or bearer, plus `ACCESS_DOCUMENTS` (`canAccessDocuments()`, via `requireDocumentsSurface()`). 403 without it.

**Request:** No parameters.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/documents"
```

**Response:**

```json
[
  {
    "id": "doc_1",
    "title": "Kérelem",
    "status": "DRAFT",
    "verifyCode": "AB12CD",
    "requiredSignatures": 1,
    "updatedAt": "2026-09-13T09:00:00.000Z"
  }
]
```

---

### `POST /api/documents`

**Why:** Creates a new draft document (the base of the "sign documents" flow), with a stable verify code assigned up front so it survives a reject/resubmit cycle.

**Auth:** Session cookie or bearer, plus `ACCESS_DOCUMENTS`.

**Request:** JSON body: `title` (string, required, trimmed non-empty, ≤ `MAX_TITLE_LENGTH`), `body` (string, optional, ≤ `MAX_BODY_LENGTH`), `requiredSignatures` (number, optional — omitted means 1, must parse within 1..`MAX_REQUIRED_SIGNATURES` via `parseRequiredSignatures`).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"Kérelem","body":"...","requiredSignatures":2}' \
  "$HSR_URL/api/documents"
```

**Response:**

```json
{
  "id": "doc_1",
  "title": "Kérelem",
  "status": "DRAFT",
  "verifyCode": "AB12CD",
  "requiredSignatures": 2
}
```
Errors: 400 `{"error":"A cím megadása kötelező"}`, `{"error":"A cím legfeljebb N karakter lehet"}`, `{"error":"A dokumentum törzse túl hosszú"}`, `{"error":"Az aláírók száma 1 és N között lehet"}`.

---

### `GET /api/duty/booking`

**Why:** The caller's own DRIVER schedule bookings (`ScheduleRequest` rows) for today and onward, shaped for the duty page's booking board.

**Auth:** Session cookie or bearer.

**Request:** Query param `date` (optional, `YYYY-MM-DD`; defaults to today UTC) — returned bookings are `date >= filterDate`.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/duty/booking?date=2026-09-14"
```

**Response:**

```json
[
  {
    "id": "req_1",
    "pairGroup": "IC101",
    "fordaGroup": "F1",
    "operator": "MÁV",
    "date": "2026-09-14T00:00:00.000Z",
    "status": "PENDING",
    "trackNumber": null,
    "responseNote": null
  }
]
```

---

### `POST /api/duty/booking`

**Why:** Lets a Train Driver reserve a train pair for a future day (or, within an hour of departure, for today) — the advance-booking half of the duty flow.

**Auth:** Session cookie or bearer, plus `permissions.canDrive()`. 403 `{"error":"Only Train Drivers can book train pairs"}` otherwise.

**Request:** JSON body: `pairGroup` (string, required), `date` (string, required, parseable date). Booking week is Monday–Sunday (`getBookingWindow`) unless the train departs within 60 minutes today ("last-minute booking").

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"pairGroup":"IC101","date":"2026-09-14"}' \
  "$HSR_URL/api/duty/booking"
```

**Response (201):**

```json
{
  "id": "req_1",
  "pairGroup": "IC101",
  "fordaGroup": "F1",
  "operator": "MÁV",
  "date": "2026-09-14T00:00:00.000Z",
  "status": "PENDING",
  "trackNumber": null,
  "responseNote": null
}
```
Errors: 400 `{"error":"pairGroup and date are required"}`, `{"error":"Invalid date"}`, `{"error":"Driver requests can only be made for the current booking week (Monday to Sunday)"}`, `{"error":"No outbound train found for this pair group"}`, `{"error":"You already have a booking for this train pair on this date"}`, `{"error":"This train pair is already booked for this date"}`; **booking time conflict** — 400 with a localized `error` message from `conflictMessage()` (built from `DRIVER_CONFLICT_MINUTES` = 120) plus a `conflicts` array naming the other train and its departure.

---

### `POST /api/duty/dispatcher-booking`

**Why:** Lets a dispatcher reserve a future hour-long dispatcher slot in advance (max 5 dispatchers per hour).

**Auth:** Session cookie or bearer, plus `permissions.canDispatch()`. 403 otherwise.

**Request:** JSON body: `date` (string `YYYY-MM-DD`, required, must be strictly after today per `getTodayStoredBudapestDate()`), `hour` (number 0–23, required).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"date":"2026-09-14","hour":18}' \
  "$HSR_URL/api/duty/dispatcher-booking"
```

**Response (201):**

```json
{
  "id": "book_1",
  "userId": "user_1",
  "date": "2026-09-14T00:00:00.000Z",
  "hour": 18,
  "status": "PENDING",
  "scheduleRequestId": "req_2"
}
```
Errors: 400 `{"error":"date and hour (0–23) are required"}`, `{"error":"Invalid date"}`, `{"error":"Bookings are only available for future dates. For today, start duty directly."}`; 409 `{"error":"This slot is full (max 5 dispatchers)"}`, `{"error":"You already booked this slot"}`.

---

### `DELETE /api/duty/dispatcher-booking`

**Why:** Cancels the caller's own dispatcher slot booking (and its underlying `ScheduleRequest`), as long as it has not already been approved.

**Auth:** Session cookie or bearer. The booking must belong to the caller.

**Request:** Query param `id` (string, required — the booking id).

```bash
curl -sS -X DELETE -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/duty/dispatcher-booking?id=book_1"
```

**Response:**

```json
{ "success": true }
```
Errors: 400 `{"error":"id is required"}`; 404 `{"error":"Booking not found"}`; 403 `{"error":"Not your booking"}`, `{"error":"Cannot cancel an approved booking"}`.

---

### `GET /api/duty/available-trains`

**Why:** Populates the driver's Start-duty train picker: which trains are free, taken, past their departure grace, or reserved — including the caller's own booking-time conflicts, so the board explains itself before the click.

**Auth:** Session cookie or bearer.

**Request:** Query param `date` (optional `YYYY-MM-DD`; defaults to today UTC).

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/duty/available-trains?date=2026-09-13"
```

**Response:**

```json
{
  "outboundTrains": [
    {
      "id": "train_1",
      "trainNumber": "9601",
      "trainType": "IC",
      "departureTime": "08:34",
      "isAvailable": false,
      "isReservedForMe": true,
      "unavailableReason": null,
      "timeConflict": { "trainNumber": "9701", "departureTime": "09:10", "minutesApart": 36 },
      "pairGroup": "IC101",
      "direction": 0
    }
  ],
  "returnTrains": [],
  "myApprovedRequests": [{ "id": "req_1", "pairGroup": "IC101", "status": "APPROVED" }],
  "currentTime": "08:20"
}
```
`unavailableReason` values include `"Assigned to <name>"`, `"Departed"` (only once past `DEPARTURE_GRACE_MINUTES`, i.e. `isPastDepartureGrace`), `"Reserved by <name>"`. 500 `{"error":"Failed to get available trains"}` on failure.

---

### `POST /api/duty/start`

**Why:** Opens a duty of any type (`TRAIN_DRIVER`, `TRAIN_DISPATCHER`, `CARGO_DRIVER`, `TICKET_INSPECTOR`) — the single entry point that enforces every rank, suspension, trainee and reservation rule before real service begins.

**Auth:** Session cookie or bearer, plus `dbUser.isActiveInGame` (or `role === "ADMIN"`). Per-type permission: `CAN_DRIVE` (`canDrive()`) or trainee driver, `CAN_DISPATCH` (`canDispatch()`), `isCargoGroup()`, or `CAN_INSPECT` (`canInspect()`). Checks `permissions.suspensionFor(scope)` for the duty's `SuspensionScope` (via `scopeForDutyType`) before the permission checks, so a trainee holding no flag is still blocked.

**Request:** JSON body: `dutyType` (required, one of the four types), plus type-specific fields: `startLocationId` (required for TRAIN_DRIVER), `trainIds` (array, optional), `trackNumber` (string, optional), `scheduleRequestId`/`scheduleRequestIds` (optional), `watchingStationIds` (array, required for TRAIN_DISPATCHER), `dispatcherPosition` (optional, one of `főrendelkező|rendelkező|külsős|gyakornok`), `cargoOrderId` (required for CARGO_DRIVER), `inspectorBookingId`/`inspectorPairGroup` (optional for TICKET_INSPECTOR), `notes` (optional).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"dutyType":"TRAIN_DRIVER","startLocationId":"loc_1","trainIds":["train_1","train_2"]}' \
  "$HSR_URL/api/duty/start"
```

**Response (201):** the created `Duty` with `user`, `startLocation`, `assignedTrains` (with train + stops + station), `cargoOrder`.

Notable errors: 403 suspension — `{"error":"You are suspended from <scope> service.","suspension":{"scope":"DRIVER","reason":"...","expiresAt":"2026-10-01T00:00:00.000Z"}}`; 403 `{"error":"Trainee Drivers can only start duty after passing the closing exam."}`; 403 `{"error":"Trainee Drivers can only start duty by providing a track number, selecting trains is not allowed yet."}`; 403 `{"error":"Trainee Drivers cannot start duty against a schedule reservation."}`; 403 `{"error":"Trainee Ticket Inspectors cannot start duty against a specific service until their required supervisions are complete."}`; 400 `{"error":"You already have an active duty. Please end it first."}`; 400 departed-service — `{"error":"This service has already departed, the reservation can no longer be resumed."}`; 409 `{"error":"This inspector booking is no longer active."}`; 409 `{"error":"This shipment has already been taken. Please choose another."}`; 409 `{"error":"This schedule request is already linked to another duty."}`.

---

### `GET /api/duty/inspector-trains`

**Why:** The ticket inspector's board of pairs their rank may work on a given day, with the driver holding each pair and who already booked it.

**Auth:** Session cookie or bearer, plus `CAN_INSPECT` (`canInspect()`). 403 `{"error":"Only ticket inspectors can view inspector services"}` otherwise.

**Request:** Query param `date` (optional `YYYY-MM-DD`; defaults to today UTC).

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/duty/inspector-trains?date=2026-09-14"
```

**Response:**

```json
{
  "date": "2026-09-14T00:00:00.000Z",
  "pairs": [
    { "pairGroup": "IC101", "driver": "user_1 name", "bookedBy": null, "isAllowed": true }
  ]
}
```
(shape from `buildInspectorBoard()`, spread after `date`.) 500 `{"error":"Failed to load inspector services"}` on failure.

---

### `GET /api/duty/dispatchers`

**Why:** Lists every dispatcher currently on active duty — used by drivers/dispatchers to see who else is working the panel right now.

**Auth:** Session cookie or bearer.

**Request:** No parameters.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/duty/dispatchers"
```

**Response:**

```json
[
  { "userId": "user_1", "displayName": "Kovács János", "position": "főrendelkező" }
]
```

---

### `POST /api/duty/end`

**Why:** Ends the caller's active duty (any type), releasing a claimed cargo order back to the pool, closing any Live Activity, and updating cumulative user stats.

**Auth:** Session cookie or bearer (`getSessionWithBearer()` directly, not `requireAuth()`).

**Request:** JSON body (optional/empty allowed): `dutyId` (string, optional — defaults to the caller's current active duty if omitted), `notes` (string, optional).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}' \
  "$HSR_URL/api/duty/end"
```

**Response:** the updated `Duty` (`status: "COMPLETED"`, `endTime` set) with `user`, `startLocation`, `assignedTrains`, `cargoOrder`.

Errors: 401 `{"error":"Unauthorized"}`; 404 `{"error":"No active duty found"}`; 500 `{"error":"Failed to end duty"}`.

---

### `GET /api/duty/inspector-booking`

**Why:** The caller's own advance ticket-inspector bookings from a given date onward.

**Auth:** Session cookie or bearer.

**Request:** Query param `date` (optional `YYYY-MM-DD`; defaults to today UTC).

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/duty/inspector-booking?date=2026-09-14"
```

**Response:**

```json
[
  { "id": "ib_1", "pairGroup": "IC101", "date": "2026-09-14T00:00:00.000Z", "status": "BOOKED", "dutyId": null }
]
```

---

### `POST /api/duty/inspector-booking`

**Why:** Books a train pair in advance for a ticket inspector to ride — no dispatcher approval needed, unlike a driver's booking, because the inspector rides a service the driver already holds.

**Auth:** Session cookie or bearer, plus `CAN_INSPECT` (`canInspect()`) — which is already false for a suspended inspector. Also refused for a trainee inspector still under the programme's supervision requirement (`permissions.isServiceTrainee("INSPECTOR")`). Requires `InspectorSettings.enabled`.

**Request:** JSON body: `pairGroup` (string, required), `date` (string, required — must fall within the current Monday–Sunday booking week, and strictly after today Budapest time; today itself is refused since inspectors start duty directly instead).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"pairGroup":"IC101","date":"2026-09-14"}' \
  "$HSR_URL/api/duty/inspector-booking"
```

**Response (201):**

```json
{ "id": "ib_1", "pairGroup": "IC101", "date": "2026-09-14T00:00:00.000Z", "status": "BOOKED", "dutyId": null }
```
Errors: 403 `{"error":"Only ticket inspectors can book inspector services"}`, `{"error":"Trainee Ticket Inspectors cannot book a service until their required supervisions are complete."}`, `{"error":"Inspector bookings are currently closed"}`, `{"error":"Your inspector rank is not qualified for this service"}`; 400 `{"error":"pairGroup and date are required"}`, `{"error":"Invalid date"}`, `{"error":"Inspector bookings can only be made for the current booking week (Monday to Sunday)"}`, `{"error":"No active train found for this pair group"}`, `{"error":"Inspector services are booked in advance — for today, just start your duty"}`, `{"error":"You already booked this service for this date"}`; 409 `{"error":"This service already has an inspector"}`.

---

### `DELETE /api/duty/inspector-booking`

**Why:** Cancels the caller's own inspector booking, as long as it has not already turned into a duty.

**Auth:** Session cookie or bearer. Booking must belong to the caller.

**Request:** Query param `id` (string, required).

```bash
curl -sS -X DELETE -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/duty/inspector-booking?id=ib_1"
```

**Response:**

```json
{ "ok": true }
```
Errors: 400 `{"error":"id is required"}`; 404 `{"error":"Booking not found"}`; 400 `{"error":"This booking is already in duty and cannot be cancelled"}`. Cancelling an already-`CANCELLED` booking still returns `{ "ok": true }`.

---

### `GET /api/duty/dispatcher-slots`

**Why:** The 24-hour peak-hour dispatcher booking board — score/peak status per hour, who has booked each one, and whether the caller has a booking in it.

**Auth:** Session cookie or bearer.

**Request:** Query param `date` (optional `YYYY-MM-DD`; defaults to today, `getTodayStoredBudapestDate()`).

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/duty/dispatcher-slots?date=2026-09-14"
```

**Response:**

```json
[
  {
    "hour": 18,
    "score": 42,
    "isPeak": true,
    "peakGroup": "evening",
    "isPast": false,
    "isFull": false,
    "bookings": [{ "userId": "user_1", "displayName": "Kovács János", "image": null, "status": "PENDING" }],
    "myBooking": { "id": "book_1", "status": "PENDING" }
  }
]
```

---

### `GET /api/duty/my`

**Why:** The full duty dashboard payload for the caller: their active duty (with a live consist match for `TRAIN_DRIVER`), today's completed duties, and this month's aggregate stats.

**Auth:** Session cookie or bearer (`getSessionWithBearer()` directly).

**Request:** No parameters.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/duty/my"
```

**Response:**

```json
{
  "activeDuty": { "id": "duty_1", "dutyType": "TRAIN_DRIVER", "consistMatch": { "matches": true } },
  "todayDuties": [{ "id": "duty_0", "status": "COMPLETED" }],
  "monthlyStats": { "totalDuties": 12, "totalMinutes": 720, "totalHours": 12 }
}
```
401 `{"error":"Unauthorized"}`; 500 `{"error":"Failed to fetch user duties"}`.

---

### `GET /api/duty/active`

**Why:** Every currently active duty on the site (all drivers/dispatchers, or filtered to one type) — used for "who's on now" views.

**Auth:** Session cookie or bearer, via full `requireAuth()` (confirms the account still exists, not just that the JWT verifies).

**Request:** Query param `type` (optional, one of `TRAIN_DRIVER`/`TRAIN_DISPATCHER`).

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/duty/active?type=TRAIN_DRIVER"
```

**Response:**

```json
[
  {
    "id": "duty_1",
    "dutyType": "TRAIN_DRIVER",
    "user": { "id": "user_1", "displayName": "Kovács János" },
    "assignedTrains": [{ "train": { "trainNumber": "9601" } }],
    "consistMatch": { "matches": true }
  }
]
```
500 `{"error":"Failed to fetch active duties"}`.

---

### `POST /api/duty/next-train`

**Why:** Switches an already-active `TRAIN_DRIVER` duty onto its next approved scheduled journey without ending the duty — asks the same rank/suspension/trainee questions `/api/duty/start` does, since it hands the duty real service just the same.

**Auth:** Session cookie or bearer, plus `CAN_DRIVE` (`canDrive()`); checks `permissions.suspensionFor("DRIVER")` and refuses a service trainee (`isServiceTrainee("DRIVER")`).

**Request:** JSON body: `scheduleRequestId` (string) or `scheduleRequestIds` (array of strings) — at least one required, must be APPROVED and belong to the caller.

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"scheduleRequestIds":["req_2"]}' \
  "$HSR_URL/api/duty/next-train"
```

**Response:** the updated `Duty` with `assignedTrains` reset to the new pair's trains, `user`, `startLocation`, `cargoOrder`.

Errors: 403 suspension — `{"error":"You are suspended from Train Driver service.","suspension":{...}}`; 403 `{"error":"Trainee Drivers cannot take a scheduled journey."}`; 403 `{"error":"You don't have permission to drive a scheduled journey."}`; 400 `{"error":"scheduleRequestId(s) required"}`, `{"error":"No active Train Driver duty"}`, `{"error":"Invalid or unapproved schedule request"}`; 404 `{"error":"One or more schedule requests not found"}`.

---

### `GET /api/dispatch-orders/active-drivers`

**Why:** Lets a dispatcher on active duty see every driver currently on active duty, to pick a recipient for a new dispatch order.

**Auth:** Session cookie or bearer, plus an active dispatcher duty (`getActiveDispatcherDuty`). 403 `{"error":"Nincs aktív forgalmi szolgálat"}` otherwise.

**Request:** No parameters.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/dispatch-orders/active-drivers"
```

**Response:**

```json
[{ "userId": "user_123", "displayName": "Kovács János" }]
```

---

### `GET /api/dispatch-orders/stream`

**Why:** Live push of dispatch-order changes to the caller (new/updated orders addressed to or from them), so the dispatch order UI updates without polling.

**Auth:** Session cookie or bearer.

**Request:** None. SSE stream, keep the connection open.

```bash
curl -sS -N -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/dispatch-orders/stream"
```

**Response:** Server-Sent Events built with `createSSEResponse()`. On an update, a plain data frame is sent:
```
data: update

```
The client re-fetches `/api/dispatch-orders` (or the single order) on receipt; there is no payload beyond the event itself. Periodic `:` heartbeat comments keep the connection alive through proxies.

---

### `GET /api/dispatch-orders/[id]`

**Why:** Fetches one dispatch order — for the issuer at any status, or for the recipient once it has left DRAFT (a draft is not yet visible to its addressee).

**Auth:** Session cookie or bearer. Must be the issuer, or the recipient of a non-DRAFT order.

**Request:** Path param `id`.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/dispatch-orders/ord_1"
```

**Response:** the serialized order (same shape as the list endpoint). 404 `{"error":"Nem található"}`; 403 `{"error":"Nincs jogosultság"}`.

---

### `PATCH /api/dispatch-orders/[id]`

**Why:** Lets the issuer revise a `FINALIZED` order's content while it is still inside its edit window — resets both signatures and restarts the sign flow at `AWAITING_MV`, notifying the recipient they must sign again.

**Auth:** Session cookie or bearer, must be the issuer, order must be `FINALIZED` and `isEditable(existing)` (inside `EDIT_WINDOW_MS`).

**Request:** JSON body: `content` (string, required, trimmed non-empty).

```bash
curl -sS -X PATCH -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content":"Módosított szöveg"}' \
  "$HSR_URL/api/dispatch-orders/ord_1"
```

**Response:** the updated, re-serialized order (`status: "AWAITING_MV"`, `revision` incremented, sign timestamps cleared). Errors: 404 `{"error":"Nem található"}`; 403 `{"error":"Nincs jogosultság"}`; 409 `{"error":"A szerkesztési idő lejárt"}`; 400 `{"error":"A rendelkezés tartalma kötelező"}`. Also pushes a notification to the recipient.

---

### `GET /api/documents/[id]`

**Why:** Fetches one signed document — the author sees their own, a signer (`SIGN_DOCUMENTS`) sees any.

**Auth:** Session cookie or bearer, plus `ACCESS_DOCUMENTS` for the surface, and either authorship or `SIGN_DOCUMENTS`. A document the caller may not see returns 404, not 403 (so document ids cannot be probed).

**Request:** Path param `id`.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/documents/doc_1"
```

**Response:** the serialized document, including attachments and signatures. 404 `{"error":"A dokumentum nem található"}` if not found or not accessible.

---

### `PATCH /api/documents/[id]`

**Why:** Lets the author edit a document while it is still a DRAFT or a REJECTED one — anything already submitted or signed is frozen, since an editable signature would prove nothing.

**Auth:** Session cookie or bearer, plus `ACCESS_DOCUMENTS`, and must be the author.

**Request:** JSON body (all optional, at least one required): `title` (string, non-empty, ≤ `MAX_TITLE_LENGTH`), `body` (string, ≤ `MAX_BODY_LENGTH`), `requiredSignatures` (number, 1..`MAX_REQUIRED_SIGNATURES`).

```bash
curl -sS -X PATCH -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"Új cím"}' \
  "$HSR_URL/api/documents/doc_1"
```

**Response:** the updated serialized document. Errors: 403 `{"error":"Csak a készítő szerkesztheti"}`; 409 `{"error":"Csak piszkozat vagy elutasított dokumentum szerkeszthető"}`; 400 `{"error":"A cím nem lehet üres"}` / length errors / `{"error":"Nincs módosítandó mező"}`.

---

### `DELETE /api/documents/[id]`

**Why:** Deletes a draft the author no longer wants — never a submitted or signed one, which belongs to the review queue or is a permanent record.

**Auth:** Session cookie or bearer, plus `ACCESS_DOCUMENTS`, and must be the author.

**Request:** Path param `id`. No body.

```bash
curl -sS -X DELETE -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/documents/doc_1"
```

**Response:**

```json
{ "ok": true }
```
Errors: 403 `{"error":"Csak a készítő törölheti"}`; 409 `{"error":"Csak piszkozat törölhető"}`. Deletes stored attachment objects from R2 first.

---

### `POST /api/duty-appeal/[strikeId]`

**Why:** Lets a driver appeal a weekly-duty strike using the token from their strike notification, before the appeal deadline. Notifies all admins by email on submission.

**Auth:** Session cookie or bearer, plus a matching `token` supplied **in the JSON body** (not the URL) that must equal the strike's stored `appealToken`; the strike must belong to the caller.

**Request:** Path param `strikeId`. JSON body: `token` (string, required — the one-time appeal token), `message` (string, required, trimmed non-empty).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"token":"tok_abc","message":"Kórházban voltam aznap."}' \
  "$HSR_URL/api/duty-appeal/strike_1"
```

**Response:**

```json
{ "success": true, "appealId": "appeal_1" }
```
Errors: 401 `{"error":"Authentication required"}`; 400 `{"error":"Token and message required"}`; 404 `{"error":"Strike not found"}`; 403 `{"error":"You can only submit your own appeal"}`, `{"error":"Invalid token"}`; 410 `{"error":"Appeal deadline has passed"}`; 409 `{"error":"Appeal already submitted"}`; 500 `{"error":"Failed to submit appeal"}`.

---

### `GET /api/duty/booking/stream`

**Why:** Live push notifying the caller when one of their schedule-request bookings changes status (approved/rejected/etc.), so the booking board updates without polling.

**Auth:** Session cookie or bearer.

**Request:** None. SSE stream.

```bash
curl -sS -N -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/duty/booking/stream"
```

**Response:** SSE via `createSSEResponse()`. On a booking change:
```
data: update

```
The client re-fetches the booking list on receipt. Periodic heartbeat comments keep the connection open.

---

### `GET /api/duty/my/consist`

**Why:** A lightweight poll (every 5s from the active-duty card) of just the caller's active `TRAIN_DRIVER` duty's consist snapshot and pool-match — avoiding the much heavier `/api/duty/my` payload for a value that changes constantly while driving.

**Auth:** Session cookie or bearer (`getSessionWithBearer()` directly).

**Request:** No parameters.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/duty/my/consist"
```

**Response:**

```json
{
  "activeDutyId": "duty_1",
  "consistSnapshot": ["V43 001", "V43 002"],
  "consistMatch": { "matches": true }
}
```
If there is no active driver duty: `{ "activeDutyId": null, "consistSnapshot": null, "consistMatch": null }`. 401 `{"error":"Unauthorized"}`; 500 `{"error":"Failed to fetch duty consist"}`.

---

### `POST /api/dispatch-orders/[id]/sign-mv`

**Why:** The recipient driver ("mozdonyvezető") signs a dispatch order that is awaiting their signature, advancing it to await the dispatcher's countersignature.

**Auth:** Session cookie or bearer, must be the order's `recipientId`, order must be `status: "AWAITING_MV"`.

**Request:** Path param `id`. No body.

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/dispatch-orders/ord_1/sign-mv"
```

**Response:** the updated order (`status: "AWAITING_FSZT"`, `mvSignedAt`/`mvSignName` set). Errors: 404 `{"error":"Nem található"}`; 403 `{"error":"Nincs jogosultság"}`; 409 `{"error":"Ebben az állapotban nem írható alá"}`. Notifies the issuer via the SSE stream.

---

### `POST /api/dispatch-orders/[id]/sign-fszt`

**Why:** The issuing dispatcher ("forgalmi szolgálattevő") gives the second, finalizing signature once the driver has signed — the order becomes legally complete (`FINALIZED`) and the driver is push-notified.

**Auth:** Session cookie or bearer, must be the order's `issuerId`, order must be `status: "AWAITING_FSZT"`.

**Request:** Path param `id`. No body.

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/dispatch-orders/ord_1/sign-fszt"
```

**Response:** the updated order (`status: "FINALIZED"`, `fsztSignedAt`/`fsztSignName` set). Errors: 404 `{"error":"Nem található"}`; 403 `{"error":"Nincs jogosultság"}`; 409 `{"error":"Ebben az állapotban nem írható alá"}`. Sends a push notification "Rendelkezés véglegesítve" to the driver.

---

### `POST /api/dispatch-orders/[id]/send`

**Why:** Issues a DRAFT (or re-sends an `AWAITING_MV`) order to the recipient, assigning it its permanent daily sequence number (`date` + `seq`) and edit-window deadline on first send only.

**Auth:** Session cookie or bearer, must be the order's `issuerId`, order must be `status: "DRAFT"` or `"AWAITING_MV"`.

**Request:** Path param `id`. No body.

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/dispatch-orders/ord_1/send"
```

**Response:** the updated order (`status: "AWAITING_MV"`, `date`/`seq`/`issuedAt`/`editableUntil` set on first send). Errors: 404 `{"error":"Nem található"}`; 403 `{"error":"Nincs jogosultság"}`; 409 `{"error":"Ebben az állapotban nem küldhető"}`. Pushes "Új írásbeli rendelkezés" to the recipient.

---

### `POST /api/documents/[id]/submit`

**Why:** Hands a DRAFT or fixed-up REJECTED document to the signers — the transition to `PENDING` is claimed atomically (`updateMany` on the status) so a double submit cannot mail every signer twice, and every current `SIGN_DOCUMENTS` holder is notified by push and email in their own locale.

**Auth:** Session cookie or bearer, plus `ACCESS_DOCUMENTS`, must be the author.

**Request:** Path param `id`. No body. Document body must be non-empty.

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/documents/doc_1/submit"
```

**Response:** the updated serialized document (`status: "PENDING"`, `submittedAt` set, prior rejection fields cleared, signatures reset to zero). Errors: 403 `{"error":"Csak a készítő küldheti be"}`; 409 `{"error":"A dokumentum már be van küldve"}` (also returned if the atomic claim loses a race); 400 `{"error":"Üres dokumentum nem küldhető be"}`.

---

### `GET /api/documents/[id]/pdf`

**Why:** Streams the signed PDF for a document through the app (rather than exposing a direct R2 URL) so only the author or a signer can retrieve it.

**Auth:** Session cookie or bearer, plus `ACCESS_DOCUMENTS`, and author-or-signer via `requireDocument()`. Document must already carry a `pdfKey` (i.e. be signed).

**Request:** Path param `id`.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" -o document.pdf "$HSR_URL/api/documents/doc_1/pdf"
```

**Response:** raw PDF bytes with `Content-Type: application/pdf`, `Content-Disposition: attachment; filename="<verifyCode>.pdf"`, `Cache-Control: private, max-age=3600`. 409 `{"error":"A dokumentum még nincs aláírva"}` if not yet signed.

---

### `POST /api/documents/[id]/attachments`

**Why:** Attaches an image or PDF (up to `MAX_ATTACHMENTS`) to a document that is still editable, to be rendered into the final signed document.

**Auth:** Session cookie or bearer, plus `ACCESS_DOCUMENTS`, must be the author, document must be DRAFT or REJECTED, and Cloudflare R2 must be configured (`isR2Configured()`).

**Request:** `multipart/form-data` with field `file` — must be PNG/JPEG (`ALLOWED_IMAGE_TYPES`) or PDF (`ALLOWED_PDF_TYPES`), ≤ `MAX_ATTACHMENT_BYTES` (15 MB).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" \
  -F "file=@scan.pdf;type=application/pdf" \
  "$HSR_URL/api/documents/doc_1/attachments"
```

**Response:**

```json
{
  "id": "att_1",
  "kind": "PDF",
  "url": "https://s3.hsrgames.hu/documents/doc_1/scan.pdf",
  "filename": "scan.pdf",
  "size": 20480,
  "contentType": "application/pdf",
  "sortOrder": 0
}
```
Errors: 403 `{"error":"Csak a készítő csatolhat"}`; 409 `{"error":"Beküldött dokumentumhoz nem csatolható melléklet"}`; 400 `{"error":"Legfeljebb N melléklet csatolható"}`, `{"error":"Nincs fájl"}`, `{"error":"Csak PNG, JPEG vagy PDF csatolható"}`, `{"error":"A fájl túl nagy (max. 15 MB)"}`; 500 `{"error":"A fájltárolás nincs beállítva"}`.

---

### `DELETE /api/documents/[id]/attachments/[attachmentId]`

**Why:** Removes an attachment from a document that is still editable, deleting the underlying R2 object too.

**Auth:** Session cookie or bearer, plus `ACCESS_DOCUMENTS`, must be the author, document must be DRAFT or REJECTED.

**Request:** Path params `id`, `attachmentId`. No body.

```bash
curl -sS -X DELETE -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/documents/doc_1/attachments/att_1"
```

**Response:**

```json
{ "ok": true }
```
Errors: 403 `{"error":"Csak a készítő törölheti"}`; 409 `{"error":"Beküldött dokumentum melléklete nem törölhető"}`; 404 `{"error":"A melléklet nem található"}`.

## Exam Slots, Exams, Game Client, Leaderboard, Map, News, Payment Webhooks, Profiles

### `GET /api/exam-slots`

**Why:** Lists the closing-exam time slots a trainee may currently register for or is already registered on, restricted to branches where they are an active trainee (and have finished any required training material).

**Auth:** session/bearer (`requireAuth`).

**Request:** No parameters.

```bash
curl -sS "$HSR_URL/api/exam-slots" \
  -H "Authorization: Bearer $TOKEN"
```

**Response:**
```json
[
  {
    "id": "slot_123",
    "scheduledAt": "2026-09-14T18:00:00.000Z",
    "status": "SCHEDULED",
    "exam": { "title": "Driver Closing Exam", "trainingBranch": "DRIVER" },
    "_count": { "registrations": 3 },
    "isRegistered": true,
    "attended": null,
    "calendarToken": "a1b2c3..."
  }
]
```
Empty array `[]` when the member is not an eligible trainee in any branch.

---

### `GET /api/exam-slots/cancel/[token]`

**Why:** Lets a trainee (or the confirmation email's link) check the details of a slot registration before cancelling it, without requiring login.

**Auth:** token-in-URL (the registration's `cancelToken`, the same token as the confirmation email and the calendar link). No session needed. `GET` only reads — an email client's link-prefetch cannot cancel by itself.

**Request:** `token` path param.

```bash
curl -sS "$HSR_URL/api/exam-slots/cancel/$TOKEN"
```

**Response:**
```json
{
  "examTitle": "Driver Closing Exam",
  "scheduledAt": "2026-09-14T18:00:00.000Z",
  "slotStatus": "SCHEDULED",
  "alreadyCancelled": false
}
```
`404` if the token does not resolve to a registration.

---

### `DELETE /api/exam-slots/cancel/[token]`

**Why:** The actual cancel action, fired by the button on the public cancel page.

**Auth:** token-in-URL (`cancelToken`), no login required.

**Request:** `token` path param, no body.

```bash
curl -sS -X DELETE "$HSR_URL/api/exam-slots/cancel/$TOKEN"
```

**Response:** `{ "ok": true }`. `404` if not found; `400 "This slot can no longer be cancelled"` if the slot is no longer `SCHEDULED` (already active/closed).

---

### `GET /api/exam-slots/calendar/[token]`

**Why:** Serves the slot as a downloadable `.ics` calendar file, keyed by the registration's own token, so the confirmation email's "add to calendar" button works from a phone that isn't signed in.

**Auth:** token-in-URL (`cancelToken`), public.

**Request:** `token` path param.

```bash
curl -sS "$HSR_URL/api/exam-slots/calendar/$TOKEN" -o exam.ics
```

**Response:** `Content-Type: text/calendar; charset=utf-8`, `Content-Disposition: attachment; filename="hsr-vizsga-2026-09-14.ics"`. The event covers the `EXAM_SLOT_WINDOW_MINUTES` (60) window from `scheduledAt`, not the exam's own duration. `404 "Not found"` (plain text) if the token doesn't resolve.

---

### `POST /api/exam-slots/[id]/register`

**Why:** Registers the signed-in trainee onto a scheduled closing-exam slot and emails a confirmation with cancel/calendar links.

**Auth:** session/bearer (`requireAuth`), plus the caller must hold the branch's trainee role and (if the programme has material) have completed it.

**Request:** `id` path param (slot id), no body.

```bash
curl -sS -X POST "$HSR_URL/api/exam-slots/slot_123/register" \
  -H "Authorization: Bearer $TOKEN"
```

**Response:** `{ "ok": true }`, or `{ "ok": true, "alreadyRegistered": true }` on a repeat call (idempotent, no duplicate email). Errors: `400 { "code": "slotUnavailable" }` (slot missing or not `SCHEDULED`), `403 { "code": "notTrainee" }`, `403 { "code": "materialIncomplete" }`.

---

### `DELETE /api/exam-slots/[id]/register`

**Why:** Lets the signed-in trainee withdraw their own registration from the dashboard (as opposed to the emailed token link).

**Auth:** session/bearer (`requireAuth`).

**Request:** `id` path param (slot id), no body.

```bash
curl -sS -X DELETE "$HSR_URL/api/exam-slots/slot_123/register" \
  -H "Authorization: Bearer $TOKEN"
```

**Response:** `{ "ok": true }` (always succeeds even if no registration existed — `deleteMany`).

---

### `GET /api/exams`

**Why:** Lists the exams (including application forms) the signed-in member may currently take, filtering out ones they've passed, ones they're blacklisted/cooldown-blocked from, and closing exams they aren't registered on an active slot for. Titles/descriptions are machine-translated to the reader's locale.

**Auth:** session/bearer (`requireAuth`).

**Request:** No parameters.

```bash
curl -sS "$HSR_URL/api/exams" -H "Authorization: Bearer $TOKEN"
```

**Response:**
```json
[
  {
    "id": "exam_1",
    "title": "Driver Closing Exam",
    "description": "...",
    "timeLimitMode": "TOTAL",
    "timeLimitSeconds": 3600,
    "passingScore": 80,
    "exitIsViolation": true,
    "audience": "CLOSING_EXAM",
    "isApplicationForm": false,
    "applicationFormType": null,
    "questionCount": 20,
    "randomQuestionCount": 15,
    "hasActiveAttempt": false,
    "canStart": true,
    "lastAttempt": null,
    "translation": { "sourceLang": "hu" }
  }
]
```

---

### `GET /api/exams/eligibility`

**Why:** Tells the member *why* application forms are missing from `/api/exams` (account-wide blacklist/ban, or a per-programme reapplication cooldown) — kept separate so the bare-array `GET /api/exams` contract used by web and mobile is not broken.

**Auth:** session/bearer (`requireAuth`).

**Request:** No parameters.

```bash
curl -sS "$HSR_URL/api/exams/eligibility" -H "Authorization: Bearer $TOKEN"
```

**Response:**
```json
{ "blocked": false, "notices": ["A(z) Driver képzésre 2026. október 1. után jelentkezhetsz újra."] }
```
or `{ "blocked": true, "message": "...", "notices": [] }` when account-wide blocked.

---

### `POST /api/exams/[id]/draft`

**Why:** Autosaves the in-progress answer for the current question, used when `exitIsViolation` is false so a page refresh doesn't lose typed work.

**Auth:** session/bearer (`requireAuth`).

**Request:** JSON body: `questionId` (string, optional), `answer` (any, optional). Requires an `IN_PROGRESS` attempt for the exam.

```bash
curl -sS -X POST "$HSR_URL/api/exams/exam_1/draft" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"questionId":"q1","answer":"B"}'
```

**Response:** `{ "success": true }`. `404 "No active attempt"` if none in progress.

---

### `POST /api/exams/[id]/answer`

**Why:** Submits the answer for the exam's current question, auto-grades multiple-choice, advances the attempt pointer, and returns the next (translated) question.

**Auth:** session/bearer (`requireAuth`).

**Request:** JSON body: `questionId` (string, required), `answer` (string, required for MC — must be one of the option keys). Server re-checks the per-total or per-question time limit (5s grace).

```bash
curl -sS -X POST "$HSR_URL/api/exams/exam_1/answer" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"questionId":"q1","answer":"B"}'
```

**Response:**
```json
{
  "completed": false,
  "currentQuestionIndex": 1,
  "nextQuestion": { "id": "q2", "type": "MULTIPLE_CHOICE", "questionText": "...", "options": {"A":"...","B":"..."}, "points": 5, "timeLimitSeconds": null }
}
```
`{ "completed": true, "nextQuestion": null }` on the last question. Errors: `404 "No active attempt"`, `400 "Time expired", "timedOut": true, "timedOutType": "TOTAL"`, `400 "Question time expired"`, `400 "Invalid option..."`, `400 "Question is not the current question"`.

---

### `POST /api/exams/[id]/complete`

**Why:** Finalizes an in-progress attempt: scores it (auto-pass/fail if fully MC), fires the closing-exam finalize pipeline (rank grants, result email) when applicable, and logs the completion.

**Auth:** session/bearer (`requireAuth`).

**Request:** `id` path param, no body.

```bash
curl -sS -X POST "$HSR_URL/api/exams/exam_1/complete" \
  -H "Authorization: Bearer $TOKEN"
```

**Response:**
```json
{ "score": 86.5, "passed": true, "totalPoints": 100, "earnedPoints": 86, "needsReview": false }
```
`passed: null` and `needsReview: true` when TEXT/IMAGE answers still need admin grading. `404 "No active attempt"` / `"Exam not found"` on failure.

---

### `POST /api/exams/[id]/start`

**Why:** Starts (or resumes) an exam attempt for the signed-in member: enforces eligibility (role, closing-exam slot registration, application-form blacklist/membership/cooldown), builds the question set (optionally a random subset), and returns the first (translated) question.

**Auth:** session/bearer (`requireAuth`), rate-limited (`apiLimiter`, key `exam-start:<userId>`).

**Request:** `id` path param, no body.

```bash
curl -sS -X POST "$HSR_URL/api/exams/exam_1/start" \
  -H "Authorization: Bearer $TOKEN"
```

**Response:**
```json
{
  "attemptId": "attempt_1",
  "currentQuestionIndex": 0,
  "totalQuestions": 15,
  "timeLimitMode": "TOTAL",
  "timeLimitSeconds": 3600,
  "exitIsViolation": true,
  "startedAt": "2026-09-13T18:00:00.000Z",
  "draftAnswer": null,
  "question": { "id": "q1", "type": "MULTIPLE_CHOICE", "questionText": "...", "options": {"A":"..."}, "points": 5, "timeLimitSeconds": null }
}
```
Errors: `404 "Exam not available"`, `403 "You are not allowed to take this exam"`, `403 "You are not registered for an active time slot"` (CLOSING_EXAM), `403` blacklist/membership/cooldown messages, `400 "Exam has no questions"`, `400 "You have already passed this exam."`, `400 "Your previous attempt must be archived before retrying."`, `400 "Your previous attempt is still under review."`.

---

### `POST /api/exams/[id]/skip`

**Why:** Skips the current question (records it as unanswered, zero points) and advances to the next.

**Auth:** session/bearer (`requireAuth`).

**Request:** `id` path param, no body.

```bash
curl -sS -X POST "$HSR_URL/api/exams/exam_1/skip" \
  -H "Authorization: Bearer $TOKEN"
```

**Response:** Same shape as `answer` — `{ "completed": false, "currentQuestionIndex": 1, "nextQuestion": {...} }` or `{ "completed": true, "nextQuestion": null }`. `404 "No active attempt"`, `400 "No current question"`, `400 "Invalid question"`.

---

### `POST /api/exams/[id]/violation`

**Why:** Records a proctoring violation (e.g. tab switch/focus loss) when `exam.exitIsViolation` is enabled, immediately failing the attempt.

**Auth:** session/bearer (`requireAuth`).

**Request:** JSON body: `reason` (string, optional, defaults to `"Left the exam page"`).

```bash
curl -sS -X POST "$HSR_URL/api/exams/exam_1/violation" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"reason":"Tab switched"}'
```

**Response:** `{ "success": true }`, or `{ "ignored": true }` if the exam doesn't treat exits as violations. `404 "No active attempt"` if none in progress.

---

### `POST /api/game` (action dispatch)

**Why:** The Roblox game server's single endpoint for reporting joins, leaves, bans, perf stats, and the periodic full-roster sync; drives team/permission assignment, duty auto-ending, and stat tracking.

**Auth:** `x-api-key` header must equal `GAME_API_KEY` (per CLAUDE.md Game Integration). Rate-limited by API key (`gameLimiter`).

**Request:** JSON body, dispatched on `action` field: `"perf_stats"`, `"sync"`, `"ban"`, `"unban"`, `"check_ban"`, `"join"`, `"refresh"`, `"check_duty"`, `"leave"`. Field requirements vary per action (see below).

```bash
# join
curl -sS -X POST "$HSR_URL/api/game" \
  -H "x-api-key: $GAME_API_KEY" -H "Content-Type: application/json" \
  -d '{"action":"join","robloxId":"12345","serverId":"srv-1","playerName":"Foo","playerDisplayName":"Foo","accountAge":365}'

# leave
curl -sS -X POST "$HSR_URL/api/game" \
  -H "x-api-key: $GAME_API_KEY" -H "Content-Type: application/json" \
  -d '{"action":"leave","robloxId":"12345"}'

# ban
curl -sS -X POST "$HSR_URL/api/game" \
  -H "x-api-key: $GAME_API_KEY" -H "Content-Type: application/json" \
  -d '{"action":"ban","robloxId":"12345","reason":"Rule 4","bannedBy":"Moderator"}'

# sync (whole server roster)
curl -sS -X POST "$HSR_URL/api/game" \
  -H "x-api-key: $GAME_API_KEY" -H "Content-Type: application/json" \
  -d '{"action":"sync","serverId":"srv-1","players":[{"robloxId":"12345"}]}'

# refresh (per-60s role re-check)
curl -sS -X POST "$HSR_URL/api/game" \
  -H "x-api-key: $GAME_API_KEY" -H "Content-Type: application/json" \
  -d '{"action":"refresh","robloxId":"12345"}'
```

**Response:**
- `join`: `{ "success": true, "permissions": {...}, "user": { "id": "...", "name": "...", "role": "Trainee Driver", "team": "Train driver", "onDuty": false, "dutyType": null, "permissions": {...} } }`. `{ "success": false, "accountTooNew": true }` for accounts under 20 days with no web link. `{ "success": false, "message": "User is banned", "banned": true }`.
- `leave`: `{ "success": true, "dutyEnded": true }`.
- `ban`/`unban`: `{ "success": true, "message": "User has been banned", "dutyEnded": true }`.
- `check_ban`: `{ "success": true, "banned": false, "reason": null }`.
- `sync`: `{ "success": true, "synced": 12, "staleRemoved": 1, "staleNames": [...], "staleServers": 0, "newJoins": [...], "dutiesEnded": 1, "playerStates": [{ "robloxId": "...", "role": "...", "onDuty": true, "dutyType": "TRAIN_DRIVER", "team": "Train driver", "permissions": {...} }] }`.
- `refresh`: same `user`/`permissions` shape as `join`.
- `check_duty`: `{ "success": true, "onDuty": false, "dutyType": null, "role": "...", "team": "Passengers" }`.
- `perf_stats`: `{ "success": true }`.
- Errors: `401 "Unauthorized"` (bad/missing key), `400 "Missing required field: action"` / `robloxId` / `serverId` / `players (array)`, `400 "Invalid action"`.

---

### `GET /api/game` (lookup)

**Why:** Roblox-side lookup of a single player's status, or the full list of currently active players.

**Auth:** `x-api-key: GAME_API_KEY`.

**Request:** Query param `robloxId` (optional — omit for the full active-player list).

```bash
curl -sS "$HSR_URL/api/game?robloxId=12345" -H "x-api-key: $GAME_API_KEY"
curl -sS "$HSR_URL/api/game" -H "x-api-key: $GAME_API_KEY"
```

**Response:**
```json
{ "found": true, "user": { "id": "...", "name": "...", "displayName": "...", "banned": false, "role": "Member", "team": "Passengers", "isActive": true } }
```
or `{ "found": false, "role": "Passenger", "team": "Passengers" }`; the no-`robloxId` form returns `{ "activePlayers": [...] }`.

---

### `OPTIONS /api/game`

**Why:** CORS preflight for the Roblox client's cross-origin fetch.

**Auth:** public.

**Response:** `200`, empty body, with `Access-Control-Allow-Origin: https://www.roblox.com` etc.

---

### `GET /api/game/user`

**Why:** On-demand single-player permission/team lookup for the game — the cache-miss and immediate-re-read complement to the `join`/`sync` batch, sharing the same `buildGamePermissions()` so the two can never disagree.

**Auth:** `x-api-key: GAME_API_KEY`, rate-limited (`gameLimiter`).

**Request:** Query param `id` (Roblox id, required).

```bash
curl -sS "$HSR_URL/api/game/user?id=12345" -H "x-api-key: $GAME_API_KEY"
```

**Response:**
```json
{
  "success": true,
  "registered": true,
  "robloxId": "12345",
  "name": "Foo",
  "displayName": "Foo",
  "banned": false,
  "role": "Trainee Driver",
  "team": "Train driver",
  "onDuty": true,
  "dutyType": "TRAIN_DRIVER",
  "permissions": {...}
}
```
A player with no site account returns `{ "success": true, "registered": false, "robloxId": "12345", "banned": false, "team": "Passengers", "onDuty": false, "dutyType": null, "permissions": {...guest...} }` (not an error). `400` missing `id`, `401` bad key.

---

### `GET /api/game/radio/active-duty`

**Why:** Feeds the in-game radio's target picker — the list of players currently on a radio-eligible duty (driver/dispatcher/inspector) on the *same* Roblox server, since delivery only works within a server.

**Auth:** `x-api-key: GAME_API_KEY`.

**Request:** Query param `serverId` (required).

```bash
curl -sS "$HSR_URL/api/game/radio/active-duty?serverId=srv-1" -H "x-api-key: $GAME_API_KEY"
```

**Response:**
```json
{ "players": [{ "robloxId": "12345", "name": "Foo", "team": "Train driver" }] }
```
`400 "serverId required"` if missing.

---

### `POST /api/game/radio/send`

**Why:** Records an in-game radio message (broadcast or targeted), sent by the Roblox server script after it has already validated team/target/rate limits itself.

**Auth:** `x-api-key: GAME_API_KEY`. The route trusts `senderRobloxId` etc. only because the caller is the server script, not a player client.

**Request:** JSON body: `senderRobloxId` (string, required), `senderTeam` (string, optional), `targetRobloxId` (string, required unless `isBroadcast`), `text` (string, required, ≤500 chars), `isBroadcast` (boolean, optional), `serverId` (string, optional).

```bash
curl -sS -X POST "$HSR_URL/api/game/radio/send" \
  -H "x-api-key: $GAME_API_KEY" -H "Content-Type: application/json" \
  -d '{"senderRobloxId":"12345","senderTeam":"Train driver","targetRobloxId":"67890","text":"Hello","serverId":"srv-1"}'
```

**Response:** `{ "id": "msg_1", "createdAt": "2026-09-13T18:00:00.000Z" }`. `400` missing/invalid fields, `404 "Unknown sender"` / `"Unknown target"`.

---

### `GET /api/game/radio/poll`

**Why:** Lets each Roblox server poll for new radio messages sent since a cursor, scoped to its own `serverId` so servers don't see each other's traffic. Text is served censored.

**Auth:** `x-api-key: GAME_API_KEY`.

**Request:** Query params: `serverId` (required), `since` (ISO date string, optional — defaults to epoch).

```bash
curl -sS "$HSR_URL/api/game/radio/poll?serverId=srv-1&since=2026-09-13T18:00:00Z" \
  -H "x-api-key: $GAME_API_KEY"
```

**Response:**
```json
{
  "messages": [
    { "id": "msg_1", "senderRobloxId": "12345", "senderName": "Foo", "senderTeam": "Train driver", "targetRobloxId": "67890", "targetName": "Bar", "text": "H*llo", "isBroadcast": false, "createdAt": "2026-09-13T18:00:01.000Z" }
  ],
  "cursor": "2026-09-13T18:00:01.000Z"
}
```
`400` missing/invalid `serverId`/`since`.

---

### `GET /api/instructor/me`

**Why:** An instructor's own weekly quota progress (supervision credits/sessions), derived read-only from the supervisions they ran.

**Auth:** session/bearer (`requireAuth`).

**Request:** No parameters.

```bash
curl -sS "$HSR_URL/api/instructor/me" -H "Authorization: Bearer $TOKEN"
```

**Response:**
```json
{
  "isInstructor": true,
  "weekStart": "2026-09-07T00:00:00.000Z",
  "weekEnd": "2026-09-13T23:59:59.999Z",
  "row": { "userId": "...", "credits": 3, "sessions": 2, "..." : "..." },
  "rules": { "requiredCredits": 4, "requiredSessions": 2 }
}
```
`{ "isInstructor": false }` for a non-instructor or a leadership-tier holder excluded from the list.

---

### `GET /api/leaderboard/[type]`

**Why:** Public leaderboard for one of the tracked stats, with the caller's own rank included if signed in.

**Auth:** public for the list; session/bearer optional (adds `userRank` when present).

**Request:** `type` path param — one of `xp`, `distance`, `playtime`, `trains_driven`. Query params: `limit` (default 50, max 100), `offset` (default 0).

```bash
curl -sS "$HSR_URL/api/leaderboard/xp?limit=20&offset=0"
```

**Response:**
```json
{
  "leaderboard": [{ "rank": 1, "userId": "u1", "name": "Foo", "displayName": "Foo", "image": null, "steamId": null, "value": 15230, "level": 12 }],
  "total": 480,
  "userRank": 37
}
```
`400 "Invalid leaderboard type..."` for an unknown `type`.

---

### `GET /api/maintenance-status`

**Why:** Public, unauthenticated check the `/maintenance` page uses to display the admin's custom message even to users locked out of every admin-gated route.

**Auth:** public.

**Request:** No parameters.

```bash
curl -sS "$HSR_URL/api/maintenance-status"
```

**Response:** `{ "enabled": false, "message": null }`.

---

### `GET /api/map/snapshot`

**Why:** One-shot fetch of the live driver-position map data (for a client that doesn't want an SSE connection).

**Auth:** session/bearer (`getSessionWithBearer`).

**Request:** No parameters.

```bash
curl -sS "$HSR_URL/api/map/snapshot" -H "Authorization: Bearer $TOKEN"
```

**Response:** `getLiveMapData()`'s shape — an array/object of live driver positions and metadata; `[]` on an internal error rather than a 500, and `no-store` cache headers. `401` if unauthenticated.

---

### `GET /api/map/live`

**Why:** SSE stream of the same live map data, pushed every 10 seconds, for a persistent live-updating view.

**Auth:** session/bearer (`getSessionWithBearer`).

**Request:** No parameters. Standard `EventSource` GET.

```bash
curl -sS -N "$HSR_URL/api/map/live" -H "Authorization: Bearer $TOKEN"
```

**Response:** `text/event-stream`; each event's `data` is a JSON payload from `getLiveMapData()`, sent immediately then every 10s. `401` (plain `Response`) if no session.

---

### `POST /api/missed-assignment-appeal/[id]`

**Why:** Lets a driver/dispatcher appeal a missed-assignment strike via a one-time token from their notification, emailing every admin so the appeal can be reviewed.

**Auth:** session/bearer (`getSessionWithBearer`) **and** a token-in-body matching the assignment's `appealToken` (single-use — cleared on submit). The caller must also own the assignment (`assignment.userId === session.id`).

**Request:** `id` path param (missed assignment id). JSON body: `token` (string, required), `message` (string, required).

```bash
curl -sS -X POST "$HSR_URL/api/missed-assignment-appeal/assign_1" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"token":"abc123","message":"I was disconnected mid-shift."}'
```

**Response:** `{ "success": true, "appealId": "appeal_1" }`. Errors: `401` unauthenticated, `404 "Assignment not found"`, `403 "You can only submit your own appeal"` / `"Invalid token"`, `410 "Appeal deadline has passed"`, `409 "Appeal already submitted"`.

---

### `POST /api/polar/webhook` (Stripe checkout webhook, despite the path name)

**Why:** Handles the Stripe `checkout.session.completed` event to finalize either a shop order or a support donation — the directory is legacy-named `polar` but the implementation is Stripe.

**Auth:** signed webhook — verifies the `stripe-signature` header against `STRIPE_WEBHOOK_SECRET` via `stripe.webhooks.constructEvent()`.

**Request:** Raw Stripe event body (read as text for signature verification).

```bash
curl -sS -X POST "$HSR_URL/api/polar/webhook" \
  -H "stripe-signature: t=...,v1=..." -H "Content-Type: application/json" \
  --data-binary @stripe-event.json
```

**Response:** `{ "received": true }` for any handled or ignored event type. Side effect: `session.metadata.type === "shop"` calls `finalizeShopOrderByCheckoutId()`; otherwise calls `finalizeDonationByCheckoutId()` using `session.metadata.userId`. `400 "Missing webhook signature/secret"` / `"Invalid signature"` on verification failure.

---

### `POST /api/polar/checkout`

**Why:** Creates a Stripe Checkout session for a one-off HUF donation and records a pending `Donation` row.

**Auth:** session/bearer (`getSessionWithBearer`).

**Request:** JSON body: `amount` (number, required, minimum 175 HUF).

```bash
curl -sS -X POST "$HSR_URL/api/polar/checkout" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"amount":2000}'
```

**Response:** `{ "url": "https://checkout.stripe.com/..." }`. `401` unauthenticated, `400 "Érvénytelen összeg (minimum 175 Ft)"` for an invalid amount.

---

### `POST /api/printful/webhook`

**Why:** Receives Printful print-on-demand order events (shipment tracking, fulfillment failure) and updates the corresponding `ShopOrder`.

**Auth:** signed webhook — HMAC-SHA256 over the raw body with `PRINTFUL_WEBHOOK_SECRET`, compared via `x-pf-webhook-signature` header using `crypto.timingSafeEqual`. No secret configured closes the endpoint (`503`), it does not skip verification.

**Request:** Raw JSON body; `event.type` dispatches (`package_shipped`, `order_failed`, others ignored).

```bash
curl -sS -X POST "$HSR_URL/api/printful/webhook" \
  -H "x-pf-webhook-signature: <hmac-sha256-hex>" -H "Content-Type: application/json" \
  --data-binary @printful-event.json
```

**Response:** `{ "received": true }` always (once past signature check). Side effects: `package_shipped` sets `ShopOrder.status = "SHIPPED"` with tracking info and emails the customer (`shopOrderShipped`, sent `now`); `order_failed` sets `status = "FULFILLMENT_FAILED"` and pushes admins. `503 "Webhook not configured"` if no secret set, `400 "Invalid signature"` on mismatch.

---

### `GET /api/profile/praise`

**Why:** Answers "have I already thanked this colleague?" for the praise dialog on their profile, plus the list of praise reasons to offer.

**Auth:** session/bearer (`requireAuth`).

**Request:** Query param `userId` (optional — the colleague being viewed).

```bash
curl -sS "$HSR_URL/api/profile/praise?userId=u2" -H "Authorization: Bearer $TOKEN"
```

**Response:** `{ "reasons": ["helpful", "friendly", "..."], "alreadyPraised": false }`.

---

### `POST /api/profile/praise`

**Why:** Gives a "+1" to a colleague worked with in the last `PRAISE_WINDOW_HOURS`, then re-syncs both members' badges immediately (a threshold may have just been crossed).

**Auth:** session/bearer (`requireAuth`).

**Request:** JSON body: `toUserId` (string, required), `reason` (string, optional), `message` (string, optional).

```bash
curl -sS -X POST "$HSR_URL/api/profile/praise" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"toUserId":"u2","reason":"helpful","message":"Thanks for the assist!"}'
```

**Response:** `{ "ok": true, "id": "praise_1" }`. Errors: `400 "SELF"` / `"INVALID"`, `409 "ALREADY_PRAISED"`.

---

### `GET /api/profile/weekly-duty`

**Why:** The signed-in member's progress toward their weekly duty-hour minimum (prorated for approved leave), plus their active/recent weekly-duty strikes and whether they're appealable.

**Auth:** session/bearer (`getSessionWithBearer`).

**Request:** No parameters.

```bash
curl -sS "$HSR_URL/api/profile/weekly-duty" -H "Authorization: Bearer $TOKEN"
```

**Response:**
```json
{
  "weekStart": "2026-09-07T00:00:00.000Z",
  "weekEnd": "2026-09-13T23:59:59.999Z",
  "minimumSeconds": 21600,
  "minimumHours": 6,
  "totalSeconds": 10800,
  "totalHours": 3,
  "progressPercent": 50,
  "isEligible": true,
  "isExempted": false,
  "canDrive": true,
  "canDispatch": false,
  "canInspect": false,
  "activeStrike": null,
  "recentStrikes": []
}
```
`401` unauthenticated, `404 "User not found"`.

---

### `GET /api/profile/locomotives`

**Why:** Feeds the "favourite locomotive" picker in the profile editor with the locomotive chip pool (number + photo) — a shortcut, not the exhaustive set of valid answers (a free-text field also exists).

**Auth:** session/bearer (`requireAuth`).

**Request:** No parameters.

```bash
curl -sS "$HSR_URL/api/profile/locomotives" -H "Authorization: Bearer $TOKEN"
```

**Response:** `{ "chips": [{ "id": "c1", "number": "V63 001", "imageUrl": "https://...", "operator": "MÁV" }] }`.

---

### `GET /api/profile/me`

**Why:** The signed-in member's own profile, including sections they've hidden from the public (`includeHidden: true`), plus the banner presets and whether Spotify linking is available.

**Auth:** session/bearer (`requireAuth`).

**Request:** No parameters.

```bash
curl -sS "$HSR_URL/api/profile/me" -H "Authorization: Bearer $TOKEN"
```

**Response:** `{ "profile": {...ProfileView}, "spotifyAvailable": true, "bannerPresets": [...] }`. `404 "Profile not found"` (should not normally happen — `ensureProfile()` elsewhere creates it).

---

### `PATCH /api/profile/me`

**Why:** Edits the signed-in member's own profile fields (handle, motto, accent colour, banner preset/framing, favourite locomotive, visibility, section toggles). Every rule is re-validated server-side since the editor is a form a `fetch` call can skip.

**Auth:** session/bearer (`requireAuth`).

**Request:** JSON body, all fields optional (absent = unchanged, `null` = clear): `handle` (string|null), `motto` (string|null, ≤`MOTTO_MAX_LENGTH`), `accentColor` (hex string|null), `bannerPreset` (string key|null), `bannerPosX`/`bannerPosY`/`bannerZoom` (number, clamped), `favoriteLocomotive` (string|null), `favoriteChipId` (string|null), `visibility` (`PUBLIC`|`MEMBERS`|`PRIVATE`), `showStats`/`showBadges`/`showSpotify`/`showPraise` (boolean).

```bash
curl -sS -X PATCH "$HSR_URL/api/profile/me" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"motto":"Always on time","visibility":"PUBLIC"}'
```

**Response:** `{ "profile": {...updated ProfileView} }`. Errors: `400 "handleFormat"` / `"handleReserved"` / `"accentInvalid"` / `"bannerInvalid"` / `"visibilityInvalid"` / `"Nothing to update"`, `404 "chipNotFound"`, `409 "handleTaken"`.

---

### `GET /api/profile/stats`

**Why:** The dashboard's own duty/game-activity stats for the signed-in member — today/week/month/all-time duty minutes and game joins, a 7-day activity chart, and their monthly duty-count leaderboard rank.

**Auth:** session/bearer (`getSessionWithBearer`).

**Request:** No parameters.

```bash
curl -sS "$HSR_URL/api/profile/stats" -H "Authorization: Bearer $TOKEN"
```

**Response:**
```json
{
  "today": { "duties": 1, "completedDuties": 1, "activeDuty": false, "totalMinutes": 120, "gameJoins": 2 },
  "week": { "duties": 4, "totalMinutes": 480, "gameJoins": 6 },
  "month": { "duties": 12, "totalMinutes": 1440, "gameJoins": 20, "byType": { "trainDriver": 8, "trainDispatcher": 4 } },
  "allTime": { "duties": 130, "gameJoins": 210 },
  "activeDuty": null,
  "leaderboardRank": 5,
  "chart": [{ "date": "2026-09-07", "duties": 1, "gameJoins": 1, "minutes": 90 }]
}
```
`401` unauthenticated.

---

### `GET /api/profiles/[handle]`

**Why:** The public member profile page's data — resolves a handle, id, or Roblox/Discord name to a user and returns their `ProfileView`, respecting the per-profile visibility setting.

**Auth:** public by default (visibility `PUBLIC`); session/bearer required to view a `MEMBERS`/`PRIVATE` profile the caller doesn't own or doesn't have `MANAGE_USERS` for.

**Request:** `handle` path param (a claimed handle, a user id, or the Roblox/Discord display name).

```bash
curl -sS "$HSR_URL/api/profiles/johndoe"
```

**Response:** `{ "profile": {...ProfileView}, "isOwner": false }`. `404 "Profile not found"`, `403 "This profile is private"`.

---

### `GET /api/profiles/[handle]/spotify`

**Why:** "Most ezt hallgatja" now-playing card, polled while a profile page is open. Every no-data case (no link, section off, private profile, Spotify down) collapses to the same `nowPlaying: null` shape rather than an error.

**Auth:** public for a `PUBLIC` profile with Spotify visible; session/bearer required for a non-public profile (and must be the owner for `PRIVATE`).

**Request:** `handle` path param.

```bash
curl -sS "$HSR_URL/api/profiles/johndoe/spotify"
```

**Response:** `{ "nowPlaying": { "track": "...", "artist": "...", "albumArt": "...", "isPlaying": true } }` or `{ "nowPlaying": null }`.

---

### `DELETE /api/profile/me/spotify`

**Why:** Unlinks the signed-in member's Spotify account by clearing the stored refresh token (never just a flag, so "unlinked" and "still holds a token" can't disagree).

**Auth:** session/bearer (`requireAuth`).

**Request:** No body.

```bash
curl -sS -X DELETE "$HSR_URL/api/profile/me/spotify" -H "Authorization: Bearer $TOKEN"
```

**Response:** `{ "ok": true }`.

---

### `GET /api/profile/me/badges`

**Why:** The signed-in member's badge shelf: held badges (with pin state) and the not-yet-held catalogue with progress bars toward automatic rules. Also re-syncs automatic badges on load.

**Auth:** session/bearer (`requireAuth`).

**Request:** No parameters.

```bash
curl -sS "$HSR_URL/api/profile/me/badges" -H "Authorization: Bearer $TOKEN"
```

**Response:**
```json
{
  "held": [{ "id": "b1", "key": "jo-kolega", "name": "Jó kolléga", "nameEn": "Good Colleague", "imageUrl": null, "emoji": "🤝", "rarity": "COMMON", "category": "SOCIAL", "pinned": true, "awardedAt": "2026-01-01T00:00:00.000Z", "reason": null }],
  "available": [{ "id": "b2", "key": "veteran", "name": "Veterán", "progress": { "current": 400, "threshold": 1000 } }],
  "maxPinned": 3
}
```

---

### `PATCH /api/profile/me/badges`

**Why:** Sets which held badges are pinned above the rest (whole set replaced at once, so the client can't bypass the pin cap).

**Auth:** session/bearer (`requireAuth`).

**Request:** JSON body: `pinned` (array of badge ids, ≤`MAX_PINNED_BADGES`, must all be badges the member holds).

```bash
curl -sS -X PATCH "$HSR_URL/api/profile/me/badges" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"pinned":["b1","b2"]}'
```

**Response:** `{ "ok": true, "pinned": ["b1","b2"] }`. `400 "tooManyPinned"` / `"notHeld"`.

---

### `POST /api/profile/me/banner`

**Why:** Uploads a custom banner image to R2 for the profile header, replacing any preset and deleting the previous uploaded file (best-effort, after the new one is live).

**Auth:** session/bearer (`requireAuth`); requires R2 configured.

**Request:** `multipart/form-data` with a `file` field — image/jpeg, image/png, image/webp or image/gif, ≤5MB.

```bash
curl -sS -X POST "$HSR_URL/api/profile/me/banner" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@banner.jpg;type=image/jpeg"
```

**Response:** `{ "bannerUrl": "https://s3.hsrgames.hu/profile-banners/<userId>/<uuid>.jpg" }`. `503 "Storage not configured"`, `400 "No file provided"` / `"bannerType"` / `"bannerTooLarge"`.

---

### `DELETE /api/profile/me/banner`

**Why:** Reverts the profile banner to a preset, deleting the uploaded object since nothing references it any more.

**Auth:** session/bearer (`requireAuth`).

**Request:** No body.

```bash
curl -sS -X DELETE "$HSR_URL/api/profile/me/banner" -H "Authorization: Bearer $TOKEN"
```

**Response:** `{ "ok": true }`.

---

### `GET /api/news`

**Why:** Public homepage news feed, mirrored from a Discord channel; re-syncs itself if the mirror is stale (>1 hour) rather than relying solely on the cron.

**Auth:** public.

**Request:** Query param `limit` (optional, default 6, clamped 1–20).

```bash
curl -sS "$HSR_URL/api/news?limit=6"
```

**Response:** `{ "items": [{ "id": "1234567890", "title": "...", "summary": "...", "publishedAt": "2026-09-10T12:00:00.000Z" }] }`. `500 "Failed to fetch news"` on unexpected error.

---

### `GET /api/roblox/stats`

**Why:** Public live game stats (players online, visits, favorites, votes) pulled from the Roblox public APIs for the homepage.

**Auth:** public.

**Request:** No parameters.

```bash
curl -sS "$HSR_URL/api/roblox/stats"
```

**Response:** `{ "playing": 42, "visits": 1500000, "favorites": 12000, "maxPlayers": 50, "upVotes": 900, "downVotes": 20 }`, cached `s-maxage=30`. `502 "Could not resolve universe"` / `"Roblox API unavailable"`, `404 "Game not found"`, `500 "Internal error"`.

## Route Permits, Schedule, Search, Sessions, Shop, Stripe

### `GET /api/route-permit`

**Why:** Lists a driver's or dispatcher's own route permits (driver requests a permit from a dispatcher before running a service) plus, for a dispatcher, the incoming pending permits waiting on their approval. An admin-panel user sees everything.

**Auth:** session (`requireAuth()`). No extra flag — visibility is scoped to `driverId`/`dispatcherId` matching the caller, unless `permissions.canViewAdminPanel()` is true, in which case every permit is returned.

**Request:** no parameters.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/route-permit"
```

**Response:**
```json
{
  "mine": [
    {
      "id": "clx1...",
      "driverId": "clu9...",
      "dispatcherId": "clu8...",
      "trainId": "clt1...",
      "pairGroup": "IC-101",
      "locomotiveNumber": "V63 001",
      "departureStation": "Budapest",
      "arrivalStation": "Győr",
      "trainName": "IC 101",
      "status": "PENDING",
      "deletedByDriver": false,
      "deletedByFszt": false,
      "createdAt": "2026-09-13T05:00:00.000Z",
      "driver": { "id": "clu9...", "displayName": "Kovács János", "name": "kovacs", "discordUsername": "kovacs#0" },
      "dispatcher": { "id": "clu8...", "displayName": "Nagy Éva", "name": "nagy" },
      "train": { "id": "clt1...", "trainNumber": "101", "trainType": "IC", "pairGroup": "IC-101", "fordaGroup": null, "stops": [] }
    }
  ],
  "incoming": []
}
```

---

### `POST /api/route-permit`

**Why:** A driver requests a route permit from a named dispatcher before starting a service. Departure/arrival stations are auto-derived from the train's own stops (accounting for direction) rather than trusted from the client.

**Auth:** session; requires `permissions.canDrive()` (`CAN_DRIVE`), otherwise 403.

**Request:** JSON body — `trainId` (string, required), `pairGroup` (string, required), `locomotiveNumber` (string, required), `trainName` (string, required), `dispatcherId` (string, required).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"trainId":"clt1...","pairGroup":"IC-101","locomotiveNumber":"V63 001","trainName":"IC 101","dispatcherId":"clu8..."}' \
  "$HSR_URL/api/route-permit"
```

**Response:** `201` with the created permit row. `400` if any required field is missing. Emits a real-time update to the named dispatcher via `emitPermitUpdate()`.
```json
{ "id": "clx2...", "driverId": "clu9...", "dispatcherId": "clu8...", "trainId": "clt1...", "status": "PENDING", "...": "..." }
```

---

### `PATCH /api/route-permit/[id]`

**Why:** The three permit lifecycle actions — a dispatcher approving a permit, and either side removing it from their own view.

**Auth:** session. `approve` and `delete_fszt` require the caller to be the permit's `dispatcherId` (or an admin-panel user); `delete_driver` requires the caller to be the permit's `driverId` (or an admin-panel user).

**Request:** JSON body — `action`: `"approve" | "delete_driver" | "delete_fszt"` (required).

```bash
curl -sS -X PATCH -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"action":"approve"}' \
  "$HSR_URL/api/route-permit/clx2..."
```

**Response:** `approve` returns the updated permit (status `APPROVED`) and pushes an update to the driver via `emitPermitUpdate()`. `delete_driver`/`delete_fszt` set `deletedByDriver`/`deletedByFszt` (soft delete — the row stays for the other side) and return `{ "ok": true }`. `404` if the permit does not exist, `403` on ownership mismatch, `400` on an unrecognized `action`.

---

### `GET /api/route-permit/stream`

**Why:** Real-time push so a driver's or dispatcher's permit list refreshes without polling — sent as a bare "update" event that tells the client to re-fetch `/api/route-permit`, not the row itself.

**Auth:** session (`requireAuth()`).

**Request:** no parameters. Built with `createSSEResponse()` per the repo's SSE convention.

```bash
curl -sS -N -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/route-permit/stream"
```

**Response:** SSE stream. Each event is simply the literal data `update`, emitted whenever `emitPermitUpdate(userId)` fires for the connected user (on permit create/approve). The client treats it as "something changed, refetch."
```
data: update

```

---

### `GET /api/schedule-request/available`

**Why:** Feeds the booking form with the trains (driver) or stations (dispatcher) still bookable for a given date — filtering out what the caller already booked, what's booked by someone else, what's already assigned to a duty, what requires a locomotive-type role the caller lacks, operator (MAV/GYSEV) mismatches, and — for today only — trains that have already departed.

**Auth:** session (`requireAuth()`). No extra permission flag to *view* availability (booking itself is gated on `POST /api/schedule-request`).

**Request:** query params — `date` (`YYYY-MM-DD`, required), `type` (`"DRIVER" | "DISPATCHER"`, optional — defaults to `DRIVER` behavior when omitted).

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/schedule-request/available?date=2026-09-14&type=DRIVER"
```

**Response:**
```json
{
  "type": "DRIVER",
  "date": "2026-09-14T00:00:00.000Z",
  "items": [
    {
      "id": "clt1...",
      "trainNumber": "101",
      "trainType": "IC",
      "direction": 0,
      "locomotiveNum": 1,
      "locomotiveType": "V63",
      "hasAC": true,
      "isAvailable": true
    }
  ]
}
```
`400` if `date` is missing or `type` is neither `DRIVER` nor `DISPATCHER`; `500` on unexpected failure.

---

### `GET /api/schedule-request`

**Why:** A member's own submitted schedule requests for a given month, for the dashboard's "my bookings" view.

**Auth:** session.

**Request:** query params — `year` (integer, optional, defaults to current year), `month` (1-12, optional, defaults to current month).

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/schedule-request?year=2026&month=9"
```

**Response:** array of requests.
```json
[
  {
    "id": "clr1...",
    "userId": "clu9...",
    "requestType": "DRIVER",
    "date": "2026-09-14T00:00:00.000Z",
    "status": "PENDING",
    "trainId": "clt1...",
    "stationId": null,
    "train": { "id": "clt1...", "trainNumber": "101", "trainType": "IC", "direction": 0 },
    "station": null
  }
]
```

---

### `POST /api/schedule-request`

**Why:** Submit a driver or dispatcher booking. Driver bookings are constrained to the open Mon–Sun booking window, checked for pair-level conflicts (a return train shares the outbound's slot), operator role, GYSEV eligibility, and — for members with unappealed weekly-duty strikes — capped at 3 bookings/week. Dispatcher bookings validate a station and an HH:MM time range. Same-day submissions are optionally auto-processed (`autoProcessSameDayRequest`) if the admin toggle is on.

**Auth:** session; requires `permissions.canSubmitScheduleRequests()` (`SUBMIT_SCHEDULE_DRIVER` or `SUBMIT_SCHEDULE_DISPATCHER`), otherwise 403.

**Request:** JSON body — `requestType` (`"DRIVER" | "DISPATCHER"`, required), `date` (`YYYY-MM-DD`, required). For `DRIVER`: `trainId` (string, required unless `isNonScheduledFreight`), `isNonScheduledFreight` (boolean, optional), `freightLabel` (string, required if freight, 2–100 chars), `note` (string, optional). For `DISPATCHER`: `stationId` (string, required), `dispatchStartTime`/`dispatchEndTime` (`HH:MM`, required, start < end), `note` (string, optional).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"requestType":"DRIVER","date":"2026-09-14","trainId":"clt1..."}' \
  "$HSR_URL/api/schedule-request"
```

**Response:** `201` with the created (or same-day auto-assigned) request. Errors: `400` invalid type/date/past date/outside booking window/weekly cap reached/train departed/duplicate active request on the pair; `403` operator mismatch (GYSEV/MAV); `404` train or station not found.
```json
{ "id": "clr2...", "requestType": "DRIVER", "date": "2026-09-14T00:00:00.000Z", "status": "PENDING", "train": { "...": "..." }, "station": null }
```

---

### `DELETE /api/schedule-request`

**Why:** Cancel one's own still-pending booking. A driver booking covers the whole train pair, so cancelling one side deletes the sibling pending request for the same pair/day too — otherwise the timetable would keep showing the driver on the half they didn't cancel.

**Auth:** session; ownership enforced (`request.userId !== session.id` → 403).

**Request:** query param — `id` (schedule request id, required).

```bash
curl -sS -X DELETE -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/schedule-request?id=clr2..."
```

**Response:** `{ "success": true }`. `400` if `id` missing or the request is not `PENDING`; `404` if not found; `403` if not the owner.

---

### `GET /api/schedule`

**Why:** The public timetable — every active train with its stops, plus who's driving each (from today's duties and approved schedule requests, propagated across paired trains), and a per-train fulfillment status (`active`/`completed`/`not_fulfilled`/`pending`) computed against the train's first departure time.

**Auth:** public — no auth call in this route at all.

**Request:** no parameters.

```bash
curl -sS "$HSR_URL/api/schedule"
```

**Response:**
```json
{
  "stations": [{ "id": "cls1...", "name": "Budapest", "km": 0, "order": 1 }],
  "outboundTrains": [
    {
      "id": "clt1...",
      "trainNumber": "101",
      "trainType": "IC",
      "direction": 0,
      "driver": { "id": "clu9...", "displayName": "Kovács János", "name": "kovacs", "image": null },
      "assignedDriver": { "id": "clu9...", "displayName": "Kovács János", "name": "kovacs", "image": null },
      "dutyStatus": "active",
      "stops": [{ "arrivalTime": null, "departureTime": "08:00", "station": { "...": "..." } }]
    }
  ],
  "returnTrains": []
}
```

---

### `GET /api/search`

**Why:** The Command Palette's (⌘K) one endpoint. Filtering happens server-side, never in the client palette, because a member who may not open the user list must not be able to read names out of it by typing. Page results ask the same navigation permission checks the sidebars use (`src/lib/search/pages.ts`); member results apply the exact profile-visibility rule the profile page enforces (a `PRIVATE` profile or a banned account is invisible without `MANAGE_USERS`).

**Auth:** session (`requireAuth()`).

**Request:** query param — `q` (string, trimmed and capped at 64 chars; queries under 2 characters return an empty result set without querying anything).

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/search?q=jarmu"
```

**Response:** results are capped at 6 per group (pages/wiki/members) and 20 total; `truncated` reports whether any group was cut.
```json
{
  "query": "jarmu",
  "results": [
    { "id": "page:vehicles", "kind": "page", "title": "Járművek", "href": "/admin/locomotive-pool" },
    { "id": "wiki:clw1...", "kind": "wiki", "title": "Járműismertető", "subtitle": "...", "badge": "Technika", "href": "/wiki/jarmuismerteto" },
    { "id": "member:clu9...", "kind": "member", "title": "Kovács János", "subtitle": "Train Driver", "href": "/profile/kovacs" }
  ],
  "truncated": false
}
```

---

### `GET /api/sessions`

**Why:** Lists the caller's own active web and mobile sessions (for the "manage devices" settings page), marking which one is the current browser session.

**Auth:** session.

**Request:** no parameters.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/sessions"
```

**Response:**
```json
[
  {
    "id": "clw1...",
    "type": "web",
    "deviceInfo": "Chrome on macOS",
    "ipAddress": "203.0.113.4",
    "lastUsedAt": "2026-09-13T09:00:00.000Z",
    "createdAt": "2026-09-01T09:00:00.000Z",
    "expiresAt": "2026-09-15T09:00:00.000Z",
    "isCurrent": true
  },
  {
    "id": "clm1...",
    "type": "mobile",
    "deviceInfo": "iPhone 15",
    "ipAddress": null,
    "lastUsedAt": "2026-09-12T10:00:00.000Z",
    "createdAt": "2026-08-01T10:00:00.000Z",
    "expiresAt": "2026-11-01T10:00:00.000Z",
    "isCurrent": false
  }
]
```

---

### `DELETE /api/sessions`

**Why:** Revoke one specific session (log out a specific device) — self-only, this is not an admin session-management endpoint.

**Auth:** session; ownership is enforced by fetching the target session and comparing `userId` to the caller (404 rather than 403 if it belongs to someone else, to avoid confirming the id exists).

**Request:** JSON body — `sessionId` (string, required), `type` (`"web" | "mobile"`, required).

```bash
curl -sS -X DELETE -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"sessionId":"clm1...","type":"mobile"}' \
  "$HSR_URL/api/sessions"
```

**Response:** `{ "success": true }`. `400` missing fields or invalid `type`; `404` session not found or not owned by the caller.

---

### `POST /api/sessions/revoke-all`

**Why:** "Log out everywhere else" — revokes every other web session and every mobile session, keeping the browser tab that made the request signed in.

**Auth:** session; self-only by construction (`userId: session.id` on both update queries). The current web session id is excluded via `getWebSessionIdFromToken()`.

**Request:** no body.

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/sessions/revoke-all"
```

**Response:**
```json
{ "success": true, "revokedCount": 3 }
```

---

### `POST /api/shop/checkout`

**Why:** Starts a Stripe Checkout session for a shop cart. Prices, stock and the shipping quote are re-verified server-side from the database rather than trusted from the client cart, and a pending `ShopOrder` is written before redirecting to Stripe so the webhook has something to finalize.

**Auth:** session (`getSessionWithBearer()` — accepts cookie or bearer, so both web and mobile can check out).

**Request:** JSON body — `items` (array of `{ productId: string, quantity: integer 1–100 }`, required, non-empty), `address` (`ShippingAddress`, required only if any item `isPhysical`), `quote` (`ShippingQuote`, required — must not be expired, i.e. `validUntil` in the future), `couponCode` (string, optional — validated as an active Stripe promotion code).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"items":[{"productId":"clp1...","quantity":1}],"quote":{"subtotalHuf":5000,"shippingFeeHuf":0,"vatFeeHuf":0,"service":null,"totalHuf":5000,"validUntil":1234567890000}}' \
  "$HSR_URL/api/shop/checkout"
```

**Response:**
```json
{ "url": "https://checkout.stripe.com/c/pay/cs_test_..." }
```
Errors: `401` unauthenticated; `400` invalid items/expired quote/missing address for physical items/insufficient stock/total below Stripe's 175 HUF minimum/invalid coupon; `409` `{ "error": "quote_stale", "newQuote": { ... } }` when a re-quote for physical items differs from the submitted quote by more than 100 HUF.

---

### `GET /api/shop/orders`

**Why:** A member's own order history, including the Apple Wallet pass serial for any `APPLE_PASS` line item so the buyer can re-download it if the delivery email's attachment was stripped.

**Auth:** session (`getSessionWithBearer()`).

**Request:** no parameters.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/shop/orders"
```

**Response:**
```json
[
  {
    "id": "clo1...",
    "userId": "clu9...",
    "status": "PAID",
    "totalHuf": 5000,
    "createdAt": "2026-09-10T12:00:00.000Z",
    "items": [
      {
        "id": "cli1...",
        "productId": "clp1...",
        "quantity": 1,
        "product": { "id": "clp1...", "nameHu": "HSR bögre", "nameEn": "HSR mug" },
        "walletPass": null
      }
    ]
  }
]
```

---

### `GET /api/shop/orders/[id]`

**Why:** A single order's detail view (line items with product images) for the order confirmation/detail page.

**Auth:** session; ownership check — `order.userId !== session.id` returns 404 (not 403, so a stranger's order id doesn't confirm existence).

**Request:** path param `id` (order id).

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/shop/orders/clo1..."
```

**Response:**
```json
{
  "id": "clo1...",
  "userId": "clu9...",
  "status": "PAID",
  "items": [
    {
      "id": "cli1...",
      "product": { "id": "clp1...", "nameHu": "HSR bögre", "images": [{ "url": "https://s3.hsrgames.hu/..." }] }
    }
  ]
}
```
`404` if the order doesn't exist or isn't owned by the caller.

---

### `GET /api/shop/passes/[serial]`

**Why:** Re-download an issued Apple Wallet membership pass. Two authorization paths exist because the pass has to reach a phone that may not be logged into the site: the token printed in the delivery email/QR code, or an ordinary session belonging to the pass's owner. The card is re-rendered fresh from the member's current record (rank, avatar) on every download, keeping the same serial so Wallet replaces rather than duplicates it.

**Auth:** owner-or-token — `token` query param compared with `timingSafeEqual()` against `pass.authToken`, OR a session/bearer whose `id` matches `pass.userId`.

**Request:** path param `serial` (wallet pass serial number); query param `token` (optional, the pass's auth token).

```bash
curl -sS "$HSR_URL/api/shop/passes/PASS-000123?token=abcdef123..." -o pass.pkpass
```

**Response:** binary `.pkpass` file (`Content-Type: application/vnd.apple.pkpass`, `Content-Disposition: attachment`). Errors: `404` pass not found or voided; `401` neither token nor session matches; `503` Apple Wallet signing isn't configured (`applePassConfigured()` false).

---

### `GET /api/shop/products/[id]`

**Why:** Public product detail page — includes computed `availableStock` for `TRACKED` stock-mode products (stock minus reservations from pending carts).

**Auth:** public.

**Request:** path param `id` (product id). Only returns `active: true` products.

```bash
curl -sS "$HSR_URL/api/shop/products/clp1..."
```

**Response:**
```json
{
  "id": "clp1...",
  "nameHu": "HSR bögre",
  "priceHuf": 5000,
  "isPhysical": true,
  "stockMode": "TRACKED",
  "stock": 20,
  "reservedStock": 3,
  "availableStock": 17,
  "images": [{ "url": "https://s3.hsrgames.hu/..." }]
}
```
`404` if not found or inactive.

---

### `GET /api/shop/products`

**Why:** The shop's public product listing. `reservedStock` and `fulfillmentData` are stripped from the response — reservation counts and fulfillment payloads (which can carry internal delivery details, e.g. an `APPLE_PASS` config) are not meant for shoppers.

**Auth:** public.

**Request:** no parameters.

```bash
curl -sS "$HSR_URL/api/shop/products"
```

**Response:** array of active products with `availableStock` computed, `reservedStock`/`fulfillmentData` omitted.
```json
[
  { "id": "clp1...", "nameHu": "HSR bögre", "priceHuf": 5000, "stockMode": "TRACKED", "stock": 20, "availableStock": 17, "images": [] }
]
```

---

### `POST /api/shop/quote`

**Why:** Computes a shipping quote (fee + VAT) for a cart and address before checkout, so the checkout screen can show a real total; re-run at checkout time to catch a stale quote.

**Auth:** session; rate limited via `checkRateLimit(createLimiter, "shop-quote:<userId>")`.

**Request:** JSON body — `items` (array of `{ productId: string, quantity: positive integer }`, required, non-empty), `address` (`ShippingAddress`, optional — if provided, `name`, `phone`, `line1`, `city`, `postalCode`, `country` are all required).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"items":[{"productId":"clp1...","quantity":1}],"address":{"name":"Kovács János","phone":"+36301234567","line1":"Fő utca 1","city":"Budapest","postalCode":"1011","country":"HU"}}' \
  "$HSR_URL/api/shop/quote"
```

**Response:**
```json
{ "subtotalHuf": 5000, "shippingFeeHuf": 1500, "vatFeeHuf": 0, "service": "MPL", "totalHuf": 6500, "validUntil": 1757760000000 }
```
`400` on invalid items/missing address fields, or the underlying quoting error message; `429` if rate limited.

---

### `GET /api/shop/settings`

**Why:** Tells the storefront whether the shop is enabled at all, so the UI can hide the shop entirely when it's off (default is enabled if no settings row exists yet).

**Auth:** public.

**Request:** no parameters.

```bash
curl -sS "$HSR_URL/api/shop/settings"
```

**Response:**
```json
{ "shopEnabled": true }
```

---

### `GET /api/shop/verify`

**Why:** Called from the shop's success page after returning from Stripe Checkout, to finalize the order (mark it paid, trigger fulfillment) as a fallback/immediate path alongside the webhook — `finalizeShopOrderByCheckoutId()` is idempotent, so this and the webhook racing each other is safe.

**Auth:** session (`getSessionWithBearer()`) — note the route does not check that the checkout session's `userId` metadata matches the caller before finalizing (unlike `/api/stripe/verify`'s donation flow).

**Request:** query param — `session_id` (Stripe Checkout session id, required).

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/shop/verify?session_id=cs_test_..."
```

**Response:** one of:
```json
{ "status": "completed" }
```
```json
{ "status": "already_completed" }
```
```json
{ "status": "pending" }
```
`400` missing `session_id`; `404` order not found.

---

### `POST /api/stripe/checkout`

**Why:** The support-donation checkout flow (distinct from the shop). This route file is a straight re-export (`export { POST } from "@/app/api/polar/checkout/route"`) — the implementation lives at `src/app/api/polar/checkout/route.ts` (a leftover name from a prior payment provider), and creates a Stripe Checkout session plus a pending `Donation` row.

**Auth:** session (`getSessionWithBearer()`).

**Request:** JSON body — `amount` (number, HUF, required, minimum 175).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"amount":2000}' \
  "$HSR_URL/api/stripe/checkout"
```

**Response:**
```json
{ "url": "https://checkout.stripe.com/c/pay/cs_test_..." }
```
`400` amount below minimum or not a number; `401` unauthenticated.

---

### `POST /api/stripe/verify`

**Why:** Finalizes a donation after returning from Stripe Checkout — the polling counterpart to the webhook for the support/donation flow (separate from `/api/shop/verify`).

**Auth:** session; ownership is explicitly checked against the Stripe session's own `metadata.userId` (`ownerUserId !== auth.id` → 403) — stricter than the shop's verify route.

**Request:** JSON body — `sessionId` (string, required, the Stripe Checkout session id).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"sessionId":"cs_test_..."}' \
  "$HSR_URL/api/stripe/verify"
```

**Response:** on `payment_status !== "paid"`:
```json
{ "status": "pending" }
```
On success, whatever `finalizeDonationByCheckoutId()` returns (donation confirmation payload). `400` missing/invalid `sessionId` or checkout session not in `payment` mode; `403` the checkout session's `userId` metadata doesn't match the caller; `401` unauthenticated.

---

### `POST /api/stripe/webhook`

**Why:** The authoritative payment confirmation path — Stripe calls this directly, independent of whether the buyer's browser ever reaches the success page. Re-exported from `src/app/api/polar/webhook/route.ts`. Routes to shop order finalization or donation finalization based on the checkout session's `metadata.type`.

**Auth:** Stripe webhook signature — verified via `stripe.webhooks.constructEvent(rawBody, signature, STRIPE_WEBHOOK_SECRET)` against the raw request body and the `stripe-signature` header. **It never fails open**: a missing signature or secret returns 400 immediately, and a signature that fails verification returns 400 without processing the event — there is no code path that trusts an unverified payload.

**Request:** raw Stripe event body (JSON), header `stripe-signature` (required). Only `checkout.session.completed` events are acted on; all others are acknowledged and ignored.

```bash
# Sent by Stripe, not called directly. Local testing via the Stripe CLI:
stripe listen --forward-to "$HSR_URL/api/stripe/webhook"
```

**Response:** `{ "received": true }` on success (including for ignored event types). `400` missing/invalid signature; `500` if finalization throws. For `metadata.type === "shop"`, calls `finalizeShopOrderByCheckoutId(checkoutId)`; otherwise treats it as a donation and calls `finalizeDonationByCheckoutId(checkoutId, userId)` (skipped with a logged error if `checkoutId`/`userId` metadata is missing).

## Supervision, Supervisor Requests, Tickets, Training, Type Trainings, User, Verify, Wiki

### `GET /api/supervisor-request`

**Why:** Lists supervisor requests. Instructors/admins (`canManageSupervisionRequests()` or `role === "ADMIN"`) see every PENDING request across all trainees to claim from the shared pool; an ordinary trainee sees only their own requests, enriched with the linked duty's `startTime` and every `SupervisionFault` marked on those requests (including warnings) so the trainee sees exactly what they were briefed on.

**Auth:** Session (`requireAuth()`). Viewing everyone's queue additionally needs `canManageSupervisionRequests()` or `role === "ADMIN"`; anyone who can neither manage nor submit requests gets 403.

**Request:** No parameters.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/supervisor-request"
```

**Response (own requests):**
```json
[
  {
    "id": "clx...",
    "userId": "clu...",
    "track": "DRIVER",
    "status": "PENDING",
    "availableFrom": "2026-09-14T06:00:00.000Z",
    "availableTo": null,
    "note": null,
    "dutyId": null,
    "duty": null,
    "faults": []
  }
]
```
**Response (manager view):** array of `SupervisorRequest` rows with status `"PENDING"` and an included `user` (`id`, `displayName`, `name`, `discordRole`). Errors: `403` not permitted, `500` on failure.

### `PUT /api/supervisor-request`

**Why:** Lets an instructor/admin resolve a pending request without starting a session (e.g. cancel or mark completed some other way).

**Auth:** Session; `role === "ADMIN"` or `canManageSupervisionRequests()`, else 403.

**Request:** JSON body — `requestId` (string, required), `status` (`"COMPLETED" | "CANCELLED"`, required), `responseNote` (string, optional).

```bash
curl -sS -X PUT -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"requestId":"clx...","status":"CANCELLED","responseNote":"Trainee unavailable"}' \
  "$HSR_URL/api/supervisor-request"
```

**Response:** the updated `SupervisorRequest` row. Errors: `400` missing/invalid fields, `403` not permitted, `500` on failure.

### `DELETE /api/supervisor-request`

**Why:** Removes a request — a trainee may withdraw their own still-PENDING request, and a manager may delete any request; deleting an IN_PROGRESS one also closes its linked active duty first.

**Auth:** Session. Own PENDING request: the owner; any request: `canManageSupervisionRequests()` or `role === "ADMIN"`.

**Request:** Query param `id` (string, required — the request id).

```bash
curl -sS -X DELETE -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/supervisor-request?id=clx..."
```

**Response:** `{ "success": true }`. Errors: `400` missing id, `403` not the owner and not a manager, `404` not found, `500` on failure.

### `POST /api/supervisor-request`

**Why:** A trainee opens a supervision request into the shared instructor pool (no named instructor). Filed under the caller's own `supervisionTrack()`, falling back to `"DRIVER"` for an admin testing the flow (admins hold no trainee branch).

**Auth:** Session; requires `permissions.canSubmitSupervisionRequest()` (the `SUBMIT_SUPERVISION` flag), else 403. The resolved track must also be open in `getTrackConfig()`, else 403.

**Request:** JSON body — `availableFrom` (string, `YYYY-MM-DDTHH:mm[:ss]`, Budapest local time, required), `availableTo` (same format, optional), `note` (string, optional). Refused with 400 if the caller already has a PENDING request.

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"availableFrom":"2026-09-14T16:00:00","note":"Free after 4pm"}' \
  "$HSR_URL/api/supervisor-request"
```

**Response (201):** the created `SupervisorRequest` row. Errors: `400` missing/invalid dates or already has a pending request, `403` no `SUBMIT_SUPERVISION` or branch closed, `500` on failure.

### `GET /api/training`

**Why:** The trainee's own training-material page: modules/sections/blocks, progress and remaining deadline/hour budget for their (or a chosen) programme. Per the Automatic Content Translation section, block text is stripped out unless a live `TrainingAccessSession` is open, so browsing the navigation shell never leaks the course, and the returned content is machine-translated into the reader's locale (never re-translating text already in that language).

**Auth:** Session (`requireAuth()`). The resolved branch must be one this member is actually enrolled/eligible for (`resolveBranch()`), else 403.

**Request:** Query param `branch` (string, optional — a programme key; falls back to the caller's own trainee branch if omitted/invalid). Reader locale resolved via cookie/`?locale=`/`User.locale` per the translation section.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/training?branch=DRIVER&locale=en"
```

**Response:**
```json
{
  "branch": "DRIVER",
  "branchName": "Train Driver",
  "availableBranches": ["DRIVER"],
  "programs": [{ "key": "DRIVER", "name": "Train Driver" }],
  "modules": [
    {
      "id": "clm...",
      "branch": "DRIVER",
      "sections": [{ "id": "cls...", "blocks": [] }]
    }
  ],
  "translation": { "sourceLang": "hu" },
  "progress": { "completedSectionIds": ["cls..."], "isCompleted": false, "totalMinutesUsed": 12, "rulesAccepted": true, "allSectionsCompleted": false },
  "status": { "daysLeft": 18, "minutesLeft": 2988, "isDeadlinePassed": false, "isHoursExhausted": false, "isAccessAllowed": true },
  "activeSession": null
}
```
`blocks` are populated only while `activeSession` is non-null; `translation` is `null` when the content already matched the reader's language. Error: `403` not an active trainee for any branch.

### `GET /api/type-trainings`

**Why:** Lists open locomotive-type training sessions a driver may sign up for — locomotive-type knowledge is a driving concern, so it is gated on the drive permission rather than open to everyone.

**Auth:** Session; requires `permissions.canDrive()` (`CAN_DRIVE`), else the `requirePermission` 403.

**Request:** No parameters.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/type-trainings"
```

**Response:**
```json
[
  { "id": "clt...", "title": "V63 type training", "scheduledAt": "2026-09-20T17:00:00.000Z", "registrantCount": 4, "isRegistered": false }
]
```

### `GET /api/supervision/my-summary`

**Why:** The trainee's own recurring-fault breakdown, grouped by fault code across every `TRAINEE`-kind supervision. Deliberately takes no `userId` — per the Trainee Supervision Summary section, reading somebody else's marked faults is the Head of Education's cross-trainee report behind `MANAGE_INSTRUCTORS`, and an id parameter here would be a second, ungated path to the same data.

**Auth:** Session (`requireAuth()`). No further permission — always the caller's own supervisions.

**Request:** No parameters.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/supervision/my-summary"
```

**Response:** built by `buildTraineeSupervisionSummary()` — an object grouping faults by `code` with counts, whether each is "recurring" (marked across multiple sessions) or "improved" (absent from the most recent third of closed sessions), and warning-only occurrences. Renders as nothing on the dashboard when there is no closed supervision at all. Error: `500` on failure to build the summary.

### `GET /api/supervision/rules`

**Why:** Serves the trainee the same fault catalogue and thresholds the supervisor's sheet marks from — a printed hand-out would go stale the moment the Head of Education re-prices an item, per the Supervision Scoring section.

**Auth:** Session (`requireAuth()`). No extra permission — read-only, active entries only.

**Request:** Query param `track` (`"DRIVER" | "DISPATCHER"`, optional — otherwise both tracks' fault types are returned).

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/supervision/rules?track=DRIVER"
```

**Response:**
```json
{
  "faultTypes": [{ "id": "clf...", "code": "SIGNAL_MISS", "label": "Missed a signal", "kind": "INSTANT_FAIL", "points": 0, "track": "DRIVER" }],
  "rules": { "enabled": true, "failPointThreshold": 6, "requiredLaps": 2, "pointsFromLap": 2, "instantFailFirstLapWarning": true }
}
```

### `POST /api/supervisor-request/start`

**Why:** An instructor claims a PENDING request and opens the trainee's supervision duty. The duty's type is derived from the programme's own trainee team (`programDutyType()`) rather than being hardcoded to `TRAIN_DRIVER`, so a dispatcher trainee's supervision correctly opens on the dispatcher team.

**Auth:** Session; `role === "ADMIN"` or `canManageSupervisionRequests()`, else 403.

**Request:** JSON body — `requestId` (string, required).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"requestId":"clx..."}' \
  "$HSR_URL/api/supervisor-request/start"
```

**Response:**
```json
{ "success": true, "dutyId": "cld...", "dutyType": "TRAIN_DRIVER", "trackNumber": "SUP-482913" }
```
Errors: `400` missing requestId or request not PENDING, `403` not permitted, `404` request not found, `409` trainee already has an active duty, `500` on failure.

### `GET /api/supervisor-request/eligibility`

**Why:** A small boolean-shaped endpoint split out of `GET /api/supervisor-request`, which answers with a bare array both web and mobile clients index into directly — this tells the dashboard whether the "request supervision" button should even be shown, and whether the branch itself is currently open.

**Auth:** Session (`requireAuth()`).

**Request:** No parameters.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/supervisor-request/eligibility"
```

**Response:**
```json
{ "track": "DRIVER", "eligible": true, "trackOpen": true, "canSubmit": true, "tracks": { "DRIVER": true, "DISPATCHER": true } }
```

### `POST /api/supervisor-request/end`

**Why:** Closes a running supervision. With the rulebook enabled, the outcome is derived from the marked `SupervisionFault` rows rather than declared by the closer — the `passed` field sent by the instructor is a confirmation of the sheet, and disagreeing with it requires `MANAGE_INSTRUCTORS` plus a written `responseNote` (per the Supervision Scoring section). A pass that hits the programme's `requiredSupervisions` count promotes the trainee (Discord role swap + email); an `OBSERVATION` session is always closed as a report, never pass/fail.

**Auth:** Session; `role === "ADMIN"` or `canManageSupervisionRequests()`, else 403. Overriding the sheet's own verdict additionally needs `canManageInstructors()`.

**Request:** JSON body — `requestId` (string, required), `passed` (boolean, required unless the session `kind` is `OBSERVATION`), `responseNote` (string, required only when overriding the sheet's derived verdict).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"requestId":"clx...","passed":true}' \
  "$HSR_URL/api/supervisor-request/end"
```

**Response:**
```json
{ "success": true, "outcomeRule": null, "outcomeReason": null, "faultPoints": 0 }
```
Errors: `400` missing requestId/passed, request not IN_PROGRESS, undocumented laps, or a disagreement with the sheet without the required override permission/reason; `403` not permitted; `404` not found; `500` on failure.

### `GET /api/supervisor-request/evaluation`

**Why:** The live evaluation sheet for one running supervision — the fault catalogue to mark from plus everything already marked and what the rulebook currently makes of it (pass/fail projection, undocumented laps).

**Auth:** Session; requires `canManageSupervisionRequests()`.

**Request:** Query param `requestId` (string, required).

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/supervisor-request/evaluation?requestId=clx..."
```

**Response:**
```json
{
  "state": { "requestId": "clx...", "track": "DRIVER", "faults": [], "undocumentedLaps": [], "outcome": { "failed": false, "points": 0, "rule": null, "detail": null, "threshold": 6 } },
  "faultTypes": [{ "id": "clf...", "code": "SIGNAL_MISS", "label": "Missed a signal" }]
}
```
Errors: `400` missing requestId, `403` not permitted, `404` supervision not found.

### `GET /api/user/confirm-account`

**Why:** Tells a member logging in via Roblox whether they need to confirm ownership of an account the game server auto-created before they ever signed up — a fresh account with no game history is silently marked confirmed instead of asking.

**Auth:** Session (`getSession()`).

**Request:** No parameters.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/user/confirm-account"
```

**Response (needs confirmation):**
```json
{
  "needsConfirmation": true,
  "account": { "id": "clu...", "robloxId": "12345", "name": "Kovacs123", "displayName": null, "image": "https://tr.rbxcdn.com/...", "totalPlayTime": 3600, "totalSessions": 5, "createdAt": "2026-01-02T10:00:00.000Z", "discordLinked": false }
}
```
Or `{ "needsConfirmation": false }`. Errors: `401` unauthenticated, `404` user not found, `500` on failure.

### `POST /api/user/confirm-account`

**Why:** Records the member's answer to the confirmation prompt: `"confirm"` accepts the auto-created game history as their own; `"reject"` keeps the account (Roblox OAuth already proved the `robloxId` is theirs) but zeroes the stats, since they don't want that play history attributed to them.

**Auth:** Session (`getSession()`).

**Request:** JSON body — `action` (`"confirm" | "reject"`, required).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"action":"confirm"}' \
  "$HSR_URL/api/user/confirm-account"
```

**Response:** `{ "success": true }`. Errors: `400` invalid/missing action, `401` unauthenticated, `500` on failure.

### `GET /api/user/me`

**Why:** The caller's own core account fields, for both web (cookie session) and mobile (bearer token) clients.

**Auth:** Session or bearer (`getSessionWithBearer()`).

**Request:** No parameters.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/user/me"
```

**Response:**
```json
{
  "id": "clu...", "robloxId": "12345", "name": "Kovacs123", "displayName": "Kovács János",
  "image": "https://tr.rbxcdn.com/...", "steamId": null, "discordId": "9876543210",
  "discordUsername": "kovacsj", "discordAvatar": "abcd1234", "discordRole": "Train driver",
  "role": "USER", "banned": false, "createdAt": "2025-11-01T09:00:00.000Z",
  "lastLoginAt": "2026-09-13T08:00:00.000Z", "locale": "hu"
}
```
Errors: `401` unauthenticated, `404` user not found, `500` on failure.

### `PATCH /api/user/me`

**Why:** Lets the member edit the two fields the site itself owns for them — their display name and their language preference. Roblox/Discord-sourced fields are never editable here.

**Auth:** Session or bearer (`getSessionWithBearer()`).

**Request:** JSON body — `displayName` (string, 1-32 chars, optional), `locale` (must be one of `SUPPORTED_LOCALES`, optional). At least one must be present.

```bash
curl -sS -X PATCH -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"locale":"en"}' \
  "$HSR_URL/api/user/me"
```

**Response:** `{ "success": true, "user": { "id": "clu...", "displayName": "...", "locale": "en", ... } }`. Errors: `400` invalid displayName length/locale or no valid fields, `401` unauthenticated, `500` on failure.

### `GET /api/user/[userId]`

**Why:** A minimal public-ish lookup of another user's basic identity (name, rank, avatar) — used where a UI needs to display who a duty/schedule row belongs to.

**Auth:** Session or bearer (`getSessionWithBearer()`) — any authenticated caller, no ownership or permission check on the target `userId`.

**Request:** Path param `userId` (string, required).

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/user/clu1234567890"
```

**Response:**
```json
{ "id": "clu1234567890", "robloxId": "12345", "name": "Kovacs123", "displayName": "Kovács János", "image": "https://tr.rbxcdn.com/...", "discordRole": "Train driver", "role": "USER" }
```
Errors: `401` unauthenticated, `404` user not found, `500` on failure.

### `GET /api/verify/[code]`

**Why:** Public verification of a signed document by its verify code — metadata only (validity, title, who signed, when); the document body never leaves the hidden authenticated surface, so a leaked code discloses nothing about the content.

**Auth:** Public, no session. Rate limited per IP (`apiLimiter`).

**Request:** Path param `code` (string, required — URL-encoded verify code, normalized via `normalizeVerifyCode()`).

```bash
curl -sS "$HSR_URL/api/verify/AB12-CD34-EF56"
```

**Response:**
```json
{
  "found": true, "verifyCode": "AB12-CD34-EF56", "title": "Driver Certification 2026",
  "valid": true, "status": "SIGNED", "signedAt": "2026-03-01T12:00:00.000Z",
  "signerName": "Nagy Péter", "signerTitle": "Director General",
  "signatures": [{ "name": "Nagy Péter", "title": "Director General", "signedAt": "2026-03-01T12:00:00.000Z" }],
  "authorName": "Kis Anna", "authorRole": "HR", "pdfSha256": "9f8e...",
  "revokedAt": null, "revokedByName": null, "revokeReason": null
}
```
A draft/pending document, an unrecognized code format, or no match all answer `404` with `{ "found": false, "reason": "NOT_FOUND" | "INVALID_FORMAT" }`.

### `GET /api/wiki/articles`

**Why:** Public list of every published wiki page (pinned first, then alphabetical), with its category — the plain data feed behind the wiki index; the reader-facing translation described in the Automatic Content Translation section is applied on the `/wiki` page route itself, not here.

**Auth:** Public — no session check.

**Request:** No parameters.

```bash
curl -sS "$HSR_URL/api/wiki/articles"
```

**Response:** array of `WikiPage` rows with `category` included, e.g. `[{ "id": "clw...", "slug": "brake-test", "title": "Brake test", "isPublished": true, "isPinned": false, "category": { "id": "clc...", "name": "Operations", "slug": "operations" } }]`. Error: `500` on failure.

### `POST /api/wiki/articles`

**Why:** Creates a new wiki page. Despite living under `/api/wiki`, this is an admin-only write, gated the same way `/admin/wiki` gates its own mutations.

**Auth:** Session; requires `MANAGE_WIKI` flag (`requirePermission((p) => p.hasFlag("MANAGE_WIKI"))`).

**Request:** JSON body — `slug` (string, required, must be unique), `title` (string, required), `content` (string, optional, defaults to `""`), `excerpt` (string, optional), `categoryId` (string, optional), `order` (number, optional, defaults to `0`), `isPublished` (boolean, optional, defaults to `true`), `isPinned` (boolean, optional, defaults to `false`).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"slug":"new-rule","title":"New rule","content":"# Text"}' \
  "$HSR_URL/api/wiki/articles"
```

**Response:** the created `WikiPage` row. Errors: `400` slug already exists, `403` missing `MANAGE_WIKI`, `500` on failure.

### `GET /api/wiki/categories`

**Why:** Public list of wiki categories, each with its published pages, for building the wiki's sidebar navigation tree.

**Auth:** Public — no session check.

**Request:** No parameters.

```bash
curl -sS "$HSR_URL/api/wiki/categories"
```

**Response:** array of `WikiCategory` rows with `pages` included, e.g. `[{ "id": "clc...", "name": "Operations", "slug": "operations", "order": 0, "pages": [{ "id": "clw...", "title": "Brake test", "slug": "brake-test" }] }]`. Error: `500` on failure.

### `POST /api/wiki/categories`

**Why:** Creates a wiki category. Admin-only write under a member-facing path.

**Auth:** Session; requires `MANAGE_WIKI`.

**Request:** JSON body — `name` (string, required), `slug` (string, required, unique), `icon` (string, optional), `description` (string, optional), `order` (number, optional, defaults `0`).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"Operations","slug":"operations"}' \
  "$HSR_URL/api/wiki/categories"
```

**Response:** the created `WikiCategory` row. Errors: `400` slug already exists, `403` missing `MANAGE_WIKI`, `500` on failure.

### `PUT /api/wiki/categories`

**Why:** Renames/reorders an existing category.

**Auth:** Session; requires `MANAGE_WIKI`.

**Request:** JSON body — `id` (string, required), `name` (string, required), `slug` (string, required, unique excluding this row), `icon` (string, optional), `description` (string, optional), `order` (number, optional, defaults `0`).

```bash
curl -sS -X PUT -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"id":"clc...","name":"Operations","slug":"operations","order":1}' \
  "$HSR_URL/api/wiki/categories"
```

**Response:** the updated `WikiCategory` row. Errors: `400` slug taken by another category, `403` missing `MANAGE_WIKI`, `500` on failure.

### `DELETE /api/wiki/categories`

**Why:** Deletes a category and every wiki page filed under it.

**Auth:** Session; requires `MANAGE_WIKI`.

**Request:** JSON body — `id` (string, required).

```bash
curl -sS -X DELETE -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"id":"clc..."}' \
  "$HSR_URL/api/wiki/categories"
```

**Response:** `{ "success": true }`. Errors: `403` missing `MANAGE_WIKI`, `500` on failure.

### `GET /api/wiki/articles/[id]`

**Why:** Fetches one wiki page by id and increments its view counter — the reader-facing article view. Machine translation of the body (per the Automatic Content Translation section) is applied by the `/wiki/[slug]` page route, not returned from this raw API.

**Auth:** Public — no session check.

**Request:** Path param `id` (string, required).

```bash
curl -sS "$HSR_URL/api/wiki/articles/clw1234567890"
```

**Response:** the `WikiPage` row with `category` included. Errors: `404` not found, `500` on failure.

### `PATCH /api/wiki/articles/[id]`

**Why:** Edits a wiki page. Only an explicit allow-list of fields is copied from the body into the Prisma update — the old code passed the raw body straight through, which made every column (including `views` and `createdAt`) writable by whoever could reach this route.

**Auth:** Session; requires `MANAGE_WIKI`.

**Request:** Path param `id` (string, required). JSON body — any of `title` (string), `slug` (string), `content` (string), `excerpt` (string or `null`), `order` (number), `isPublished` (boolean), `isPinned` (boolean), `categoryId` (string) — all optional, only present fields are updated.

```bash
curl -sS -X PATCH -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"isPublished":true,"order":2}' \
  "$HSR_URL/api/wiki/articles/clw1234567890"
```

**Response:** the updated `WikiPage` row. Errors: `403` missing `MANAGE_WIKI`, `500` on failure (including a bad `categoryId`).

### `DELETE /api/wiki/articles/[id]`

**Why:** Removes a wiki page.

**Auth:** Session; requires `MANAGE_WIKI`.

**Request:** Path param `id` (string, required).

```bash
curl -sS -X DELETE -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/wiki/articles/clw1234567890"
```

**Response:** `{ "success": true }`. Errors: `403` missing `MANAGE_WIKI`, `500` on failure.

### `GET /api/training/section`

**Why:** Fetches one section's blocks, translated into the reader's language, for the trainee currently viewing it. `/api/training` deliberately never translates block text live for the whole course (it would produce a course half in each language before the cache warms); this route handles the one section actually being read, which is small enough to finish inside its own (longer, 25s default) translation budget. Same access-session gate as `/api/training`, so this cannot be used to pull the whole course section by section without an open session.

**Auth:** Session (`requireAuth()`). The section's branch must resolve to one the caller may access (`resolveBranch()`), and a live `TrainingAccessSession` for that branch must be open, else 403.

**Request:** Query param `id` (string, required — the section id). `?locale=` supported via `requestLocale()`.

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/training/section?id=cls1234567890&locale=en"
```

**Response:**
```json
{ "sectionId": "cls1234567890", "sourceLang": "hu", "blocks": [{ "id": "clb...", "text": "Translated section text..." }] }
```
`sourceLang` is `null` when no translation was needed. Errors: `400` missing id, `403` wrong branch / no active session, `404` section not found.

### `POST /api/training/progress`

**Why:** Marks one section as completed for the caller's own progress in a branch, rolling up whether the whole module is done (a completion-gate section, or every section in the branch completed).

**Auth:** Session (`requireAuth()`). Branch must resolve for the caller.

**Request:** JSON body — `sectionId` (string, required), `branch` (string, optional — resolved/validated against the caller's programmes).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"sectionId":"cls1234567890","branch":"DRIVER"}' \
  "$HSR_URL/api/training/progress"
```

**Response:** `{ "ok": true, "isCompleted": false, "allSectionsCompleted": false }` (or `{ "ok": true, "alreadyCompleted": true }` if already marked). Errors: `400` missing sectionId or section belongs to another branch, `403` branch not accessible, `404` section not found.

### `POST /api/training/access`

**Why:** Opens a timed (45-minute) training-material access session, which is what unlocks block text from `GET /api/training`/`/api/training/section`. Also enforces the programme's day/hour limits and closes any other open session first (a member can only sit in front of one material at a time), crediting the elapsed minutes to that other branch's progress before starting the new one.

**Auth:** Session (`requireAuth()`). Branch must resolve for the caller.

**Request:** JSON body — `branch` (string, optional — resolved against the caller's programmes).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"branch":"DRIVER"}' \
  "$HSR_URL/api/training/access"
```

**Response:** `{ "token": "b3f1...", "expiresAt": "2026-09-13T09:45:00.000Z" }`. Errors: `403` branch not accessible, training deadline expired, or hour budget exhausted.

### `POST /api/training/heartbeat`

**Why:** Confirms an access-session token is still valid, so the client's training-material page knows whether to keep the material unlocked or force a re-open.

**Auth:** Session (`requireAuth()`). Caller must have at least one accessible branch.

**Request:** JSON body — `token` (string, required — the session token from `/api/training/access`).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"token":"b3f1..."}' \
  "$HSR_URL/api/training/heartbeat"
```

**Response:** `{ "valid": true, "expiresAt": "2026-09-13T09:45:00.000Z" }`, or `401 { "valid": false }` if the token is missing/expired/not the caller's. Error: `403` if no branch is accessible at all.

### `POST /api/training/rules-accept`

**Why:** Records that the trainee has acknowledged the training rules for a branch, gating further progress the same way the material's other checkpoints do.

**Auth:** Session (`requireAuth()`). Branch must resolve for the caller.

**Request:** JSON body — `branch` (string, optional — resolved against the caller's programmes).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"branch":"DRIVER"}' \
  "$HSR_URL/api/training/rules-accept"
```

**Response:** `{ "ok": true }`. Error: `403` branch not accessible.

### `POST /api/training/access/end`

**Why:** Closes a training access session and writes back the minutes consumed against the trainee's hour budget. Deliberately checks only token ownership, not the trainee's current role — a trainee who loses the role mid-session must still be able to close it, or the consumed minutes are never credited.

**Auth:** Session (`requireAuth()`) — ownership of the session token, no permission/branch gate.

**Request:** JSON body — `token` (string, required).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"token":"b3f1..."}' \
  "$HSR_URL/api/training/access/end"
```

**Response:** `{ "ok": true, "minutesConsumed": 12 }`, or `{ "ok": true }` (idempotent) if no matching open session exists.

### `POST /api/supervisor-request/evaluation/lap`

**Why:** Documents one completed lap of a running supervision — required before the session can be closed, per the Supervision Scoring section's "documentation is a gate, not a report" rule.

**Auth:** Session; requires `canManageSupervisionRequests()`.

**Request:** JSON body — `requestId` (string, required), `index` (number, required — the lap index), `note` (string, optional).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"requestId":"clx...","index":0,"note":"Clean lap"}' \
  "$HSR_URL/api/supervisor-request/evaluation/lap"
```

**Response:** `{ "state": { ...evaluation state... } }`. Errors: `400` missing requestId/index or the underlying rule rejects it, `403` not permitted.

### `DELETE /api/supervisor-request/evaluation/lap`

**Why:** Undoes a documented lap.

**Auth:** Session; requires `canManageSupervisionRequests()`.

**Request:** Query params `requestId` (string, required), `index` (number, required).

```bash
curl -sS -X DELETE -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/supervisor-request/evaluation/lap?requestId=clx...&index=0"
```

**Response:** `{ "state": { ... } }`. Errors: `400` missing params or rejected, `403` not permitted.

### `POST /api/supervisor-request/evaluation/fault`

**Why:** Marks a fault against the trainee during a running supervision, from the shared catalogue.

**Auth:** Session; requires `canManageSupervisionRequests()`.

**Request:** JSON body — `requestId` (string, required), `faultTypeId` (string, required), `lap` (number, required), `points` (number, optional — override the catalogue's own point cost), `note` (string, optional).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"requestId":"clx...","faultTypeId":"clf...","lap":0}' \
  "$HSR_URL/api/supervisor-request/evaluation/fault"
```

**Response:** `{ "state": { ... } }`. Errors: `400` missing fields or rejected, `403` not permitted.

### `DELETE /api/supervisor-request/evaluation/fault`

**Why:** Undoes a marked fault — a mis-click during a live supervision is not a verdict.

**Auth:** Session; requires `canManageSupervisionRequests()`.

**Request:** Query param `id` (string, required — the fault id).

```bash
curl -sS -X DELETE -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/supervisor-request/evaluation/fault?id=clfault..."
```

**Response:** `{ "state": { ... } }`. Errors: `400` missing id or rejected, `403` not permitted.

### `GET /api/tickets/mine/[id]`

**Why:** Reads one of the caller's own Discord-bridged support tickets with its full message thread. A ticket that is not the caller's answers `404` rather than `403` — since the id is a cuid, confirming existence at all would leak that somebody else owns a ticket by that id.

**Auth:** Session or bearer (`getSessionWithBearer()`), plus ownership (`ownsTicket()`).

**Request:** Path param `id` (string, required).

```bash
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/tickets/mine/clt1234567890"
```

**Response:**
```json
{
  "ticket": {
    "id": "clt1234567890", "status": "CLOSED", "categoryLabel": "General",
    "createdAt": "2026-09-01T10:00:00.000Z", "closedAt": "2026-09-01T12:00:00.000Z",
    "closedByUsername": "staffuser", "claimedByName": "staffuser", "channelId": "1122334455",
    "rating": { "stars": 5, "comment": "Thanks!", "createdAt": "2026-09-01T12:05:00.000Z" },
    "messages": [{ "id": "clm...", "discordUsername": "member", "isStaff": false, "content": "Hi", "attachments": [], "createdAt": "2026-09-01T10:00:00.000Z" }]
  }
}
```
Errors: `401` unauthenticated, `404` not found or not owned by caller.

### `POST /api/type-trainings/[id]/register`

**Why:** Signs the caller up for a scheduled locomotive-type training. Upserted so a repeat click is harmless.

**Auth:** Session; requires `permissions.canDrive()`.

**Request:** Path param `id` (string, required — the training id). No body.

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/type-trainings/clt1234567890/register"
```

**Response:** `{ "ok": true }`. Error: `400` training not found or not `SCHEDULED`.

### `DELETE /api/type-trainings/[id]/register`

**Why:** Withdraws the caller's own registration from a type training.

**Auth:** Session (`requireAuth()`) — no `CAN_DRIVE` gate on withdrawal, just ownership by construction (deletes only the caller's own registration row).

**Request:** Path param `id` (string, required).

```bash
curl -sS -X DELETE -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/type-trainings/clt1234567890/register"
```

**Response:** `{ "ok": true }` regardless of whether a registration existed.

### `POST /api/tickets/mine/[id]/rating`

**Why:** The web half of the ticket star rating — writes the same `DiscordTicketRating` row the Discord-side rating buttons write, so a member can rate wherever they happen to be and the rating board counts it exactly once either way. Only a `CLOSED` ticket may be rated.

**Auth:** Session or bearer (`getSessionWithBearer()`), plus ownership (`ownsTicket()`).

**Request:** Path param `id` (string, required). JSON body — `stars` (integer 1-5, required), `comment` (string, optional, trimmed to 1000 chars).

```bash
curl -sS -X POST -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"stars":5,"comment":"Quick and helpful"}' \
  "$HSR_URL/api/tickets/mine/clt1234567890/rating"
```

**Response:** `{ "rating": { "stars": 5, "comment": "Quick and helpful", "createdAt": "2026-09-13T09:00:00.000Z" } }`. Errors: `400` invalid stars or ticket not `CLOSED`, `401` unauthenticated, `404` not found or not owned by caller.

## Source of truth

The implementation for every endpoint is under `src/app/api/`. Shared authentication, authorization, validation, and business logic is under `src/lib/`. For complete mobile OAuth examples and response payloads, see [Mobile API](mobile-api.md).
