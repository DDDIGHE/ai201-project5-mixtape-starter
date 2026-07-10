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
