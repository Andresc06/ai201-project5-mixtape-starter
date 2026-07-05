# Mixtape Bug Hunt — Submission

## Codebase Map

### App structure

- **`app.py`** — Flask application factory (`create_app`). Initializes the shared `SQLAlchemy` `db` object, configures the database URI (SQLite by default), registers the four blueprints (`songs`, `playlists`, `users`, `feed`), and calls `db.create_all()` on startup. This is the single source of the `db` instance that every model and service imports.

- **`models.py`** — Defines all SQLAlchemy models and association tables:
  - `User` — has `listening_streak`/`last_listened_at` (for streaks), and relationships to songs shared, ratings, listening events, notifications, playlists, and friends.
  - `Song` — has `shared_by` (the original sharer, a FK to `User`), and a `tags` relationship via the `song_tags` association table.
  - `ListeningEvent` — one row per "user listened to song at time T"; this is the raw data both the streak service and the feed service are built on.
  - `Rating` — one row per (user, song) pair (unique constraint), score 1–5.
  - `Playlist` — has a `songs` relationship via the `playlist_entries` association table, which (unlike a plain many-to-many) carries extra columns: `position`, `added_by`, `added_at`. Songs in a playlist have an explicit order, not just insertion order.
  - `Notification` — generic notification record (`user_id`, `notification_type`, `body`, `read`).
  - Association tables: `friendships` (symmetric many-to-many, inserted as two rows per friendship by `seed_data.py`), `song_tags`, `playlist_entries`.

- **`routes/`** — Thin HTTP layer. Every route parses the request, delegates immediately to a `services/` function, and formats the response (JSON + status code). No business logic lives here — routes catch `ValueError` from services and turn it into a 400/404 JSON error.
  - `songs.py` — `/songs/search`, `/songs/<id>`, `/songs/<id>/rate`, `/songs/<id>/listen`
  - `playlists.py` — `POST /playlists/`, `/playlists/<id>`, `/playlists/<id>/songs` (GET + POST)
  - `users.py` — `/users/<id>`, `/users/<id>/streak`, `/users/<id>/notifications`, `POST /users/notifications/<id>/read`
  - `feed.py` — `/feed/<id>/listening-now`, `/feed/<id>/activity`

- **`services/`** — All business logic. This is where the five tracked bugs live.
  - `streak_service.py` — `record_listening_event()` creates a `ListeningEvent` and calls `update_listening_streak()`, which compares `now.date()` to `user.last_listened_at.date()` to decide: same day → no-op, +1 day → increment, otherwise → reset to 1.
  - `feed_service.py` — `get_friends_listening_now()` finds each friend's most recent `ListeningEvent` within a `RECENT_THRESHOLD` window; `get_activity_feed()` returns the most recent N events with no time filter at all.
  - `search_service.py` — `search_songs()` matches title/artist (case-insensitive `ILIKE`) and joins against `song_tags` to include tag data.
  - `notification_service.py` — `create_notification()` is the generic constructor. `add_to_playlist()` adds a song to a playlist and notifies the original sharer. `rate_song()` upserts a `Rating` (one per user/song, enforced by a unique constraint).
  - `playlist_service.py` — `create_playlist()`, `get_playlist_songs()` (orders songs by the `position` column on `playlist_entries`), `get_playlist()`, `get_user_playlists()`.

- **`tests/`** — `pytest` suites for streaks, search, and playlists, each using an in-memory SQLite DB per test. Several tests encode the *expected* (bug-free) behavior directly, e.g. `test_streak_increments_on_sunday`, `test_search_no_duplicates_multi_tag_song`, `test_playlist_returns_all_songs` — these currently fail against the buggy service code and are effectively regression tests for the fixes.

- **`seed_data.py`** — Populates the DB with 5 users/friendships, 25 songs (with 0/1/3+ tags to exercise the search bug), 3 playlists, and a mix of very-recent (~10–20 min old) and older (2–58 hr old) listening events — the older ones are explicitly commented as "should NOT appear in listening now after fix," which is a direct hint about the feed bug's expected fix.

### Data flow — a user rates a song

1. Client sends `POST /songs/<song_id>/rate` with `{user_id, score}`.
2. `routes/songs.py::rate()` parses the body, validates `user_id`/`score` are present, and calls `services.notification_service.rate_song(user_id, song_id, int(score))`.
3. `rate_song()` validates the score is 1–5, loads the `Song` and `User`, then checks for an existing `Rating` for that `(user_id, song_id)` pair — if found it updates the score in place (upsert), otherwise it creates a new `Rating` row. It commits and returns the `Rating`.
4. The route serializes `rating.to_dict()` and returns 201.
5. **Notably, `rate_song()` never calls `create_notification()`** — contrast with `add_to_playlist()` in the same file, which explicitly notifies `song.shared_by` after adding a song to a playlist. This asymmetry is Issue #4: a friend rating your shared song produces no notification, even though the "song added to playlist" path does.

### Patterns noticed

- **Routes are pure glue.** Every route function is ~5–10 lines: parse input → call one service function → format output. All logic (including validation beyond "is this field present") lives in `services/`.
- **Services raise `ValueError` for not-found/invalid-input, which routes translate to 400/404.** There's no custom exception hierarchy — just this one convention, applied consistently.
- **Association tables carry extra metadata when the relationship isn't a plain many-to-many** (`playlist_entries` has `position`/`added_by`/`added_at`; `friendships` and `song_tags` don't need extra columns and stay plain join tables).
- **Tests double as bug specifications.** Several test names/comments literally say what the bug is and what the fix should produce (e.g. `# Bug causes this to return 4`, `# Should be 1, bug causes it to be 3`, `# Should increment, not reset`), which is a strong signal for where to look and how to verify a fix.

## Bugs to Fix

I attempted to reproduce all five before picking three:

- **Issue #3 (duplicate search results) did NOT reproduce** in this environment, so I swapped it out. Running `tests/test_search.py` (all 5 tests pass, including `test_search_no_duplicates_multi_tag_song`) and manually querying a 3-tag song both showed exactly one result, not three. I traced this to the installed **SQLAlchemy 2.0.51**: the legacy `Query.all()` API used in `search_service.py` (`db.session.query(Song).outerjoin(...).all()`) automatically de-duplicates ORM entities by identity when the query selects whole mapped objects, even without an explicit `.distinct()`. So the join-fanout code path that would produce duplicates on an older SQLAlchemy version doesn't actually manifest as a bug against this project's pinned dependencies. Since I could not trigger the reported symptom, I moved to a different issue instead of fixing something I couldn't observe.
- I chose **Issues #1, #4, and #5** to fix, since all three reproduced deterministically on the first attempt.

### Issue #1 — Listening streak keeps resetting (Sunday)

**How I reproduced it:** Ran the existing `tests/test_streaks.py::test_streak_increments_on_sunday`, which calls `update_listening_streak()` directly with a Saturday timestamp (streak → 1), then a Sunday timestamp one day later. Expected the streak to increment to 2 (one day gap, weekend included); got 1 instead — the code takes the `else` branch and resets. Confirmed with `pytest tests/test_streaks.py -v`:

Root cause is visible directly in `streak_service.py:73`: `elif days_since_last == 1 and today.weekday() != 6:` — the extra `and today.weekday() != 6` clause excludes Sundays from the normal "listened yesterday → increment" path, so any user who listens on a Sunday after listening Saturday gets their streak reset to 1 instead of incremented, even though only one calendar day passed.

**The root cause:** The `elif` branch responsible for incrementing the streak on a one-day gap was gated by `today.weekday() != 6` in addition to `days_since_last == 1`. `datetime.weekday()` returns `6` for Sunday, so whenever the current listen happened to fall on a Sunday, this extra condition evaluated to `False` even when the user had listened exactly one day prior — sending execution into the `else` branch, which resets `listening_streak` to `1`. There was no comment or docstring rule justifying a Sunday exception; the streak rules only mention "same day," "yesterday," and "more than one day," with no weekend carve-out. The condition was simply extraneous logic that happened to break every week for any user listening on a Sunday after listening Saturday.

**My fix and side-effect check:** Removed the `and today.weekday() != 6` clause, leaving `elif days_since_last == 1:` to match the documented rule exactly. Verified with `pytest tests/test_streaks.py -v`, including `test_streak_increments_on_sunday` and the other boundary tests that were already passing.

### Issue #4 — No notification when a friend rates your song

**How I reproduced it:** Wrote a small script creating a `sharer` user, a `rater` user, and a song shared by `sharer`. Called `get_notifications(sharer.id)` (empty, as expected), then called `rate_song(rater.id, song.id, 5)` to have `rater` give the song 5 stars, then checked `get_notifications(sharer.id)` again — still empty.

Root cause: `rate_song()` in `notification_service.py` upserts the `Rating` row and commits, but never calls `create_notification()`. Contrast with `add_to_playlist()` a few lines above, which explicitly notifies `song.shared_by` after adding a song. The two actions are structurally parallel but only one of them was wired up to actually notify.

### Issue #5 — Last song in a playlist never shows up

**How I reproduced it:** Ran the existing `tests/test_playlists.py`, which seeds a playlist with 5 songs at positions 1-5 and calls `get_playlist_songs()`. Both playlist tests fail.

Root cause is visible directly in `playlist_service.py`: `return [song.to_dict() for song in songs[:-1]]`. The songs are correctly queried and ordered by `position` ascending, but the `[:-1]` slice unconditionally drops the last element of that ordered list before returning it, so whichever song is actually last in the playlist is always omitted from the response, regardless of playlist length.
