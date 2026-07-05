# Mixtape Bug Hunt — Submission

## AI Usage

I used AI tools mainly to help clean up and organize my write-up in
submission.md. It also helped me find the bugs. For each issue,
I read through the relevant service file and the corresponding test
file on my own, formed a hypothesis about what was going wrong, and
with the help of AI fix the issues. I also asked it a couple of general questions, like the
difference between `weekday()` and `isoweekday()` in Python, to
double check my understanding of the streak fix before committing to
it. Moreover, I used AI to help me write the commit messages and the final submission write-up. Moreover, for the stretch features, I used AI to help me write the code for the notification service and the corresponding tests. Those issues were harder to debug and I needed to make sure I was using the correct syntax and methods for the notification service. Overall, AI was a helpful tool in my workflow, but I made sure to verify its suggestions and understand the changes being made before committing them.

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

- **Issue #3 (duplicate search results) did NOT reproduce** in this environment initially. Running `tests/test_search.py` (all 5 tests pass, including `test_search_no_duplicates_multi_tag_song`) and manually querying a 3-tag song both showed exactly one result, not three. I traced this to the installed **SQLAlchemy 2.0.51**: the legacy `Query.all()` API used in `search_service.py` (`db.session.query(Song).outerjoin(...).all()`) automatically de-duplicates ORM entities by identity when the query selects whole mapped objects, even without an explicit `.distinct()`. So the join-fanout code path that would produce duplicates on an older SQLAlchemy version doesn't actually manifest as a bug against this project's pinned dependencies. I moved on to a different issue first since I couldn't observe the reported symptom, then came back and fixed the underlying code smell anyway (see below) since it's a real latent defect independent of ORM version behavior.
- I chose **Issues #1, #4, and #5** to fix as the required three, since all three reproduced deterministically on the first attempt, then also fixed **Issues #2 and #3** as bonus fixes.

### Issue #1 — Listening streak keeps resetting (Sunday)

**How I reproduced it:** Ran the existing `tests/test_streaks.py::test_streak_increments_on_sunday`, which calls `update_listening_streak()` directly with a Saturday timestamp (streak → 1), then a Sunday timestamp one day later. Expected the streak to increment to 2 (one day gap, weekend included); got 1 instead — the code takes the `else` branch and resets. Confirmed with `pytest tests/test_streaks.py -v`:

Root cause is visible directly in `streak_service.py:73`: `elif days_since_last == 1 and today.weekday() != 6:` — the extra `and today.weekday() != 6` clause excludes Sundays from the normal "listened yesterday → increment" path, so any user who listens on a Sunday after listening Saturday gets their streak reset to 1 instead of incremented, even though only one calendar day passed.

**The root cause:** The `elif` branch responsible for incrementing the streak on a one-day gap was gated by `today.weekday() != 6` in addition to `days_since_last == 1`. `datetime.weekday()` returns `6` for Sunday, so whenever the current listen happened to fall on a Sunday, this extra condition evaluated to `False` even when the user had listened exactly one day prior — sending execution into the `else` branch, which resets `listening_streak` to `1`. There was no comment or docstring rule justifying a Sunday exception; the streak rules only mention "same day," "yesterday," and "more than one day," with no weekend carve-out. The condition was simply extraneous logic that happened to break every week for any user listening on a Sunday after listening Saturday.

**My fix and side-effect check:** Removed the `and today.weekday() != 6` clause, leaving `elif days_since_last == 1:` to match the documented rule exactly. Verified with `pytest tests/test_streaks.py -v`, including `test_streak_increments_on_sunday` and the other boundary tests that were already passing.

### Issue #4 — No notification when a friend rates your song

**How I reproduced it:** Wrote a small script creating a `sharer` user, a `rater` user, and a song shared by `sharer`. Called `get_notifications(sharer.id)` (empty, as expected), then called `rate_song(rater.id, song.id, 5)` to have `rater` give the song 5 stars, then checked `get_notifications(sharer.id)` again — still empty.

**How I found the root cause:** Started at `routes/songs.py::rate()`, which calls `services.notification_service.rate_song()`. Read `rate_song()` top-down: it validates the score, loads the `Song`/`User`, upserts the `Rating`, commits, and returns — no call to `create_notification()` anywhere in the function. Scrolled up to `add_to_playlist()` in the same file, which handles a structurally identical situation ("a friend did something to a song you shared") and does call `create_notification(user_id=song.shared_by, ...)` after its main action. Comparing the two side by side confirmed `rate_song()` was simply missing the equivalent call — not a logic error inside an existing notification path, but an absent step.

**The root cause:** `rate_song()` in `notification_service.py` upserts the `Rating` row and commits, but never calls `create_notification()`. Contrast with `add_to_playlist()` a few lines above, which explicitly notifies `song.shared_by` after adding a song. The two actions are structurally parallel but only one of them was wired up to actually notify.

**My fix and side-effect check:** Added a `create_notification()` call at the end of `rate_song()`, guarded by `song.shared_by != user_id` (mirroring `add_to_playlist()`'s `song.shared_by != added_by_user_id` check) so rating your own shared song doesn't notify yourself. Verified with a manual script: a friend's rating now creates a `song_rated` notification for the sharer, rating your own song creates none, and updating an existing rating notifies again.

### Issue #5 — Last song in a playlist never shows up

**How I reproduced it:** Ran the existing `tests/test_playlists.py`, which seeds a playlist with 5 songs at positions 1-5 and calls `get_playlist_songs()`. Both playlist tests fail.

**How I found the root cause:** Started at `routes/playlists.py::get_songs()`, which calls `services.playlist_service.get_playlist_songs()`. Read the function top-down: it loads the `Playlist`, then queries `Song` joined to `playlist_entries` filtered by `playlist_id` and ordered ascending by `position`. That query is correct — it returns all songs in the right order. The bug was in the very last line, converting query results to dicts: `[song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice stood out immediately as unrelated to anything the docstring describes, and dropping the last element of an already-correctly-ordered list explains exactly the reported symptom.

**The root cause:** The songs were correctly queried and ordered by `position` ascending, but the final list comprehension sliced with `songs[:-1]` before building the response, unconditionally discarding the last song in the ordered list.

**My fix and side-effect check:** Changed `songs[:-1]` to `songs`, so all queried songs are returned. Verified with `pytest tests/test_playlists.py -v`: `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order`, now pass, and `test_empty_playlist_returns_empty_list` still passes, confirming the fix doesn't break the empty-playlist boundary on the other side of the slice.

### Issue #2 — Friends Listening Now shows people from yesterday

**How I reproduced it:** No existing test covers `feed_service.py`, so I wrote a manual script: created two friends, one `ListeningEvent` for the friend at 20 hours before "now," and called `get_friends_listening_now()` for the other user. It returned that 20-hour-old event as a "listening now" entry — a listen from roughly yesterday showing up as if it were happening now.

**How I found the root cause:** Started at `routes/feed.py` with `listening_now()`, which calls `services.feed_service.get_friends_listening_now()`. That function computes `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD` and filters `ListeningEvent.listened_at >= cutoff` — the filtering logic itself is correct and does exactly what "friends listening now" should do. The only place to look was the value of `RECENT_THRESHOLD`, defined at the top of the file as `timedelta(hours=24)`. That immediately looked too generous for a "listening now" feature, and `seed_data.py` confirmed it: its comments explicitly say events seeded at 10-20 minutes old ("within the past 30 minutes") "should appear in listening now," while events seeded starting at 2 hours old ("Older events") "should NOT appear in listening now after fix." A 24-hour window lets everything from 2 hours up through very-nearly-24-hours old through, directly contradicting that stated intent.

**The root cause:** `RECENT_THRESHOLD` was set to 24 hours, so any friend activity from within the last day — not just genuinely "now" — passed the `listened_at >= cutoff` filter and got surfaced as if it were happening live. The feature is named and used for "listening now," but the threshold effectively implemented "listened at some point today," which is a materially different, much weaker guarantee. Nothing else in the function was wrong; the recency window itself was simply too wide for what the feature promises.

**My fix and side-effect check:** Changed `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(minutes=30)`, matching the window `seed_data.py`'s comments describe as "recent." Verified with a manual script covering both sides of the new boundary: a 20-hour-old event now correctly returns 0 feed entries (previously 1); a 10-minute-old event still returns 1; and when both a 10-minute-old and a 2-hour-old event exist for the same friend, the feed still returns exactly 1 entry (the most recent one), confirming the per-friend dedup logic is unaffected by the threshold change.

### Issue #3 — The same song keeps showing up twice in search

**How I reproduced it:** As noted above, this did not reproduce against this project's installed SQLAlchemy 2.0.51: `tests/test_search.py` in `test_search_no_duplicates_multi_tag_song`, and a manual query for a 3-tag song returned exactly 1 result. I confirmed the cause independently of the symptom by checking the raw `song_tags` join table directly (3 rows for the 3-tag song) against `db.session.query(Song).outerjoin(song_tags, ...).all()` (1 result) — the join fan-out data was genuinely there (3 rows worth of join keys), but the legacy ORM `Query.all()` API auto-deduplicated the mapped `Song` entities by identity before returning them, silently absorbing the fan-out that an equivalent `select()`/`Session.execute()` call (SQLAlchemy 2.0-style) would not.

**How I found the root cause:** Read `search_songs()` in `search_service.py` top-down: it joins `Song` to `song_tags` on `song_id`, filters on `Song.title`/`Song.artist`, and returns `.to_dict()` for each result. The `filter()` clause never references `song_tags.c.tag_id` or any tag-related condition — the join contributes nothing to which songs match the query. Then checked `Song.to_dict()` in `models.py` and the `Song.tags` relationship, which is defined with `lazy="subquery"` — meaning tags are already fetched via a separate query keyed off `Song.id`, entirely independent of whatever join `search_songs()` performs. That made it clear the `outerjoin(song_tags, ...)` in the search query was doing nothing useful: it wasn't needed for filtering and wasn't needed for tag data (the relationship handles that separately). Its only real effect was producing one result row per matching `song_tags` row, N rows for a song with N tags, which is precisely the "duplicate in search" symptom the issue describes, just currently masked by this SQLAlchemy version's auto-dedup on `Query.all()`.

**The root cause:** `search_songs()` performed an `outerjoin` against `song_tags` that was never used for filtering or data selection, tags are loaded independently through the `Song.tags` relationship. Joining a one-to-many association table without aggregating or deduplicating means the query returns one row per matching join pair, so a song with 3 tags produces 3 rows for the same song. On SQLAlchemy versions/APIs that return raw rows instead of auto-deduplicating identity-mapped ORM entities, this manifests exactly as "the same song shows up multiple times in search," matching the reported symptom.

**My fix and side-effect check:** Removed the unnecessary `outerjoin(song_tags, ...)` (and the now-unused `Tag`/`song_tags` imports) from `search_songs()`, since the query only ever filtered on `Song.title`/`Song.artist` and tags come from the separate `Song.tags` relationship regardless. Verified with `pytest tests/test_search.py -v` and a manual check that a 3-tag song still returns exactly 1 result with all 3 tags present.