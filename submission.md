# Project 5: Mixtape Bug Hunt

## AI Usage

I used Codex to trace service calls and review fixes. I verified its suggestions with the code and remote tests. For Issue 3, I rejected its first assumption and checked the duplicate rows directly.

## Codebase Map

- `app.py`: creates the Flask app, configures SQLAlchemy, and registers route blueprints.
- `models.py`: defines users, songs, tags, listening events, ratings, playlists, notifications, friendships, and association tables.
- `routes/`: parses HTTP requests, calls service functions, and formats JSON responses.
- `services/`: contains streak, feed, search, notification, and playlist business logic.
- `seed_data.py`: creates representative users, songs, tags, listening events, playlists, and notifications.
- `tests/`: exercises the required streak, search, and playlist behaviors.

### Data Flow: Recording a Listen

`POST /songs/<song_id>/listen` reaches `routes/songs.py`, which validates `user_id` and calls `services.streak_service.record_listening_event()`. The service creates a `ListeningEvent`, calls `update_listening_streak()` to update the related `User`, commits both changes through SQLAlchemy, and returns the event for the route's JSON response.

## Issue 1: Listening Streak Resets on Sunday

### How I Reproduced It

I ran `tests/test_streaks.py::test_streak_increments_on_sunday`. The test recorded a listen on Saturday and another on Sunday. The expected streak was 2, but the service returned 1.

### How I Found the Root Cause

I followed the listen route in `routes/songs.py` to `record_listening_event()` and then `update_listening_streak()` in `services/streak_service.py`. The docstring says every consecutive calendar day increments the streak, but the consecutive-day branch included an extra Sunday exclusion.

### Root Cause

`days_since_last == 1 and today.weekday() != 6` treated Sunday as a reset even when the previous listen was exactly one day earlier. The weekday condition contradicted the consecutive-day rule.

### Fix and Side-Effect Check

I removed the weekday condition so every one-day gap increments the streak. I checked new-user, same-day, ordinary consecutive-day, skipped-day, and Saturday-to-Sunday behavior.

## Issue 3: Duplicate Songs in Search Results

### How I Reproduced It

`Crown Heights Anthem` has three tags, and the join returned its song ID three times. SQLAlchemy returned one `Song` object, so the supplied test passed.

### How I Found the Root Cause

`GET /songs/search` calls `search_songs()`. Its tag join returned one row per tag.

### Root Cause

The query did not request unique songs after joining `song_tags`.

### Fix and Side-Effect Check

I added `DISTINCT` and checked songs with zero, one, and multiple tags, plus a search with no match.

## Issue 5: Last Playlist Song Is Missing

### How I Reproduced It

I ran the playlist tests with a five-song playlist. `get_playlist_songs()` returned only four songs, and the ordered titles stopped at `Track 4` instead of including `Track 5`.

### How I Found the Root Cause

I followed `GET /playlists/<playlist_id>/songs` in `routes/playlists.py` to `get_playlist_songs()` in `services/playlist_service.py`. The database query returned the songs in the correct position order, but the return expression sliced the result before serialization.

### Root Cause

The expression `songs[:-1]` always removed the final query result. This also made a one-song playlist appear empty.

### Fix and Side-Effect Check

I serialized the complete `songs` list. I checked the five-song count, position order, and empty-playlist behavior.
