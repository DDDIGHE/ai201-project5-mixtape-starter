# Project 5: Mixtape Bug Hunt

## AI Usage

I used Codex to summarize call chains, compare service contracts with tests, and suggest candidate root causes. I verified each suggestion against the source code and remote test output before changing code. For Issue 3, the initial expectation was that the provided test would fail, but SQLAlchemy 2.0.51 deduplicated ORM entities; I then verified that the underlying join still produced duplicate rows before applying `DISTINCT`.

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

The seed data gives `Crown Heights Anthem` three tags. Running the search query's outer join for that title produced three database rows with the same song ID. SQLAlchemy 2.0.51 collapsed those rows when returning full `Song` entities, so the provided API-level test happened to pass in this environment; the underlying query still produced the duplicate rows reported by the issue.

### How I Found the Root Cause

I followed `GET /songs/search` in `routes/songs.py` to `search_songs()` in `services/search_service.py`. The query outer-joined `song_tags`, so a song produced one result row per matching tag association before ORM materialization.

### Root Cause

The search query joined a one-to-many association without requesting unique songs. Songs with multiple tags therefore appeared multiple times at the SQL row level, and the result depended on ORM entity deduplication rather than the query's contract.

### Fix and Side-Effect Check

I added `DISTINCT` before materializing the query. I checked searches for songs with zero, one, and multiple tags, plus a query with no match.

## Issue 5: Last Playlist Song Is Missing

### How I Reproduced It

I ran the playlist tests with a five-song playlist. `get_playlist_songs()` returned only four songs, and the ordered titles stopped at `Track 4` instead of including `Track 5`.

### How I Found the Root Cause

I followed `GET /playlists/<playlist_id>/songs` in `routes/playlists.py` to `get_playlist_songs()` in `services/playlist_service.py`. The database query returned the songs in the correct position order, but the return expression sliced the result before serialization.

### Root Cause

The expression `songs[:-1]` always removed the final query result. This also made a one-song playlist appear empty.

### Fix and Side-Effect Check

I serialized the complete `songs` list. I checked the five-song count, position order, and empty-playlist behavior.
