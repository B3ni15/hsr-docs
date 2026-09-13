# HSR Mobile API

This document describes the server endpoints used by an external mobile client. The standalone Expo app is no longer stored in this repository, but the backend mobile endpoints remain available.

## Setup

```bash
export HSR_URL="https://hsr.example.com"
export REDIRECT_URI="hsr://auth/callback"
export STATE="a-random-value-generated-by-the-client"
```

Production redirect URIs use `hsr://`. `exp://` is also accepted for development builds. Generate an unpredictable `state` value and verify it after the callback.

## OAuth login

### 1. Start authorization

Open this URL in a system browser:

```bash
curl -i -c cookies.txt -G "$HSR_URL/api/auth/mobile/authorize" \
  --data-urlencode "redirect_uri=$REDIRECT_URI" \
  --data-urlencode "state=$STATE"
```

On success the endpoint returns HTTP `307` and redirects to `/login?mobile=true`. It stores a short-lived `mobile-oauth` cookie.

Errors:

- `400` `{ "error": "redirect_uri is required" }`
- `400` `{ "error": "Invalid redirect_uri. Must use hsr:// or exp:// scheme" }`
- `400` `{ "error": "state is required" }`

After the user completes web login, call the complete endpoint with the same browser cookie.

### 2. Complete authorization

```bash
curl -sS -X POST \
  -b cookies.txt \
  "$HSR_URL/api/auth/mobile/complete"
```

Successful response:

```json
{
  "redirectUrl": "hsr://auth/callback?code=<one-time-code>&state=<state>"
}
```

The authorization code expires after 5 minutes and can be exchanged once only.

Errors:

- `401` `{ "error": "Not authenticated" }`
- `400` `{ "error": "No mobile OAuth context found" }`
- `400` `{ "error": "Mobile OAuth context expired" }`

Parse `code` and `state` from `redirectUrl`, verify `state`, then exchange the code.

### 3. Exchange the code for tokens

```bash
export AUTH_CODE="<code-from-redirectUrl>"

curl -sS -X POST "$HSR_URL/api/auth/mobile/token" \
  -H "Content-Type: application/json" \
  -d "{\"code\":\"$AUTH_CODE\",\"state\":\"$STATE\"}"
```

Successful response:

```json
{
  "access_token": "<jwt>",
  "refresh_token": "<opaque-refresh-token>",
  "token_type": "Bearer",
  "expires_in": 604800,
  "user": {
    "id": "user-id",
    "robloxId": "roblox-id",
    "name": "Roblox name",
    "displayName": "Display name",
    "image": "https://...",
    "discordId": "discord-id",
    "discordUsername": "username",
    "discordAvatar": "avatar-hash",
    "discordEmail": "member@example.com",
    "discordRole": "Member",
    "role": "USER",
    "banned": false,
    "sessionVersion": 1
  }
}
```

`expires_in` is 604800 seconds, or 7 days. The refresh session lasts 90 days unless revoked. Store both tokens in secure platform storage.

Errors include:

- `400` missing, invalid, expired, or already-used authorization code; or state mismatch
- `404` `{ "error": "User not found" }`
- `403` `{ "error": "User account is banned" }`
- `429` rate-limit response

## Refresh the access token

The refresh endpoint returns a new access token and the same refresh token.

```bash
export REFRESH_TOKEN="<refresh-token>"

curl -sS -X POST "$HSR_URL/api/auth/mobile/refresh" \
  -H "Content-Type: application/json" \
  -d "{\"refresh_token\":\"$REFRESH_TOKEN\"}"
```

Successful response:

```json
{
  "access_token": "<new-jwt>",
  "refresh_token": "<same-refresh-token>",
  "token_type": "Bearer",
  "expires_in": 604800,
  "user": { "id": "user-id", "banned": false }
}
```

Errors: `400` when the token is missing, `401` when it is invalid, revoked, or expired, and `403` for a banned account.

## Use the bearer token

```bash
curl -sS \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$HSR_URL/api/auth/session"
```

The session endpoint returns HTTP `200` with an object containing `user`, or HTTP `401` with `{ "user": null }`. The returned user includes current permission flags and role context.

Bearer auth is supported on `/api/auth/session`, `/api/duty`, `/api/profile`, `/api/admin`, `/api/schedule-request`, `/api/supervisor-request`, `/api/push/token`, `/api/mobile`, `/api/user`, `/api/sessions`, `/api/leaderboard`, `/api/news`, `/api/shop`, `/api/cargo`, `/api/dispatch-orders`, `/api/route-permit`, `/api/map`, and `/api/training`. Individual handlers may apply additional checks.

## Register an iOS Live Activity

```bash
curl -sS -X POST "$HSR_URL/api/mobile/live-activity/register" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "dutyId": "duty-id",
    "pushToken": "apns-push-token",
    "activityId": "live-activity-id"
  }'
```

Successful response:

```json
{ "success": true }
```

The endpoint updates an existing `dutyId` + `activityId` pair or creates it. Errors:

- `400` `{ "error": "dutyId, pushToken, and activityId are required" }`
- `401` `{ "error": "Invalid or expired token" }`
- `404` `{ "error": "Active duty not found" }`
- `500` `{ "error": "Failed to register Live Activity token" }`

## Revoke mobile sessions

### Revoke one refresh session

The refresh token must belong to the access-token user.

```bash
curl -sS -X POST "$HSR_URL/api/auth/mobile/revoke" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"refresh_token\":\"$REFRESH_TOKEN\"}"
```

Response:

```json
{ "success": true }
```

Possible errors include `400` when neither token is supplied, `401` for an invalid access token, and `403` when the refresh token belongs to another user.

### Revoke all mobile sessions

```bash
curl -sS -X POST "$HSR_URL/api/auth/mobile/revoke" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

The response is `{ "success": true }`; all active mobile sessions for that user are revoked.

## App review shortcut

When review variables are configured, this endpoint creates a short-lived authorization code for the configured review user:

```bash
curl -sS -X POST "$HSR_URL/api/auth/mobile/review" \
  -H "Content-Type: application/json" \
  -d '{
    "accessCode": "<review-access-code>",
    "redirectUri": "hsr://auth/callback"
  }'
```

Successful response:

```json
{ "code": "<one-time-code>" }
```

The endpoint is disabled and returns `404` unless review credentials are configured. An invalid access code returns `401`.

## Client rules

- Store access and refresh tokens in secure storage.
- Always validate OAuth `state`.
- Do not send refresh tokens to ordinary API routes.
- Treat `403` as an account or permission decision, not as an expired access token.
- Read the structured `error` value from failed responses.
- Use permission data from `/api/auth/session`; do not authorize features from the legacy `role` string alone.

## Full Endpoint Reference

Every mobile OAuth, Live Activity, and push-token route handler, individually, with its own request/response and `curl` example. The narrative walkthrough above covers the common login path; this section is the exhaustive reference, including registration/unregistration, refresh/revoke, and the review shortcut already shown above.

### `GET /api/auth/mobile/authorize`

**Why:** Entry point of the mobile OAuth-provider flow — the mobile app is a separate OAuth client of the web app rather than sharing its cookie session, so it needs its own authorize step that stashes the mobile-specific redirect intent before handing the user to the ordinary web login.

**Auth:** Public (no session required — this is the unauthenticated start of the flow).

**Request:** Query params — `redirect_uri` (required, string, must satisfy `isValidMobileRedirectUri()`: `hsr://` or `exp://` scheme), `state` (required, string, opaque value returned by the mobile client).

```bash
curl -sS "$HSR_URL/api/auth/mobile/authorize?redirect_uri=hsr://auth/callback&state=abc123"
```

**Response:** A 302 redirect to `/login?mobile=true`, with an httpOnly `mobile-oauth` cookie (10 min max-age) carrying `{ redirectUri, state, createdAt }`. Errors (400 JSON): `{ "error": "redirect_uri is required" }`, `{ "error": "Invalid redirect_uri. Must use hsr:// or exp:// scheme" }`, `{ "error": "state is required" }`.

### `POST /api/auth/mobile/complete`

**Why:** Called once the user has finished the ordinary web login (Roblox/Discord) with the `mobile-oauth` cookie still present — it turns an authenticated web session into a short-lived authorization code the mobile app can exchange, without ever exposing the web session's JWT to the app.

**Auth:** Requires an existing web session (httpOnly `auth-token` cookie, read via `getSession()`).

**Request:** No body. Reads the `mobile-oauth` cookie set by `authorize`.

```bash
curl -sS -X POST "$HSR_URL/api/auth/mobile/complete" -H "Cookie: auth-token=..."
```

**Response:**
```json
{ "redirectUrl": "hsr://auth/callback?code=<mobileAuthCode>&state=abc123" }
```
Also creates a `MobileAuthCode` row (5 min expiry), logs `MOBILE_LOGIN` to `ActivityLog`, and clears the `mobile-oauth` cookie. Errors (400): no session (401 `"Not authenticated"`), no/invalid/expired `mobile-oauth` cookie (`"No mobile OAuth context found"` / `"Invalid mobile OAuth context"` / `"Mobile OAuth context expired"`); 500 on unexpected failure.

### `POST /api/auth/mobile/token`

**Why:** The mobile app exchanges the one-time authorization code (from `complete`) for the actual credentials it will use going forward — a short-lived bearer access token plus a long-lived refresh token — completing the OAuth code-exchange half of the mobile provider flow.

**Auth:** Public endpoint, but gated by the single-use, expiring `MobileAuthCode` row and IP rate limiting (`authLimiter`, keyed `mobile-token:<ip>`).

**Request:** JSON body — `code` (required, string, the authorization code), `state` (optional, string; if present must match the code's stored state).

```bash
curl -sS -X POST "$HSR_URL/api/auth/mobile/token" \
  -H "Content-Type: application/json" \
  -d '{"code":"<authCode>","state":"abc123"}'
```

**Response:**
```json
{
  "access_token": "<jwt>",
  "refresh_token": "<opaque token>",
  "token_type": "Bearer",
  "expires_in": 604800,
  "user": { "id": "...", "robloxId": "...", "name": "...", "displayName": "...", "image": null, "discordId": "...", "discordUsername": "...", "discordAvatar": null, "discordEmail": null, "discordRole": "...", "role": "USER", "banned": false, "sessionVersion": 0 }
}
```
Creates a `MobileSession` row (90-day expiry, stores `deviceInfo` from the `user-agent` header) and logs `MOBILE_LOGIN`. Errors (400): missing `code`, invalid/expired/already-used code (atomically claimed via `updateMany`), state mismatch; 404 `"User not found"`; 403 `"User account is banned"`; 429 from the rate limiter; 500 on unexpected failure.

### `POST /api/auth/mobile/refresh`

**Why:** Lets the mobile app mint a new 7-day access token from its 90-day refresh token without forcing the user to log in again — the access/refresh split from the "Mobile Token Strategy" pattern.

**Auth:** Public endpoint; authorization is the refresh token itself (looked up as a `MobileSession.refreshToken`), plus IP rate limiting (`mobile-refresh:<ip>`).

**Request:** JSON body — `refresh_token` (required, string).

```bash
curl -sS -X POST "$HSR_URL/api/auth/mobile/refresh" \
  -H "Content-Type: application/json" \
  -d '{"refresh_token":"<refreshToken>"}'
```

**Response:**
```json
{
  "access_token": "<new jwt>",
  "refresh_token": "<same refreshToken>",
  "token_type": "Bearer",
  "expires_in": 604800,
  "user": { "id": "...", "robloxId": "...", "name": "...", "displayName": "...", "image": null, "discordId": "...", "discordUsername": "...", "discordAvatar": null, "discordEmail": null, "discordRole": "...", "role": "USER", "banned": false, "sessionVersion": 0 }
}
```
Also stamps `lastUsedAt` on the session. Errors (401): missing/unknown `refresh_token`, session `revokedAt` set (`"Session has been revoked"`), session expired (`"Refresh token has expired"`); 403 if the user is now banned (also revokes the session as a side effect); 400 missing `refresh_token`; 500 on unexpected failure.

### `POST /api/auth/mobile/revoke`

**Why:** Sign-out / "log out of this device" (or all devices) for the mobile app — invalidates refresh tokens server-side so a stolen or discarded device's session can be killed.

**Auth:** Bearer access token required in all paths. Revoking a specific `refresh_token` additionally requires that the token's `userId` matches the bearer token's subject (ownership check). Revoking "all sessions" is derived from the bearer token alone.

**Request:** JSON body — `refresh_token` (optional, string: revoke this one session) — and/or `Authorization: Bearer <access_token>` header (required either way; if only the header is given, all of that user's active `MobileSession` rows are revoked).

```bash
curl -sS -X POST "$HSR_URL/api/auth/mobile/revoke" \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"refresh_token":"<refreshToken>"}'
```

**Response:**
```json
{ "success": true }
```
Errors: 400 `"Refresh token or access token is required"` (neither provided); 401 `"Authorization required"` (refresh_token given but no bearer token), `"Invalid access token"` (bearer token fails `verifyToken`); 403 `"Forbidden"` (refresh token belongs to a different user); 500 on unexpected failure. A refresh token that doesn't exist or is already revoked still returns `{ "success": true }` (idempotent).

### `POST /api/auth/mobile/review`

**Why:** Apple App Store / TestFlight review needs a way into the app without a real Roblox/Discord OAuth round-trip, since reviewers cannot complete third-party logins reliably — this mints a mobile authorization code directly for a pre-configured review account.

**Auth:** Public endpoint, gated entirely by an out-of-band shared secret (`accessCode` matched against `REVIEW_ACCESS_CODE`) plus IP rate limiting (`review:<ip>`). The whole route 404s unless `REVIEW_ACCESS_CODE` and at least one of `REVIEW_USER_ID`/`REVIEW_ROBLOX_ID`/`REVIEW_DISCORD_ID` are set in the environment.

**Request:** JSON body — `accessCode` (required, string, must equal `REVIEW_ACCESS_CODE`), `redirectUri` (optional, string; if given must satisfy `isValidMobileRedirectUri()`, else defaults to `hsr://auth/callback`).

```bash
curl -sS -X POST "$HSR_URL/api/auth/mobile/review" \
  -H "Content-Type: application/json" \
  -d '{"accessCode":"<REVIEW_ACCESS_CODE>","redirectUri":"hsr://auth/callback"}'
```

**Response:**
```json
{ "code": "<mobileAuthCode>" }
```
The returned `code` is exchanged the normal way at `POST /api/auth/mobile/token` (state is fixed to `"review"`). Errors: 404 `"Not found"` (review mode not configured); 401 `"Unauthorized"` (bad/missing access code); 400 `"Invalid redirect URI"`; 404 `"Review user not found"`; 403 `"User account is banned"`; 429 from the rate limiter.

### `POST /api/mobile/live-activity/register`

**Why:** Registers the iOS device's APNs push token for a specific Live Activity instance, so the batch cron (`/api/cron/live-activity-update`) can push duty-state updates to the lock screen while a duty is running.

**Auth:** Bearer access token (`getSessionWithBearer()`).

**Request:** JSON body — `dutyId` (required, string, must be an `ACTIVE` duty owned by the caller), `pushToken` (required, string, APNs push token), `activityId` (required, string, the iOS `Activity` identifier).

```bash
curl -sS -X POST "$HSR_URL/api/mobile/live-activity/register" \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"dutyId":"<dutyId>","pushToken":"<apnsToken>","activityId":"<activityId>"}'
```

**Response:**
```json
{ "success": true }
```
Upserts a `LiveActivityToken` row keyed on `(dutyId, activityId)`. Errors: 401 `"Invalid or expired token"`; 400 `"dutyId, pushToken, and activityId are required"`; 404 `"Active duty not found"` (duty missing, not the caller's, or not `ACTIVE`); 500 `"Failed to register Live Activity token"`.

### `POST /api/mobile/live-activity/unregister`

**Why:** Called when the Live Activity ends (duty stops) so the device stops receiving push updates for a duty that is no longer running.

**Auth:** Bearer access token (`getSessionWithBearer()`).

**Request:** JSON body — `dutyId` (required, string), `activityId` (optional, string; if omitted, deletes every token for that duty and user).

```bash
curl -sS -X POST "$HSR_URL/api/mobile/live-activity/unregister" \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"dutyId":"<dutyId>","activityId":"<activityId>"}'
```

**Response:**
```json
{ "success": true, "deleted": 1 }
```
Deletes matching `LiveActivityToken` rows scoped to `dutyId` + `userId` (+ `activityId` if given). Errors: 401 `"Invalid or expired token"`; 400 `"dutyId is required"`; 500 `"Failed to unregister Live Activity token"`.

### `POST /api/push/token`

**Why:** Registers an Expo push token for the mobile app (Android/iOS) so `expo-push.ts` can deliver ordinary in-app notifications (not Live Activity) to that device.

**Auth:** Bearer access token (`getSessionWithBearer()`).

**Request:** JSON body — `token` (required, string, Expo push token), `platform` (required, `"ios"` | `"android"`), `deviceId` (optional, string).

```bash
curl -sS -X POST "$HSR_URL/api/push/token" \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"token":"ExponentPushToken[...]","platform":"ios","deviceId":"device-123"}'
```

**Response:**
```json
{ "success": true }
```
Delegates to `registerPushToken()` (`src/lib/notifications/mobile-push.ts`), which upserts an `ExpoPushToken` row. Errors: 401 `"Unauthorized"`; 400 `"Token is required"`, `"Platform must be 'ios' or 'android'"`, `"Invalid push token format"` (thrown by `registerPushToken`); 500 `"Failed to register push token"`.

### `DELETE /api/push/token`

**Why:** Unregisters an Expo push token (e.g. on mobile logout) so the device stops receiving push notifications.

**Auth:** Bearer access token (`getSessionWithBearer()`).

**Request:** JSON body — `token` (required, string).

```bash
curl -sS -X DELETE "$HSR_URL/api/push/token" \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"token":"ExponentPushToken[...]"}'
```

**Response:**
```json
{ "success": true }
```
Delegates to `unregisterPushToken()`, which deletes the matching `ExpoPushToken` row (no ownership check beyond the token value itself). Errors: 401 `"Unauthorized"`; 400 `"Token is required"`; 500 `"Failed to unregister push token"`.

### `POST /api/push/subscribe`

**Why:** Registers a browser's Web Push subscription (VAPID) so the web app can deliver push notifications to a logged-in member's browser — the web-side counterpart to the mobile Expo token above.

**Auth:** Web session cookie (`getSession()`).

**Request:** JSON body is the raw `PushSubscription` object from the browser — `endpoint` (required, string, truncated to 255 chars for the DB unique constraint), `keys.p256dh` (required, string), `keys.auth` (required, string).

```bash
curl -sS -X POST "$HSR_URL/api/push/subscribe" \
  -H "Cookie: auth-token=..." \
  -H "Content-Type: application/json" \
  -d '{"endpoint":"https://fcm.googleapis.com/...","keys":{"p256dh":"...","auth":"..."}}'
```

**Response:**
```json
{ "success": true }
```
Upserts a `PushSubscription` row keyed on `(userId, endpoint)`. Errors: 401 `"Unauthorized"`; 400 `"Invalid subscription data"` (missing endpoint/keys); 500 `"Failed to save subscription"`.

### `DELETE /api/push/subscribe`

**Why:** Unsubscribes a browser from web push (e.g. the user disabled notifications), removing the stored VAPID subscription.

**Auth:** Web session cookie (`getSession()`).

**Request:** JSON body — `endpoint` (required, string).

```bash
curl -sS -X DELETE "$HSR_URL/api/push/subscribe" \
  -H "Cookie: auth-token=..." \
  -H "Content-Type: application/json" \
  -d '{"endpoint":"https://fcm.googleapis.com/..."}'
```

**Response:**
```json
{ "success": true }
```
Deletes every `PushSubscription` row matching `(userId, endpoint)`. Errors: 401 `"Unauthorized"`; 400 `"Endpoint required"`; 500 `"Failed to delete subscription"`.

### `GET /api/push/duty-reminders`

**Why:** External cron endpoint that scans today's approved schedule requests and sends push reminders 30/10/5 minutes before a driver's or dispatcher's shift starts, so members don't have to watch a clock.

**Auth:** Shared-secret cron auth — `Authorization: Bearer <CRON_SECRET>` header (not `CRON_API_KEY`, unlike most other cron routes documented elsewhere in this app).

**Request:** No body/params.

```bash
curl -sS "$HSR_URL/api/push/duty-reminders" -H "Authorization: Bearer $CRON_SECRET"
```

**Response:**
```json
{
  "success": true,
  "remindersSent": ["János Kovács (30 min)", "Péter Nagy (10 min)"],
  "timestamp": "2026-09-13T07:00:00.000Z"
}
```
For `DRIVER` requests, the reminder time is the train's origin departure time (`originDepartureTime()`, direction-aware for return workings); for `DISPATCHER` requests it defaults to 08:00 (no per-request start time is stored). Sends via `sendDutyReminder()`; a failure to send one reminder is caught and logged, not fatal to the run. Errors: 401 `"Unauthorized"` (missing/wrong `CRON_SECRET`); 500 `"Failed to process reminders"`.
