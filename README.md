# HSR API Documentation

Reference for the HTTP API of the HSR (Hungarian State Railways) website. The API is the website's own backend — Next.js route handlers under `src/app/api/` — so everything the site does is available here, with the same permissions.

| Document | What is in it | Endpoints |
|---|---|---:|
| [Public API](public-api.md) | Everything callable **without** signing in: news, status, timetable and delays, leaderboards, public profiles, the forum's read side, the shop catalogue, document verification, and the links in emails | 30 |
| [Authenticated API](api.md) | Everything that needs a **session or a token**: sign-in, your own account and settings, duty, training, exams and supervision, the forum's write side, tickets, documents and the shop, the admin panel, and the server-to-server jobs | 490 |
| [Mobile API](mobile-api.md) | How an external app signs in (OAuth authorization code exchanged for an access + refresh token), Live Activity and push-token registration | 12 |

## Which one do I need?

- **Reading public data** (a timetable widget, a forum archive, a status badge) → [Public API](public-api.md). No key, no account.
- **Acting as a member** (an app, a bot acting for one person) → sign the member in through the [Mobile API](mobile-api.md), then call the [Authenticated API](api.md) with `Authorization: Bearer <token>`.
- **Admin tooling** → the Admin sections of the [Authenticated API](api.md); the member's token carries their permission flags.
- **The game server and the crontab** → the Server-to-server sections of the [Authenticated API](api.md).

## Quick start

```bash
export HSR_URL="https://hsrgames.hu"

# Public: no credentials
curl -sS "$HSR_URL/api/status"
curl -sS "$HSR_URL/api/forum/feed?kind=latest"

# Authenticated: a bearer token from the mobile OAuth flow
export ACCESS_TOKEN="<access-token>"
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/auth/session"
curl -sS -H "Authorization: Bearer $ACCESS_TOKEN" "$HSR_URL/api/profile/notifications"
```

## Conventions

- JSON in, JSON out. Errors are `{ "error": "…" }` with a matching HTTP status.
- Times are ISO 8601 in UTC (`2026-10-16T08:00:00.000Z`); operating days and "Budapest time" rules (e.g. the 06:00 closing-exam vote) are Europe/Budapest.
- Ids are opaque strings (cuid). Discord ids are snowflake strings.
- Paths written as `[id]` take a concrete value.

## Recent changes

- Notification settings (`GET`/`PATCH /api/profile/notifications`): per-topic e-mail and Discord DM switches.
- Terms of Service / Privacy Policy versions: `GET /api/legal/current`, `POST /api/legal/accept`, `GET`/`POST /api/admin/legal/versions`; `user.legal` in `GET /api/auth/session`. `POST /api/admin/terms/notify` was removed.
- Documented: the forum, timetable and pool drafts, scheduled Discord messages, delay detection, the activity log, account export/unlink, goals, league, reminders, and the newer cron jobs.
- Removed from the reference: the wiki, the track map, Apple Wallet passes and the old pool-assignment endpoints, which no longer exist.
