# HSR Public API

Endpoints you can call **without signing in and without a token**. Everything else — your own data, duty, training, admin — is in the [Authenticated API](api.md). [README](README.md) has the overview.

```bash
export HSR_URL="https://hsrgames.hu"
curl -sS "$HSR_URL/api/news?limit=3"
```

## How these endpoints behave

- **No credentials.** No cookie, no `Authorization` header. When a session cookie or a bearer token *is* sent, a few of them add the caller's own details (your rank on a leaderboard, your like on a post), but none of them requires it.
- **Two kinds of public.** Most are open reads. The [links from emails](#links-from-emails-token-in-the-url) are public too, but the token in their URL *is* the credential: it names one registration or one vote and nothing else — treat those URLs as private.
- **Cross-origin.** The forum reads answer `GET` from any origin (`Access-Control-Allow-Origin: *`, never with credentials), so a browser app on another site can read the forum.
- **Caching.** Responses carry `Cache-Control` where the data allows it (e.g. 15 s for the feed, 5 min for topics). Respect it.
- **Rate limits.** Some endpoints are limited per IP; a limited request gets `429` with `{ "error": "…" }`.
- **Errors.** `{ "error": "message" }` with the usual HTTP status. Forum errors are locale keys (`forumPostMissing`) rather than sentences.

## Contents

- [Site and Status](#site-and-status)
- [Timetable and Delays](#timetable-and-delays)
- [Leaderboards and Profiles](#leaderboards-and-profiles)
- [Forum (public reads)](#forum-public-reads)
- [Shop Catalogue](#shop-catalogue)
- [Document Verification](#document-verification)
- [Links from Emails (token in the URL)](#links-from-emails-token-in-the-url)

## Site and Status

### `GET /api/maintenance-status`

**Why:** Public, unauthenticated check the `/maintenance` page uses to display the admin's custom message even to users locked out of every admin-gated route.

**Auth:** public.

**Request:** No parameters.

```bash
curl -sS "$HSR_URL/api/maintenance-status"
```

**Response:** `{ "enabled": false, "message": null }`.

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

---

### `GET /api/status`

**Why:** The status page's data: whether the website's dependencies are working right now — database, Discord, email, game server and translation.

**Auth:** None. Also allowed for a half-linked account.

```bash
curl -sS "$HSR_URL/api/status"
```

**Response:**
```json
{
  "checkedAt": "2026-10-16T08:00:00.000Z",
  "overall": "UP",
  "services": [{ "key": "database", "label": "…", "state": "UP", "detail": "…" }]
}
```
`state` / `overall` is `UP`, `DEGRADED`, `DOWN` or `OFF` (switched off on purpose). If the check itself fails the answer is `overall: "DOWN"` with no services. Cached for 20 seconds.

---

### `GET /api/legal/current`

**Why:** The Terms of Service / Privacy Policy version in force, with what changed — for an acceptance screen. With a session it also says whether the caller still has to accept it.

**Auth:** None (session or bearer optional).

**Request:** Query `locale` (`hu` | `en` | `de`, optional — the language of `changes` and `effectiveDateLabel`; defaults to the `hsr_locale` cookie, then Hungarian).

```bash
curl -sS "$HSR_URL/api/legal/current?locale=en"
```

**Response:**
```json
{
  "version": { "id": "clv1234567890", "effectiveDate": "2026-10-16T00:00:00.000Z", "effectiveDateLabel": "16 October 2026", "publishedAt": "2026-10-16T09:00:00.000Z", "changes": ["Notifications can now also arrive as Discord messages."] },
  "mustAccept": true
}
```
`effectiveDateLabel` is the date already written out in that language, as the website prints it. `{ "version": null }` while no version has been published (nothing to accept). `mustAccept` is only present with a session; accept with `POST /api/legal/accept`.

## Timetable and Delays

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

### `GET /api/schedule/delays`

**Why:** Today's live train delays for the timetable: per train, the projected delay in minutes and the last reported station event.

**Auth:** None.

```bash
curl -sS "$HSR_URL/api/schedule/delays"
```

**Response:**
```json
{
  "thresholds": { "lateThresholdMinutes": 3, "earlyThresholdMinutes": 1 },
  "runs": {
    "<trainId>": { "trainId": "…", "delayMinutes": 4, "lastStationId": "…", "lastEventKind": "DEPARTURE", "lastEventAt": "2026-10-16T08:12:00.000Z", "events": [ … ] }
  }
}
```
A train with no run today is absent from `runs`. Not cached.

## Leaderboards and Profiles

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

## Forum (public reads)

Reading the forum needs no account. These endpoints also answer cross-origin `GET` requests (`Access-Control-Allow-Origin: *`, no credentials), so a browser-side client on another site can read them. A session cookie, when present, only adds the caller's own viewer state (`liked`, `canEdit`, …).

### The post object

Every endpoint that returns posts returns the same `ForumPostDTO` shape:

```json
{
  "id": "clp...",
  "body": "Új menetrend holnaptól #gysev",
  "author": {
    "id": "clu...",
    "handle": "mozdonyvezeto",
    "displayName": "Mozdonyvezető",
    "avatarUrl": "https://tr.rbxcdn.com/...",
    "badges": [{ "id": "...", "name": "Mozdonyvezető", "nameEn": "Train Driver", "imageUrl": null, "emoji": "🚆", "color": "#3b82f6", "primary": true }],
    "isSupporter": false,
    "isStaff": false,
    "isOfficial": false
  },
  "media": [{ "id": "...", "url": "https://s3.hsrgames.hu/forum/...", "mime": "image/webp", "width": 1280, "height": 720, "durationMs": null, "video": false }],
  "topics": [{ "id": "...", "key": "gysev", "name": "GYSEV", "nameEn": null, "color": "#22c55e", "emoji": null }],
  "parentId": null,
  "rootId": null,
  "depth": 0,
  "replyingTo": null,
  "quoted": null,
  "likeCount": 12,
  "replyCount": 3,
  "repostCount": 1,
  "viewCount": 240,
  "createdAt": "2026-10-15T18:20:00.000Z",
  "editedAt": null,
  "deleted": false,
  "official": null,
  "viewer": { "liked": false, "reposted": false, "canEdit": false, "canDelete": false, "canReport": false, "canRepost": false }
}
```

- `official` is `{ "type": "UPDATE" | "DEVELOPMENT" | "NOTICE" | "EVENT" | "MAINTENANCE", "pinned": boolean }` on a post by the official HSR account; `pinned` is true for its first 24 hours.
- `deleted: true` marks a tombstone kept to hold a thread together; its `body` is empty.
- `quoted` is the quoted post (one level only, never a chain).

Forum errors are locale keys, not sentences (`{ "error": "forumPostMissing" }`); render them through the locale files.

---

### `GET /api/forum/feed`

**Why:** The home feed of the forum.

**Auth:** None. With a session, `for-you` is personalised and `following` works; without one, `following` falls back to `latest`.

**Request:** Query `kind` (`for-you` | `following` | `latest`, default `for-you`), `cursor` (opaque, from the previous page), `limit` (optional page size).

```bash
curl -sS "$HSR_URL/api/forum/feed?kind=latest"
```

**Response:** `{ "posts": [ForumPostDTO], "nextCursor": "…" | null, "kind": "latest" }`. Cached `public, max-age=15` for visitors, `private, no-store` for members.

---

### `GET /api/forum/posts/[id]`

**Why:** One post with its whole thread. Asking for a reply returns the conversation from its root, with `focusId` pointing at the reply.

**Auth:** None.

**Request:** Path `id`. Query `cursor` (ISO time of the last reply received) for the next 50 replies.

```bash
curl -sS "$HSR_URL/api/forum/posts/clp1234567890"
```

**Response:**
```json
{ "post": { "id": "clp...", "body": "…" }, "focusId": "clp...", "replies": [], "nextCursor": null }
```
`post` and each reply are `ForumPostDTO`s; replies are oldest first. `404 { "error": "forumPostMissing" }` if the post does not exist.

---

### `GET /api/forum/search`

**Why:** One search box over posts, topics and members. A query starting with `#` searches topics/tags only.

**Auth:** None.

**Request:** Query `q` (at least 2 characters), `cursor` (ISO time from the previous page).

```bash
curl -sS --get --data-urlencode "q=#gysev" "$HSR_URL/api/forum/search"
```

**Response:** `{ "query": "#gysev", "topics": [{ "id", "key", "name", "nameEn", "color", "emoji", "posts": 14 }], "members": [ForumAuthorDTO], "posts": [ForumPostDTO], "nextCursor": "…" | null }`. Shorter queries return empty lists. Posts are newest first, 20 per page; `members` is empty for a `#` query.

---

### `GET /api/forum/topics`

**Why:** The enabled topic chips the composer offers and the feed filters by.

**Auth:** None.

```bash
curl -sS "$HSR_URL/api/forum/topics"
```

**Response:** `{ "topics": [{ "id", "key", "name", "nameEn", "color", "emoji" }] }`. Cached for 5 minutes.

---

### `GET /api/forum/trending`

**Why:** The topics with the most new top-level posts over the last day.

**Auth:** None.

```bash
curl -sS "$HSR_URL/api/forum/trending"
```

**Response:** `{ "topics": [{ "topic": { "id", "key", "name", "nameEn", "color", "emoji" }, "posts": 9 }] }` — at most 5, cached for 1 minute.

---

### `GET /api/forum/users/[handle]`

**Why:** The forum half of a profile header: who the member is, their post/follower/following counts, and whether the caller follows them.

**Auth:** None (the `viewer` block is only meaningful with a session).

**Request:** Path `handle` (profile handle).

```bash
curl -sS "$HSR_URL/api/forum/users/mozdonyvezeto"
```

**Response:**
```json
{
  "author": { "id": "clu...", "handle": "mozdonyvezeto", "displayName": "Mozdonyvezető", "avatarUrl": null, "badges": [], "isSupporter": false, "isStaff": false, "isOfficial": false },
  "counts": { "posts": 42, "followers": 10, "following": 7 },
  "viewer": { "isSelf": false, "isFollowing": false, "canFollow": false }
}
```
`404 { "error": "forumUserMissing" }` for an unknown handle.

---

### `GET /api/forum/users/[handle]/posts`

**Why:** One member's posts for their profile tabs. Not gated by profile visibility: forum posts are public the moment they are written.

**Auth:** None. The `likes` tab is only returned to the member themselves (anyone else gets an empty list).

**Request:** Path `handle`. Query `tab` (`posts` | `replies` | `media` | `likes`, default `posts`), `cursor` (ISO time).

```bash
curl -sS "$HSR_URL/api/forum/users/mozdonyvezeto/posts?tab=media"
```

**Response:** `{ "posts": [ForumPostDTO], "nextCursor": "…" | null, "tab": "media" }` — 20 per page, newest first. `404 { "error": "forumUserMissing" }`.

---

### `GET /api/forum/export`

**Why:** The whole forum, every post once, oldest first — for archives, search engines and AI agents. Unlike the ranked feed, the order never changes between requests, so following `nextCursor` until it is `null` reads every post exactly once.

**Auth:** None.

**Request:** Query `limit` (default 100, max 200), `cursor` (id from the previous page), `order` (`asc` | `desc`, default `asc`), `since` (ISO time — only posts created at or after it), `topic` (topic key).

```bash
curl -sS "$HSR_URL/api/forum/export?limit=200&since=2026-10-01T00:00:00Z"
```

**Response:**
```json
{ "posts": [], "nextCursor": "clp..." , "next": "/api/forum/export?limit=200&cursor=clp...", "count": 200, "order": "asc", "docs": "/llms.txt" }
```
Replies are included; deleted posts appear as tombstones. Cached `public, max-age=60, s-maxage=300`.

---

### `GET /api/forum/link-preview`

**Why:** Title, description, image and embeddable player for a link in a post, for the card that renders it.

**Auth:** None. Rate limited per IP. Only allowlisted hosts are fetched; anything else returns `{ "preview": null }` without a request being made.

**Request:** Query `url`.

```bash
curl -sS --get --data-urlencode "url=https://www.youtube.com/watch?v=dQw4w9WgXcQ" "$HSR_URL/api/forum/link-preview"
```

**Response:** `{ "preview": { "url", "title", "description", "image", "video", … } | null }`. Cached for 30 minutes.

---

### `GET /api/forum/live`

**Why:** Server-sent events for the forum, so the feed and the notification bell update without polling.

**Auth:** None for the public events; with a session the stream also carries the caller's `notification` events.

```bash
curl -sS -N "$HSR_URL/api/forum/live"
```

**Response:** `text/event-stream` with three event names:
- `post` — a new top-level post (`ForumPostDTO` JSON).
- `announcement` — a new official post, which the feed pins (`ForumPostDTO` JSON).
- `notification` — the signed-in member has something new in the bell (payload `{}`; refetch `GET /api/forum/notifications`).

---

### `POST /api/forum/posts/[id]/view`

**Why:** Counts a view when a post is opened, and marks it as opened for a signed-in reader (used by the feed's "seen" ranking).

**Auth:** None (session optional).

```bash
curl -sS -X POST "$HSR_URL/api/forum/posts/clp1234567890/view"
```

**Response:** `{ "ok": true }`.

## Shop Catalogue

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

## Document Verification

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

## Links from Emails (token in the URL)

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
