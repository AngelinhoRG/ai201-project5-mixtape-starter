# Mixtape — Codebase Map & Bug Fixes

## Main files and responsibilities

- **`app.py`** — Flask application factory (`create_app`). Owns the single `db = SQLAlchemy()` instance, reads `DATABASE_URL`/`SECRET_KEY` from the environment, registers the four blueprints (`songs`, `playlists`, `users`, `feed`) each under its own URL prefix, and calls `db.create_all()` on startup. There is no `if __name__ == "__main__": app.run()` path used in dev — the app is launched via `flask run` against the `create_app` factory (running `python app.py` double-imports the app and breaks SQLAlchemy's mapper registration).

- **`models.py`** — All SQLAlchemy models, imported from the shared `db` in `app.py`:
  - `User` — has `listening_streak` (int) and `last_listened_at` (datetime) columns used directly by streak logic (no separate streak table). Self-referential `friends` many-to-many via the `friendships` association table, keyed by `(user_id, friend_id)`. Friendship is **not automatically symmetric** — the app relies on inserting both directions at write time (see `seed_data.py`'s `add_friendship` helper, which inserts `(u1,u2)` and `(u2,u1)`).
  - `Song` — owns `shared_by` (FK to `User`) and has a `tags` many-to-many relationship via the `song_tags` association table (loaded eagerly with `lazy="subquery"`, i.e. tags come back as a separate query, not a SQL join).
  - `ListeningEvent` — one row per "user listened to song" action; this is the source of truth for both the streak service and the friend feed.
  - `Rating` — one row per `(user_id, song_id)` pair, enforced by a `UniqueConstraint`; rating twice updates the same row instead of creating a new one.
  - `Playlist` — songs live in `playlist_entries`, a many-to-many association table that (unlike `song_tags`/`friendships`) carries extra columns: `position` (explicit ordering, not insertion order), `added_by`, `added_at`.
  - `Notification` — flat table of `(user_id, notification_type, body, read)`; no polymorphic target, the `body` string is pre-rendered human-readable text at creation time.

- **`routes/`** — one blueprint per resource area (`songs.py`, `playlists.py`, `users.py`, `feed.py`). Every route does the same three things and nothing else: parse the request, call exactly one `services/` function, translate a `ValueError` into a 4xx JSON error. No business logic or DB queries appear in routes.

- **`services/`** — all business logic:
  - `streak_service.py` — `record_listening_event()` writes a `ListeningEvent` then calls `update_listening_streak()`, which compares calendar dates (`.date()`) of `now` vs `user.last_listened_at` to decide same-day/consecutive-day/gap.
  - `feed_service.py` — `get_friends_listening_now()` looks up `user.friends`, queries `ListeningEvent` for those friend IDs within a `RECENT_THRESHOLD` window, and dedupes to one (most recent) event per friend. `get_activity_feed()` is the same query without the recency filter, just a `LIMIT`.
  - `search_service.py` — `search_songs()` does an `ilike` match on title/artist.
  - `notification_service.py` — `create_notification()` is the single low-level writer; `add_to_playlist()` and `rate_song()` are the two call sites that are supposed to invoke it after their respective side effects.
  - `playlist_service.py` — `get_playlist_songs()` joins `playlist_entries` to `Song` and orders by `position`.

- **`tests/`** — one file per feature area (`test_streaks.py`, `test_search.py`, `test_playlists.py`), each spinning up an isolated in-memory SQLite DB per test via the `app`/fixture pattern. These tests directly encode the expected behavior for 3 of the 5 tracked issues (see below).

- **`seed_data.py`** — populates a dev DB with users, friendships, tagged songs, listening events (both "recent" and "older" — deliberately shaped to expose the feed bug), playlists, and one example notification.

## Data flow — rating a song (Issue #4's path)

`POST /songs/<song_id>/rate` → `routes/songs.py:rate()` parses `user_id`/`score` from the JSON body → calls `notification_service.rate_song(user_id, song_id, score)` → which validates the score range, looks up the `Song` and rater `User`, upserts a `Rating` row (update if `(user_id, song_id)` already exists, thanks to the `UniqueConstraint`), commits, and returns the `Rating`.

Compare with the sibling flow, `POST /playlists/<id>/songs` → `routes/playlists.py:add_song()` → `notification_service.add_to_playlist()`, which does its side effect (append song to playlist) **and then calls `create_notification()`** for the song's original sharer if the adder isn't the sharer themselves.

`rate_song()` has no equivalent call — it does the side effect (saves the `Rating`) but never calls `create_notification()`. That's the entire bug behind issue #4: the two functions are structural siblings (same shape: look up song → look up actor → do side effect → notify original sharer) and one of them is just missing its last step.

## Patterns noticed

- **Routes are thin, services own logic.** No route file touches `db.session` directly; they only call into `services/`. This makes each bug traceable to exactly one service function per the README's issue table.
- **Association tables are the model for "relationship + extra data."** Plain many-to-many (`friendships`, `song_tags`) uses a bare join table; the moment ordering/attribution matters (`playlist_entries`), the join table grows extra columns instead of becoming its own model. `Rating` is the outlier — it's a real model (not a join table) because it needs an identity and a score, but it still functions like a join row on `(user_id, song_id)`.
- **Sibling functions with the same shape diverge silently.** `add_to_playlist()` and `rate_song()` in `notification_service.py` are structurally parallel but only one performs the notification step — this is exactly the shape of Issue #4, and worth checking for elsewhere (e.g. any other "user did X to another user's song" action that should notify).
- **Time-based filtering is a recurring soft spot.** Both the streak logic (calendar-day math with a day-of-week special case) and the feed logic (a recency threshold) hinge on datetime arithmetic that's easy to get subtly wrong without a test walking through boundary days/hours — and indeed both are tracked issues (#1, #2).
- **Tests exist for 3 of the 5 issues already** (`test_streaks.py` → #1, `test_search.py` → #3, `test_playlists.py` → #5), with assertions and comments that describe the buggy behavior directly (e.g. `assert len(songs) == 5  # Bug causes this to return 4`). Issues #2 and #4 (`feed_service.py`, `notification_service.py`) have no existing test coverage and were reproduced manually.

## Reproducing the five issues

1. **Streak resets on Sunday** — `services/streak_service.py:73`, `elif days_since_last == 1 and today.weekday() != 6:`. The `today.weekday() != 6` clause means a consecutive-day listen is only counted as consecutive if today isn't Sunday; on Sunday it falls through to the `else` branch and resets to 1. Reproduced by `tests/test_streaks.py::test_streak_increments_on_sunday` (currently failing).
2. **Friends Listening Now shows stale entries** — `services/feed_service.py:13`, `RECENT_THRESHOLD = timedelta(hours=24)`. A 24-hour window means a friend who listened at 11pm still shows up at 9am the next morning ("listening now" for a full calendar day). Reproduced manually: seeding a `ListeningEvent` ~10 hours old returns it as "listening now," matching nova's report about darius's 11pm-the-night-before listen still showing up.
3. **Duplicate search results** — `services/search_service.py:25-35` does `.outerjoin(song_tags, ...)` against `Song` without a corresponding `.distinct()`, so a song with N tags produces N joined rows, each turned into a duplicate dict. Reproduced by `tests/test_search.py::test_search_no_duplicates_multi_tag_song` (currently failing — 3 tags means 3 duplicate rows).
4. **No notification on rating** — `services/notification_service.py`, `rate_song()` never calls `create_notification()`, unlike its sibling `add_to_playlist()`. Reproduced manually by calling `rate_song()` and then `get_notifications()` — no new notification appears.
5. **Last playlist song missing** — `services/playlist_service.py:66`, `return [song.to_dict() for song in songs[:-1]]` truncates the last song in the ordered list. Reproduced by `tests/test_playlists.py::test_playlist_returns_all_songs` (currently failing — 5 songs seeded, 4 returned).
