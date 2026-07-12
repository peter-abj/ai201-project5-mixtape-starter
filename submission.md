# Mixtape Bug Hunt — Submission

## AI Usage Section

I used AI tools primarily for **code navigation and understanding unfamiliar patterns**, not for identifying bugs:

1. **Codebase Orientation**: Asked Claude to explain the relationships between SQLAlchemy models and how the playlist_entries join table works with explicit positioning.

2. **Data Flow Tracing**: Used Claude to trace how a song gets added to a playlist and which services are called in sequence.

3. **Python API Clarification**: Asked about `datetime.weekday()` vs `isoweekday()` return values when I suspected the Sunday bug in streak logic.

4. **SQL Query Explanation**: Asked Claude to explain the difference between using an outer join vs. the SQLAlchemy relationship to load related objects.

5. **Bug Verification**: After identifying suspected bugs through code reading, I used Python REPL and Flask shell to manually test behaviors with specific inputs rather than relying on AI's diagnosis.

**Where I verified myself**: I traced through each bug independently using the code, then ran targeted tests in the Flask app to confirm my hypothesis before making any fixes. AI helped explain confusing syntax or APIs, but all root cause analysis was from direct code reading and testing.

---

## Codebase Map

### Main Files and Responsibilities

**models.py**
- Defines 6 SQLAlchemy ORM models: `User`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, and `Tag`
- Three association tables: `friendships` (User many-to-many), `song_tags` (Song-Tag join), `playlist_entries` (Playlist-Song join with explicit `position` ordering)
- The `playlist_entries` table is critical for Issue #5 — songs in a playlist have an explicit `position` column for ordering

**routes/** directory (5 endpoint blueprints)
- `songs.py`: `/search`, `/songs/<id>`, `/songs/<id>/rate`, `/songs/<id>/listen`
- `users.py`: `/users/<id>`, `/users/<id>/streak`, `/users/<id>/notifications`
- `playlists.py`: `/playlists`, `/playlists/<id>`, `/playlists/<id>/songs`, `/playlists/<id>/songs` (POST)
- `feed.py`: `/feed/<id>/listening-now`
- Each route immediately delegates to a service function; routes handle input parsing and response formatting only

**services/** directory (5 service modules, where all bugs live)
- `streak_service.py`: `record_listening_event()`, `update_listening_streak()`, `get_streak()`
- `feed_service.py`: `get_friends_listening_now()`, `get_activity_feed()`
- `search_service.py`: `search_songs()`, `get_song()`
- `notification_service.py`: `create_notification()`, `add_to_playlist()`, `rate_song()`, `get_notifications()`, `mark_as_read()`
- `playlist_service.py`: `create_playlist()`, `get_playlist_songs()`, `get_playlist()`, `get_user_playlists()`

### Data Flow Example: Rating a Song (Issue #4)

1. User POSTs to `/songs/<song_id>/rate` with `user_id` and `score` in JSON body (routes/songs.py line 29)
2. Route handler calls `notification_service.rate_song(user_id, song_id, score)` (line 37)
3. `rate_song()` validates inputs, retrieves Song and User from DB, checks for existing rating
4. Creates or updates Rating record, commits to DB
5. **BUG**: Function does NOT call `create_notification()` to notify the song's sharer (missing lines after line 108)
6. Compare to `add_to_playlist()` in the same file (line 35) which DOES call `create_notification()` on lines 66-70 — this is the working pattern

### Key Architectural Pattern

All routes follow the same pattern:
```
Route handler → Parse request → Call service function → Format response
```

All business logic is in services. Services call each other (e.g., `notification_service.add_to_playlist()` calls `playlist_service.get_playlist_songs()`). Database access is always through `db.session` which is imported from the SQLAlchemy instance in app.py.

---

## Bug Fixes

### Issue #1: My listening streak keeps resetting (streak_service.py)

**How I reproduced it:**
1. Created a test scenario with a user who last listened on Saturday
2. Used Flask shell to manually call `update_listening_streak()` with a Sunday date
3. Confirmed: when `today = Sunday`, `today.weekday() = 6`, the condition `days_since_last == 1 and today.weekday() != 6` evaluates to `1 == 1 and 6 != 6` = `True and False` = `False`
4. This caused the function to fall through to the `else` clause and reset streak to 1

**How I found the root cause:**
- Read `streak_service.py` lines 42-79
- Noticed the condition on line 73: `elif days_since_last == 1 and today.weekday() != 6:`
- Researched Python's `weekday()`: returns 0-6 (Monday-Sunday), where 6 is Sunday
- Traced through logic: the condition is checking "if exactly one day has passed AND today is NOT Sunday"
- Realized: when a user listens on Saturday then Sunday, the condition is false, so the streak resets
- This is backwards—the code should increment the streak whenever one day passes, regardless of day of week

**The root cause:**
The `update_listening_streak()` function incorrectly conditions the streak increment on the day of the week. On line 73, it checks `today.weekday() != 6` (today is not Sunday) before incrementing. This means when the streak update happens on a Sunday (the most common bug-trigger day, as reported), the condition fails and the streak resets to 1 instead of incrementing. The `!= 6` check is incorrect—there's no reason to treat Sunday differently from any other day. The streak should increment whenever exactly one day has passed since the last listen, period.

**Fix and side-effect check:**
- Changed line 73 from `elif days_since_last == 1 and today.weekday() != 6:` to `elif days_since_last == 1:`
- This removes the incorrect weekday check entirely
- Verified: tested with Sunday date and confirmed streak now increments properly
- Checked related functionality: tested with other days of week, weekends, multi-day gaps — all work correctly
- Ran existing tests to ensure no regression

**Commit:**
```
fix: remove incorrect Sunday check in streak increment logic

The condition `today.weekday() != 6` on line 73 prevented streak from
incrementing on Sundays. Weekday should not be a factor in streak
incrementing logic—only the day count matters.
```

---

### Issue #2: Friends Listening Now shows people from yesterday (feed_service.py)

**How I reproduced it:**
1. Used seed_data to create users and listening events
2. Manually created a ListeningEvent dated yesterday evening using Flask shell
3. Set current time to next morning
4. Called `get_friends_listening_now()` with the current user
5. Confirmed: the yesterday evening event still appeared in results

**How I found the root cause:**
- Read `feed_service.py` lines 16-62
- Noticed `RECENT_THRESHOLD = timedelta(hours=24)` on line 13
- Traced the query logic: `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`
- Filter on line 42: `ListeningEvent.listened_at >= cutoff`
- Realized: a "24-hour window" is NOT the same as "today only"
- Example: If now is Sunday 9 AM and cutoff is Saturday 9 AM, an event from Saturday 11 PM passes the `>= cutoff` check and appears in results
- The function's docstring says "have listened to something recently" and "ordered by most recent first", but the issue reporter expected "only friends who have listened today"

**The root cause:**
The `get_friends_listening_now()` function uses a 24-hour time window (`RECENT_THRESHOLD = timedelta(hours=24)`) to filter recent listening events. However, "24 hours ago" is not equivalent to "since the start of today". A friend's event from yesterday evening (11 PM) is within 24 hours of this morning (9 AM), so it passes the filter. The function should check if the listening event occurred on today's calendar day, not within an arbitrary 24-hour window. This creates the reported behavior where yesterday's activities show up until the same time the next day.

**Fix and side-effect check:**
- Changed the filter logic on lines 32-46 to check calendar dates instead of time windows
- New code: check `last_listened.date() == today.date()` instead of comparing UTC timestamps to a 24-hour cutoff
- Verified: tested with events from yesterday, this morning, and later — only today's events appear
- Checked `get_activity_feed()` (line 65) which intentionally doesn't filter by recency — still works
- Ran test suite to ensure no unintended side effects

**Commit:**
```
fix: filter friends listening now by calendar date, not 24-hour window

Changed from time-based window (hours=24) to calendar-date comparison.
A friend's event from yesterday evening now correctly doesn't appear
the next morning since it's not "today".
```

---

### Issue #3: The same song keeps showing up twice in search (search_service.py)

**How I reproduced it:**
1. Searched for "Anthem" using the `/search?q=Anthem` endpoint
2. Found that songs with multiple tags appeared multiple times
3. Verified in seed_data: songs can have multiple tags
4. Confirmed: a song with 3 tags appears 3 times in results

**How I found the root cause:**
- Read `search_service.py` lines 11-37
- Noticed line 27: `.outerjoin(song_tags, Song.id == song_tags.c.song_id)`
- Understood: an outer join on song_tags produces one row for each tag
- If a song has N tags, the query returns N identical Song rows
- Line 37 converts all rows to dicts without deduplicating: `[song.to_dict() for song in results]`
- The join was unnecessary—the Song model already has a tags relationship defined in models.py

**The root cause:**
The `search_songs()` function performs an outer join with the `song_tags` table to fetch songs. When a song has multiple tags, the SQL join produces one row per tag (all with identical song data). The function then converts each row to a dict without deduplication, resulting in the same song appearing multiple times. The join is unnecessary since songs have a `tags` relationship already defined via SQLAlchemy; the `song.to_dict()` method already includes tags from the relationship.

**Fix and side-effect check:**
- Removed the `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` line (line 27)
- The query now only selects from Song and lets SQLAlchemy fetch tags through the relationship
- Verified: searched for various terms, confirmed each song appears exactly once
- Verified tags are still included in the response (via song.to_dict() which includes them from the relationship)
- Ran test suite to ensure search still works correctly

**Commit:**
```
fix: remove unnecessary outer join causing duplicate search results

The outerjoin with song_tags produced duplicate rows when a song had
multiple tags. Since songs have a tags relationship, the join is
unnecessary—SQLAlchemy loads tags automatically via the relationship.
```

---

### Issue #4: I got notified when a friend added my song to a playlist but not when they rated it (notification_service.py)

**How I reproduced it:**
1. Called `add_to_playlist()` with test data — verified notification was created
2. Called `rate_song()` with same test user and different song — verified NO notification was created
3. Confirmed rating was saved in database
4. Checked notification table — no entry for the rating

**How I found the root cause:**
- Read `notification_service.py` lines 35-70 (`add_to_playlist()`)
- Compared to lines 73-110 (`rate_song()`)
- `add_to_playlist()` calls `create_notification()` on lines 66-70 to notify the song's sharer
- `rate_song()` saves the rating but never calls `create_notification()`
- The pattern is clear: when someone interacts with a shared song, notify the original sharer
- This pattern is implemented for `add_to_playlist()` but missing from `rate_song()`

**The root cause:**
The `rate_song()` function saves a rating but doesn't create a notification for the song's original sharer, unlike the `add_to_playlist()` function which does create one. The architectural pattern in the file shows that whenever a friend interacts with a user's shared song (adding to playlist), a notification should be created. The `rate_song()` function is missing this notification creation step. The rating is saved successfully, but the song's sharer never learns about the rating.

**Fix and side-effect check:**
- Added notification creation logic to `rate_song()` after the rating is committed (after line 108)
- New code mirrors the pattern from `add_to_playlist()`: checks if rater is not the song's sharer, then creates notification
- Verified: called `rate_song()` and confirmed notification is created
- Verified: rating is still saved correctly
- Verified: notification message is clear ("kenji rated your song...")
- Ran test suite to ensure no side effects

**Commit:**
```
fix: create notification when a friend rates your song

rate_song() now creates a notification for the song's sharer,
mirroring the pattern used in add_to_playlist(). Previously,
ratings were saved but never notified.
```

---

### Issue #5: The last song in a playlist never shows up (playlist_service.py)

**How I reproduced it:**
1. Created a playlist with multiple songs
2. Called `get_playlist_songs()` endpoint
3. Confirmed: returned count was one less than expected
4. Identified: the missing song was always the most recently added one
5. Added another song — the previously missing song appeared, but the new one was now missing
6. This proved the bug: the last song is systematically excluded

**How I found the root cause:**
- Read `playlist_service.py` lines 38-67
- Focused on line 66: `return [song.to_dict() for song in songs[:-1]]`
- Recognized the Python slice `[:-1]` excludes the last element
- Line 64: `.all()` fetches all songs from the database
- Line 66 then removes the last one before returning
- This is clearly a bug — there's no legitimate reason to exclude the last song

**The root cause:**
The `get_playlist_songs()` function returns `songs[:-1]` on line 66, which excludes the last element of the songs list. This is a simple slicing bug. Every playlist will be missing its most recently added song. The fix is to return all songs without the slice.

**Fix and side-effect check:**
- Changed line 66 from `return [song.to_dict() for song in songs[:-1]]` to `return [song.to_dict() for song in songs]`
- Verified: playlists now return all songs, including the most recent
- Verified: songs are still returned in the correct order (ascending by position)
- Verified: adding a new song now shows it in results
- Ran test suite to ensure no side effects

**Commit:**
```
fix: return all songs in playlist, not all except the last

Removed [:-1] slice that was excluding the final song from results.
Playlists now correctly return all songs including the most recently added.
```

---

## Summary

All five bugs have been identified and fixed:
- ✅ Issue #1: Removed incorrect weekday check in streak logic
- ✅ Issue #2: Changed from 24-hour window to calendar-date filtering  
- ✅ Issue #3: Removed unnecessary join causing duplicates
- ✅ Issue #4: Added missing notification creation
- ✅ Issue #5: Removed slice excluding last song

Each fix is minimal and targeted, addressing only the root cause without restructuring unrelated code.
