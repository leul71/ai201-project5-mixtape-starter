# Mixtape Bug Hunt — Submission

## AI Usage

I used Claude as my primary AI assistant throughout this project. During codebase
orientation I pasted all five service files and asked Claude to identify the main
responsibility of each file and spot any bugs. I was able to locate bug 1 and 2 and 5. 
I used claude to identify the other bugs by eading the code directly. For each bug I verified the diagnosis myself by reading the relevant function before making any change. Claude generated the fix for each bug and I reviewed and applied it. I also used Claude to generate this submission document based on my testing and our conversation.

---

## Codebase Map

### Main files and their roles

**app.py** — Flask app factory. Creates the Flask app, initializes SQLAlchemy, and
registers all route blueprints. Also handles database setup.

**models.py** — Defines 6 SQLAlchemy models: User, Song, Playlist, ListeningEvent,
Rating, Notification, and Tag. Also defines 3 association tables: friendships
(user-to-user many-to-many), song_tags (song-to-tag many-to-many), and
playlist_entries (song-to-playlist many-to-many with position and added_by columns).

**routes/** — Four route files that handle HTTP input/output only. Each route
immediately delegates to a service function. No business logic lives in routes.

**services/** — Five service files where all business logic lives:
- `streak_service.py` — tracks consecutive listening days per user
- `feed_service.py` — returns friends currently listening and activity feed
- `search_service.py` — searches songs by title or artist
- `notification_service.py` — creates notifications when friends interact with songs
- `playlist_service.py` — creates playlists and retrieves ordered song lists

**seed_data.py** — Populates the database with test users, songs, playlists, and
listening events for development and testing.

### Data flow — user rates a song

1. `POST /songs/<song_id>/rate` in `routes/songs.py` receives the request
2. It calls `notification_service.rate_song(user_id, song_id, score)`
3. `rate_song()` validates the score, looks up the song and rater, checks for an
   existing rating and updates or creates it, then commits to the database
4. After commit, it checks if the rater is different from the song's original sharer
5. If so, it calls `create_notification()` to notify the sharer

### Patterns noticed

Every route delegates immediately to a service — routes handle parsing and response
formatting only. All business logic is in services. Models use UUIDs as primary keys
generated at creation time. The playlist_entries association table adds extra columns
(position, added_by, added_at) beyond a simple join table.

---

## Bug Fixes

### Bug 1 — My listening streak keeps resetting

**How I reproduced it:** The bug is conditional — it only triggers on Sundays.
Reading the streak logic revealed the condition directly without needing to wait
for a Sunday. Tracing `update_listening_streak()` showed the branch that increments
the streak had an extra weekday check.

**How I found the root cause:** I read `streak_service.py` and traced
`update_listening_streak()`. The `elif` branch that increments the streak had an
extra condition: `and today.weekday() != 6`. I checked what `weekday()` returns
for Sunday — it returns 6. That condition was blocking the increment on Sundays.

**Root cause:** `datetime.weekday()` returns 6 for Sunday. The streak increment
branch had the condition `today.weekday() != 6` which evaluates to False on
Sundays, causing the code to fall through to the `else` branch and reset the streak
to 1 instead of incrementing it. Any user who listened on a Sunday would have their
streak broken even if they had listened the day before.

**Fix:** Removed the `and today.weekday() != 6` condition entirely. The day-of-week
check was never part of the streak logic — consecutive days should always increment
regardless of what day of the week it is. Checked that the other two branches
(already listened today = no change, more than one day gap = reset) were unaffected.

---

### Bug 2 — Friends Listening Now shows people from yesterday

**How I reproduced it:** Read `feed_service.py` and checked the `RECENT_THRESHOLD`
constant at the top of the file. It was set to `timedelta(hours=24)` — any listening
event in the past 24 hours would appear as "listening now," including events from
the previous day.

**How I found the root cause:** `RECENT_THRESHOLD` is defined as a module-level
constant and used directly in `get_friends_listening_now()` as the cutoff for
filtering `ListeningEvent` records. 24 hours is the entire previous day, which
contradicts the "now" framing of the feature.

**Root cause:** `RECENT_THRESHOLD = timedelta(hours=24)` was too wide. A friend who
listened to something 23 hours ago would appear as currently active. The feature is
called "Friends Listening Now" — the intent is a short active window, not a full day.

**Fix:** Changed `RECENT_THRESHOLD` to `timedelta(minutes=30)`. 30 minutes is a
reasonable window for "currently listening." Checked that `get_activity_feed()` does
not use this constant — it doesn't, so that function was unaffected.

---

### Bug 3 — The same song keeps showing up twice in search

**How I reproduced it:** Read `search_songs()` in `search_service.py`. The query
used an `outerjoin` on the `song_tags` association table. A song with multiple tags
produces multiple rows in that join — one per tag — causing the same song to appear
multiple times in results.

**How I found the root cause:** The `outerjoin(song_tags, Song.id == song_tags.c.song_id)`
line was joining the song_tags table into the query. SQLAlchemy returns one result
row per joined row, so a song with 3 tags appears 3 times. The join was unnecessary
because `Song.to_dict()` already loads tags through the SQLAlchemy relationship
defined in `models.py`.

**Root cause:** The `outerjoin` on `song_tags` was producing duplicate Song rows —
one per tag association. Since tags are already accessible via the `Song.tags`
relationship and loaded automatically in `to_dict()`, the join served no purpose
and only caused duplicates.

**Fix:** Removed the `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` line
from the query. The filter on title and artist still works correctly without it.
Verified that `song.to_dict()` still returns tags by checking the model relationship.

---

### Bug 4 — Got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:** Read `notification_service.py` and compared `add_to_playlist()`
and `rate_song()` side by side. `add_to_playlist()` calls `create_notification()`
after committing. `rate_song()` commits and returns the rating with no notification call.

**How I found the root cause:** The two functions follow the same pattern — look up
song and user, perform an action, commit — but only `add_to_playlist()` had the
notification step. `rate_song()` was missing it entirely.

**Root cause:** `rate_song()` never called `create_notification()`. The function
saved and committed the rating correctly but had no code to notify the song's
original sharer. The notification logic existed in `add_to_playlist()` as a clear
template but was never added to `rate_song()`.

**Fix:** Added a `create_notification()` call at the end of `rate_song()`, after
the commit, following the same pattern as `add_to_playlist()`. The notification is
only sent if the rater is different from the song's sharer. Checked that the rater
and song objects were already available in scope from earlier in the function.

---

### Bug 5 — The last song in a playlist never shows up

**How I reproduced it:** Read `get_playlist_songs()` in `playlist_service.py`.
The return statement was `return [song.to_dict() for song in songs[:-1]]`. The
`[:-1]` slice removes the last element of any list.

**How I found the root cause:** The `[:-1]` slice on the return line was immediately
visible as wrong. Python's `[:-1]` returns all elements except the last one. The
docstring says "This function returns all songs in the playlist" — the slice
directly contradicts that.

**Root cause:** `songs[:-1]` slices off the last song in the ordered list before
returning it. Every playlist was missing its final song regardless of playlist size.
This appears to be an accidental edit — the correct return is simply `songs`.

**Fix:** Changed `songs[:-1]` to `songs`. Verified the query itself was correct —
it joins on `playlist_entries`, filters by `playlist_id`, and orders by
`position` ascending, which is the intended behavior.

---

## Git Log

_(Paste screenshot of `git log --oneline` output here after all commits are made)_