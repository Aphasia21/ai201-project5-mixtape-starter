# Mixtape — Codebase Map

Mixtape is a Flask + SQLAlchemy JSON API for a social music app: friends share
songs, build collaborative playlists, rate tracks, and track listening streaks
and feeds. This document maps the code, traces two real data flows end to end,
and calls out the architectural patterns the app follows.

---

## Layer overview

The app is a clean three-layer stack, plus the data model:

```
HTTP request
   │
   ▼
routes/      ── thin HTTP layer: parse request, call one service, format JSON
   │
   ▼
services/    ── all business logic lives here
   │
   ▼
models.py    ── SQLAlchemy models + association tables (the data model)
   │
   ▼
SQLite (instance/mixtape.db)
```

`app.py` wires it together; `seed_data.py` populates a realistic dataset.

---

## Main files and what each does

### `app.py` — application factory
`create_app(config=None)` builds the Flask app: sets the SQLite URI (overridable
via `DATABASE_URL`), initializes the shared `db = SQLAlchemy()` object, registers
the four blueprints under URL prefixes, and runs `db.create_all()`. The `db`
instance is defined here and imported everywhere else, which is what keeps a
single session/engine across the app.

| Blueprint | Prefix | Source |
|-----------|--------|--------|
| `songs_bp` | `/songs` | `routes/songs.py` |
| `playlists_bp` | `/playlists` | `routes/playlists.py` |
| `users_bp` | `/users` | `routes/users.py` |
| `feed_bp` | `/feed` | `routes/feed.py` |

### `models.py` — the data model
Defines **7 models** and **3 association tables**.

**Models:**
- `User` — `username`, `email`, `listening_streak`, `last_listened_at`. Has a
  self-referential many-to-many `friends` relationship (via the `friendships`
  table) declared `lazy="dynamic"`, plus backref'd collections for shared songs,
  ratings, listening events, notifications, and playlists.
- `Tag` — just an `id` + unique `name`.
- `Song` — `title`, `artist`, `album`, `genre`, and `shared_by` (FK to the user
  who shared it). `shared_by` is the field every notification flow keys off of.
- `ListeningEvent` — one row per listen (`user_id`, `song_id`, `listened_at`).
  This is the backbone of both the feed and the streak features.
- `Rating` — `score` (1–5) with a `UniqueConstraint(user_id, song_id)`, so a user
  has at most one rating per song (re-rating updates in place).
- `Playlist` — `name`, `created_by`, `is_collaborative`.
- `Notification` — `user_id` (recipient), `notification_type`, `body`, `read`.

**Association tables** (plain `db.Table`, not model classes):
- `friendships` — symmetric user↔user. The app inserts **two rows** per
  friendship (see `seed_data.add_friendship`) to keep it bidirectional.
- `song_tags` — song↔tag many-to-many.
- `playlist_entries` — playlist↔song, but **not a plain join table**: it carries
  extra columns `position` (explicit ordering, not insertion order), `added_by`,
  and `added_at`. Songs in a playlist have an explicit position.

Every model exposes a `to_dict()` used for JSON serialization; `Song.to_dict()`
flattens its tags to a list of name strings.

### `routes/` — HTTP layer (one blueprint per resource)
Each route does the same three things: pull params/JSON off the request,
validate presence, call exactly one service function, and translate the result
(or a raised `ValueError`) into a JSON response with a status code. No business
logic, no direct DB queries beyond a couple of trivial `db.session.get(User, …)`
existence checks.

- `routes/songs.py` — `GET /songs/search?q=`, `GET /songs/<id>`,
  `POST /songs/<id>/rate`, `POST /songs/<id>/listen`.
- `routes/playlists.py` — `POST /playlists/`, `GET /playlists/<id>`,
  `GET /playlists/<id>/songs`, `POST /playlists/<id>/songs`.
- `routes/users.py` — `GET /users/<id>`, `GET /users/<id>/streak`,
  `GET /users/<id>/notifications`, `POST /users/notifications/<id>/read`.
- `routes/feed.py` — `GET /feed/<user_id>/listening-now`,
  `GET /feed/<user_id>/activity`.

### `services/` — business logic (all of it)
- `streak_service.py` — `record_listening_event()` writes a `ListeningEvent` and
  calls `update_listening_streak()`, which applies the consecutive-calendar-day
  rules (start at 1, no-op if already today, +1 if yesterday, reset otherwise).
  `get_streak()` reads the stored value.
- `feed_service.py` — `get_friends_listening_now()` returns each friend's single
  most recent listen within a recency window (deduped one-per-friend);
  `get_activity_feed()` returns the latest N friend events with no recency filter.
- `search_service.py` — `search_songs()` case-insensitive matches title/artist
  and outer-joins tags; `get_song()` fetches one song by id.
- `notification_service.py` — `create_notification()` (the shared writer),
  `add_to_playlist()` (adds a song to a playlist **and** notifies the sharer),
  `rate_song()` (upserts a `Rating`), `get_notifications()`, `mark_as_read()`.
- `playlist_service.py` — `create_playlist()`, `get_playlist_songs()`
  (ordered by `playlist_entries.position`), `get_playlist()`,
  `get_user_playlists()`.

### `seed_data.py` — test dataset
Drops and recreates all tables, then inserts 5 users with bidirectional
friendships, 25 songs (deliberately spanning 0-tag, 1-tag, and 3+-tag cases),
3 playlists with positioned entries, recent + older listening events, seeded
streaks, and one sample notification. Run with `python seed_data.py`.

### `tests/`
`pytest` suites for the streak, search, and playlist services
(`test_streaks.py`, `test_search.py`, `test_playlists.py`).

---

## Data flow #1 — a friend adds your song to a playlist (triggers a notification)

This is the notification flow that actually fires today.

```
POST /playlists/<playlist_id>/songs   {song_id, added_by}
  │
  ▼  routes/playlists.py :: add_song()            (parses JSON, validates presence)
  │
  ▼  notification_service.add_to_playlist(playlist_id, song_id, added_by)
        1. db.session.get(Song, song_id)          → 404-able ValueError if missing
        2. db.session.get(User, added_by)         → validate adder
        3. db.session.get(Playlist, playlist_id)  → validate playlist
        4. if song not in playlist.songs:
               playlist.songs.append(song)         → writes a playlist_entries row
               db.session.commit()
        5. if song.shared_by != added_by:          → don't notify yourself
               create_notification(
                   user_id = song.shared_by,        ← recipient is the ORIGINAL sharer
                   notification_type = "song_added_to_playlist",
                   body = "<adder> added your song '<title>' to '<playlist>'.")
```

Key point: the notification recipient is `song.shared_by` — the person who
originally shared the song — **not** the playlist owner. The `if song.shared_by
!= added_by` guard suppresses self-notifications. Retrieval is the mirror image:
`GET /users/<id>/notifications` → `get_notifications()` reads the recipient's
`Notification` rows newest-first.

## Data flow #2 — a song appears in a friend's feed

Notable because it's **decoupled through the `ListeningEvent` table**: nothing
"pushes" to a feed. The feed is computed on read.

```
Write path (a friend listens):
  POST /songs/<id>/listen → routes/songs.py :: listen()
     → streak_service.record_listening_event()
         → INSERT ListeningEvent(user_id, song_id, listened_at=now); commit

Read path (you open your feed):
  GET /feed/<user_id>/listening-now → routes/feed.py :: listening_now()
     → feed_service.get_friends_listening_now(user_id)
         friend_ids = [f.id for f in user.friends]
         SELECT ListeningEvent WHERE user_id IN friend_ids
                               AND listened_at >= cutoff
                ORDER BY listened_at DESC
         dedupe to most-recent-per-friend
         join each event → User + Song → feed dicts
```

A song lands in *your* feed because a *friend played it* and that friend is in
your `friends` set — the song's own `shared_by` is irrelevant here.

---

## Patterns I noticed

1. **Strict route→service delegation.** Every route immediately hands off to one
   service function. Routes never contain query logic or business rules; they
   only parse input and format output. All logic is in `services/`.

2. **`ValueError` as the error protocol across the boundary.** Services raise
   `ValueError("<thing> not found")` for missing/invalid entities; routes catch
   `ValueError` and map it to a 404 or 400 JSON response. This is the single,
   consistent error-handling convention between the two layers.

3. **`db.session.get(Model, id)` for existence checks everywhere.** Both routes
   and services validate entities by primary-key lookup before acting, keeping
   the "does it exist?" check uniform.

4. **`to_dict()` as the serialization contract.** Models never get jsonified
   directly; every model owns its JSON shape via `to_dict()`, so the API payload
   is defined next to the schema.

5. **A shared `db` singleton created in `app.py`.** Defining `db` in `app.py` and
   importing it in `models.py` and every service is what avoids circular-import
   and multiple-engine problems in the factory pattern.

6. **Explicit ordering via association-table columns.** Playlists don't rely on
   insertion order — `playlist_entries.position` makes order first-class, and
   `get_playlist_songs()` sorts on it explicitly.

7. **Read-time computation over stored aggregates.** Feeds and (mostly) streaks
   are derived from `ListeningEvent` rows at request time rather than maintained
   as denormalized state, which keeps writes simple at the cost of read queries.

---

## Root Cause Analysis

All bugs were reproduced against a freshly seeded database (`python seed_data.py`)
before any fix was written, by driving the same service function each route calls.
Status: **#1, #2, #4, #5 reproduced; #3 investigated but not reproducible under
this stack (see below).**

### Issue #1 — Listening streak resets on Sundays *(reporter: kenji)* — ✅ FIXED (commit `508e163`)
- **Navigation strategy:** Symptom is a streak change after listening, so I
  started at the listen endpoint. `POST /songs/<id>/listen` →
  [routes/songs.py:43](routes/songs.py:43) `listen()` → calls
  `record_listening_event()` in
  [streak_service.py:14](services/streak_service.py:14) → which delegates the
  streak math to `update_listening_streak(user, now)`
  ([streak_service.py:42](services/streak_service.py:42)). Reading that function
  top to bottom, the day-difference branch at line 73 was the only place a
  weekday was referenced — that pointed straight at the root cause.
- **How reproduced:** Set a user to `listening_streak=12`, `last_listened_at=`
  Saturday 2026-07-11 22:00 UTC, then called
  `update_listening_streak(user, now=Sunday 2026-07-12 09:00 UTC)` — the same
  function `record_listening_event` calls. Used a controlled `now` because the
  function takes it as a parameter and today (2026-07-07) is a Tuesday. Then
  listened again Monday.
- **Observed vs expected:** Sat→Sun gave streak **1** (expected 13); the follow-up
  Monday listen gave **2** — matching kenji's "bumped it to 2" exactly.
- **Data condition:** triggers *only* when the current day is a Sunday; every
  other consecutive-day pair increments correctly (verified Mon→Tue = ok).
- **Root cause:** [streak_service.py:73](services/streak_service.py:73) —
  `elif days_since_last == 1 and today.weekday() != 6:`. `weekday()==6` is Sunday,
  so a legitimate consecutive-day listen skips the increment branch and falls
  into `else: listening_streak = 1`. The `weekday() != 6` clause has no business
  being in the consecutive-day check.
- **Fix:** Dropped the `and today.weekday() != 6` clause, leaving
  `elif days_since_last == 1: user.listening_streak += 1`. Smallest change that
  restores the documented rule (consecutive day → +1) with no special-casing.
- **Verification:** `pytest tests/test_streaks.py` → 5/5 pass, including the
  previously-failing `test_streak_increments_on_sunday`. Repro rerun: Sat(12)→Sun
  now yields **13**. Other branches unchanged (new-user=1, same-day no-op,
  skipped-day reset all still green), so no related behavior regressed.

### Issue #2 — "Friends Listening Now" shows people from yesterday *(reporter: nova)* — ✅ reproduced
- **How reproduced:** Deleted darius' seeded events, inserted a single
  `ListeningEvent` for him `listened_at = now − 10 hours` (his "11pm last night"
  vs nova's "9am" check), then called `get_friends_listening_now(nova.id)` —
  the function behind `GET /feed/<id>/listening-now`.
- **Observed vs expected:** darius appeared in the feed (`['simone','kenji','darius']`);
  expected only friends who listened *today / right now*.
- **Data condition:** any friend whose most recent listen is between "minutes ago"
  and 24 hours ago is treated as "listening now."
- **Root cause:** [feed_service.py:13](services/feed_service.py:13) —
  `RECENT_THRESHOLD = timedelta(hours=24)`. A full-day window means last night's
  listen stays in "now" until the same clock time the next day, exactly as nova
  described. "Listening now" needs a short window (minutes) or a same-calendar-day
  check, not 24h.

### Issue #3 — Same song appears 2–3× in search *(reporter: simone)* — ⚠️ investigated, NOT reproduced here
- **How reproduced (attempted):** Confirmed `Crown Heights Anthem` has **3 tags**
  in the seed (state required by the bug is present), then called
  `search_songs("Anthem")` — the function behind `GET /songs/search?q=Anthem`.
- **Observed vs expected:** returned **1** row for Crown Heights Anthem (bug report
  expects 3). **Not reproduced.**
- **Why it doesn't reproduce:** The `.outerjoin(song_tags)` at
  [search_service.py:27](services/search_service.py:27) genuinely fans the result
  out to one row per tag — I confirmed the raw SQL returns **3 rows**. But the code
  uses the legacy `db.session.query(Song).all()` API, which **auto-de-duplicates
  full ORM entities by primary key**. Verified on the identical statement under
  SQLAlchemy 2.0.51:
  - `session.query(Song)…all()` → **1** (what the app runs; auto-uniqued)
  - `select(Song)…scalars().all()` → **3** (2.0 style; not uniqued)
  - `select(Song)…unique().scalars().all()` → 1
- **Conclusion:** the duplicate-producing join is a real latent defect and the fix
  (add `.distinct()` / drop the unused tag join) is still correct, but it cannot be
  *observed* through the current code path on this SQLAlchemy version. It would
  surface if the query were ported to 2.0 `select()`/`scalars()` style. Because it
  is not reproducible here, it is **not** counted among the fixes; #1/#2/#4/#5 are.

### Issue #4 — Rating a song sends no notification *(reporter: aaliya)* — ✅ FIXED (commit `790134b`)
- **Navigation strategy:** The report contrasts a *working* flow (playlist add
  notifies) with a *broken* one (rating doesn't), and both live in
  `notification_service.py`, so I read them side by side. Rating:
  `POST /songs/<id>/rate` → [routes/songs.py:29](routes/songs.py:29) `rate()` →
  `rate_song()` ([notification_service.py:73](services/notification_service.py:73)).
  Working comparison: `add_to_playlist()`
  ([notification_service.py:35](services/notification_service.py:35)) ends with a
  `create_notification(...)` call; `rate_song()` had no such call. The diff
  between the two functions *was* the bug.
- **How reproduced:** Picked a song shared by simone; had kenji rate it via
  `rate_song(kenji.id, song.id, 5)` — the function behind
  `POST /songs/<id>/rate` — and counted the sharer's notifications before/after.
- **Observed vs expected:** notifications stayed **0 → 0**; the rating row *was*
  saved (matching "it shows on the song"), but no notification was created.
- **Data condition:** deterministic — happens for every rating of any user's song.
- **Root cause:** [notification_service.py:73](services/notification_service.py:73)
  — `rate_song()` upserts the `Rating` and commits but never calls
  `create_notification()`. The parallel `add_to_playlist()` *does* notify, which is
  why aaliya sees playlist notifications but not rating ones.
- **Fix:** After the commit in `rate_song`, added a `create_notification(...)` with
  `notification_type="song_rated"`, guarded by `if song.shared_by != user_id` so a
  user rating their own song isn't notified — the same guard `add_to_playlist`
  uses. Reused the existing `create_notification` helper rather than writing new
  persistence logic.
- **Verification:** Rating another user's song now creates exactly one
  `song_rated` notification (`"kenji rated your song 'After Hours' 5 stars."`) and
  the `Rating` is still saved; rating one's own song creates none (guard works).
  Full suite `pytest tests/` → **13/13 pass** (up from 10/13 baseline), so no
  existing behavior regressed. *Note:* re-rating an already-rated song sends
  another notification — consistent with how `add_to_playlist` behaves and
  acceptable for this fix's scope.
- **Bonus finding (unlisted, NOT fixed — out of scope for the 3 chosen bugs):**
  `add_to_playlist()` itself raises `IntegrityError: NOT NULL constraint failed:
  playlist_entries.position` because `playlist.songs.append(song)` populates only
  the two FK columns, leaving the NOT-NULL `position` and `added_by` columns unset.
  Seed data avoids this by inserting `playlist_entries` rows explicitly, so the
  append path had never run. Flagged for a follow-up.

### Issue #5 — Last song in a playlist never shows up *(reporter: darius)* — ✅ FIXED (commit `2fcc67f`)
- **Navigation strategy:** Symptom shows when listing a playlist's songs.
  `GET /playlists/<id>/songs` → [routes/playlists.py:34](routes/playlists.py:34)
  `get_songs()` → `get_playlist_songs()` in
  [playlist_service.py:38](services/playlist_service.py:38). The query itself
  looked correct (ordered by `position`), so I read the return statement — the
  `[:-1]` slice on the last line was the defect.
- **How reproduced:** Read the stored `playlist_entries` rows for "Friday Energy"
  directly (7 entries), then called `get_playlist_songs(pl.id)` — the function
  behind `GET /playlists/<id>/songs` — and compared counts and the highest-position
  song.
- **Observed vs expected:** **7 stored, 6 returned**; the position-7 song
  (`Harlem Renaissance`) was missing — matching darius's "always hiding exactly one
  song: the last one added."
- **Data condition:** deterministic for any non-empty playlist; a 1-song playlist
  would return 0.
- **Root cause:** [playlist_service.py:66](services/playlist_service.py:66) — the
  query correctly orders by `playlist_entries.position` ascending, then the return
  statement slices `songs[:-1]`, dropping the last (highest-position, most-recently-
  added) element.
- **Fix:** Changed `songs[:-1]` to `songs` — return every ordered row.
- **Verification:** `pytest tests/test_playlists.py` → 3/3 pass, including the
  previously-failing `test_playlist_returns_all_songs` (now 5) and
  `test_playlist_returns_songs_in_order`. The empty-playlist test still passes
  (`[]` unaffected). Repro rerun on "Friday Energy": **7 stored, 7 returned**, and
  `Harlem Renaissance` (position 7) is present. Ordering is preserved because only
  the slice changed, not the `ORDER BY position`.
