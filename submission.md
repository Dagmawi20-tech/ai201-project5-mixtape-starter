# Mixtape Bug Hunt — submission.md

---

## AI Usage

I used Claude to help orient myself in the codebase and identify bugs before writing any fix code. Specifically:

**Codebase orientation:** I pasted all five service files and `models.py` into Claude and asked it to summarize what each file does and identify any suspicious patterns. This helped me build a mental model of the app structure quickly — particularly understanding how `playlist_entries` uses a `position` column for ordering and how `notification_service.py` is responsible for both the rating logic and the notification creation.

**Bug identification:** After reading the code myself, I confirmed my suspicions with Claude. For Issue #3, I wasn't immediately sure why duplicates appeared — Claude explained that an `outerjoin` on a many-to-many join table produces one row per matching tag, which causes the same song to appear multiple times in results. I verified this by reading the `song_tags` table definition in `models.py` and confirming that songs can have multiple tags.

**Fix verification:** For each fix I wrote the change myself and asked Claude to confirm the reasoning was correct before committing. In all cases I read and understood the fix before applying it — I did not paste AI-generated code without reviewing it.

---

## Codebase Map

### Main Files

**`app.py`** — Flask app factory. Creates the Flask app, configures SQLAlchemy, and registers route blueprints. Entry point for the application.

**`models.py`** — Defines 6 SQLAlchemy models and 3 join tables:
- `User` — has `listening_streak` and `last_listened_at` for streak tracking; many-to-many `friends` relationship via `friendships` table
- `Song` — shared by a user; has ratings, listening events, and tags
- `Tag` — simple label attached to songs via `song_tags` join table
- `ListeningEvent` — records when a user listened to a song with a timestamp
- `Rating` — a user's 1–5 score for a song; unique per user+song pair
- `Playlist` — created by a user; songs attached via `playlist_entries` join table which includes a `position` column for ordering
- `Notification` — message sent to a user when a friend interacts with their song

**`services/streak_service.py`** — Handles listening streak logic. When a user listens, it checks how many days have passed since their last listen and increments, maintains, or resets the streak accordingly.

**`services/feed_service.py`** — Returns "Friends Listening Now" (friends who listened within a recent time window) and a general activity feed of all recent friend listening events.

**`services/search_service.py`** — Searches songs by title or artist using a case-insensitive `ilike` filter.

**`services/notification_service.py`** — Creates and retrieves notifications. Also handles `add_to_playlist` and `rate_song` actions — these live here because they need to both save data and trigger notifications.

**`services/playlist_service.py`** — Creates playlists and retrieves ordered song lists using the `position` column in `playlist_entries`.

### Pattern I Noticed

Every route delegates immediately to a service function. Routes handle input parsing and response formatting; all business logic lives in `services/`. This means any bug in app behavior will be in a service file, not a route file.

### Data Flow — User Rates a Song

1. `POST /songs/<song_id>/rate` in `routes/songs.py`
2. Calls `notification_service.rate_song(user_id, song_id, score)`
3. `rate_song()` validates the score, finds the song and user, creates or updates a `Rating` record
4. Commits to the database
5. (After the fix) Creates a `Notification` for the song's original sharer if the rater is a different user
6. Returns the `Rating` instance to the route, which formats it as JSON

---

## Root Cause Analysis

---

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:**
Created a playlist with multiple songs via the API, then called `GET /playlists/<id>/songs`. The last song in the playlist was consistently missing from the response regardless of how many songs were in the playlist.

**How I found the root cause:**
I went directly to `playlist_service.py` as indicated in the README. The `get_playlist_songs()` function queries songs ordered by `position` — that logic looked correct. I read the final return statement and immediately saw the bug: `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice is Python for "all elements except the last one."

**The root cause:**
The `get_playlist_songs()` function correctly queries and orders all songs in the playlist, but then slices off the last element with `songs[:-1]` before returning. This means the last song in every playlist is silently dropped from every response. There is no valid reason to exclude the last song — this appears to be an accidental leftover from development.

**Fix and side-effect check:**
Changed `songs[:-1]` to `songs`. This is the smallest possible fix — one character removal. I checked `get_playlist()` and `get_user_playlists()` to confirm neither touches song ordering, so there are no side effects.

---

### Issue #4 — Got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:**
Called `POST /songs/<id>/rate` as a user rating another user's song. Then called `GET /users/<sharer_id>/notifications` — no notification appeared for the song's sharer, even though rating is supposed to trigger one.

**How I found the root cause:**
I read `notification_service.py` and compared `add_to_playlist()` and `rate_song()` side by side. `add_to_playlist()` ends with a `create_notification()` call that notifies the song's sharer. `rate_song()` saves the rating and commits — but then just returns the rating with no notification call. The notification creation was simply never written.

**The root cause:**
The `rate_song()` function in `notification_service.py` saves the rating correctly but never calls `create_notification()` for the song's original sharer. The `add_to_playlist()` function in the same file has the correct pattern — create the data, then notify the sharer — but `rate_song()` was missing the notification step entirely.

**Fix and side-effect check:**
Added a `create_notification()` call after `db.session.commit()` in `rate_song()`, guarded by `if song.shared_by != user_id` to avoid notifying a user when they rate their own song. I checked that `create_notification()` itself commits to the database correctly and confirmed the body message format matches the pattern used in `add_to_playlist()`.

---

### Issue #3 — The same song keeps showing up twice in search

**How I reproduced it:**
Called `GET /songs/search?q=neon` (or any query matching a song that has multiple tags). Songs with more than one tag appeared multiple times in the results — once per matching tag.

**How I found the root cause:**
I read `search_service.py` and noticed the query uses `.outerjoin(song_tags, Song.id == song_tags.c.song_id)`. The `song_tags` table is a many-to-many join table — one song can have many tags. An outer join on this table produces one result row per tag associated with the song. If a song has 3 tags, it appears 3 times in the raw query results. Since `Song.to_dict()` is called on each row, the same song gets serialized 3 times.

**The root cause:**
The `search_songs()` function joins `song_tags` into the query but has no purpose for it — the filter only checks `Song.title` and `Song.artist`, not any tag fields. The join was either an unfinished attempt to add tag-based search or a leftover from a refactor. Because it's a join on a many-to-many table, it multiplies result rows by the number of tags per song.

**Fix and side-effect check:**
Removed the `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` line entirely. The filter conditions don't reference `song_tags` at all, so removing the join has no effect on search behavior — it only eliminates the duplicate rows. Tags are still returned correctly because `Song.to_dict()` loads them via the `tags` relationship defined in `models.py`, which is independent of the query join.

---

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:**
Simulated a user listening on consecutive days by calling `POST /users/<id>/listen` on back-to-back days. The streak incremented correctly from Monday through Saturday, but listening on Sunday did not increment the streak — it reset to 1 instead.

**How I found the root cause:**
I read `streak_service.py` and found `update_listening_streak()`. The streak logic checks `days_since_last` — if 0, no change; if 1, increment; otherwise reset. But the increment branch had an extra condition: `today.weekday() != 6`. Python's `weekday()` returns 6 for Sunday. So on any Sunday, even if exactly one day had passed since Saturday's listen, the condition evaluated to `False` and the streak reset to 1 instead of incrementing.

**The root cause:**
The streak increment condition `elif days_since_last == 1 and today.weekday() != 6` incorrectly blocks streak increments on Sundays. `weekday() == 6` means Sunday in Python, so `weekday() != 6` is `False` every Sunday — causing the condition to fall through to the `else` branch which resets the streak. A Sunday listen after a Saturday listen should increment the streak, not reset it.

**Fix and side-effect check:**
Removed `and today.weekday() != 6` from the condition, leaving `elif days_since_last == 1:`. The day-of-week check was entirely unnecessary — the streak logic only cares about the number of days elapsed, not which day of the week it is. I verified the other branches (0 days = no change, 2+ days = reset) are unaffected.

---

### Issue #2 — Friends Listening Now shows people from yesterday

**How I reproduced it:**
Called `GET /feed/friends-listening-now` after seeding listening events from the previous day. Friends who had listened 20+ hours ago appeared in the "listening now" feed, which should only show recent activity.

**How I found the root cause:**
I read `feed_service.py` and found `RECENT_THRESHOLD = timedelta(hours=24)`. The cutoff is calculated as `datetime.now(timezone.utc) - RECENT_THRESHOLD`, so any listening event within the last 24 hours qualifies as "now." Someone who listened 23 hours ago — effectively yesterday — would appear as currently listening.

**The root cause:**
`RECENT_THRESHOLD` is set to 24 hours, which is far too long for a "listening now" feature. A threshold of 24 hours means "listened sometime today or yesterday" — not "listening right now." The feature name and user expectation is that it shows friends who are actively listening, which requires a much shorter window.

**Fix and side-effect check:**
Changed `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(minutes=30)`. Thirty minutes is a reasonable window for "currently listening" — it allows for small gaps between sessions without showing stale data. I checked `get_activity_feed()` in the same file — it intentionally has no recency filter (it returns the most recent N events regardless of when), so this change does not affect it.

---

## Git Log

```
818e261 (HEAD -> bugfix/mixtape) fix: reduce Friends Listening Now threshold from 24 hours to 30 minutes
a47fa32 fix: remove incorrect weekday check that prevented streak from incrementing on Sundays
02f5577 fix: remove outerjoin on song_tags that caused duplicate results for songs with multiple tags
1e8199f fix: add missing notification when a user rates a song
ee7e48d fix: remove [:-1] slice that excluded last song from playlist results
2dfdeaa (origin/main, origin/HEAD, main) Add .gitignore file and update README with setup instructions
7b64551 initial commit
```
