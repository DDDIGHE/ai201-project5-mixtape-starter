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
