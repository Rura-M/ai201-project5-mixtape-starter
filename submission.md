# Bug Fixes

## Codebase Map
The code has 5 blueprints used in its functionality which are registered in the app.py file.
The models.py defines the database structure and its relations.

## Database Model

`models.py` contains this schema:

- `User`: account/profile data, listening streak fields, friendships, notifications, playlists.
- `Song`: shared track metadata, the sharing user, ratings, listening events, tags.
- `Tag`: song labels connected to songs through `song_tags`.
- `ListeningEvent`: record of a user listening to a song at a time.
- `Rating`: one rating per user/song pair, enforced by a unique constraint.
- `Playlist`: playlist metadata connected to songs through `playlist_entries`.
- `Notification`: messages created when friends interact with shared songs.

Association tables:

- `friendships`: user-to-user many-to-many relationship.
- `song_tags`: song-to-tag many-to-many relationship.
- `playlist_entries`: playlist-to-song many-to-many relationship with `position`, `added_by`, and `added_at`.


## Routes

### Songs: `routes/songs.py`

- `GET /songs/search?q=...`
  - Calls `services.search_service.search_songs`.
  - Searches songs by title or artist.
- `GET /songs/<song_id>`
  - Calls `services.search_service.get_song`.
  - Returns one song or 404.
- `POST /songs/<song_id>/rate`
  - Calls `services.notification_service.rate_song`.
  - Creates or updates a rating.
- `POST /songs/<song_id>/listen`
  - Calls `services.streak_service.record_listening_event`.
  - Creates a listening event and updates the user's streak.

### Playlists: `routes/playlists.py`

- `POST /playlists/`
  - Calls `services.playlist_service.create_playlist`.
  - Creates playlist metadata.
- `GET /playlists/<playlist_id>`
  - Calls `services.playlist_service.get_playlist`.
  - Returns playlist metadata without songs.
- `GET /playlists/<playlist_id>/songs`
  - Calls `services.playlist_service.get_playlist_songs`.
  - Returns songs ordered by playlist position.
- `POST /playlists/<playlist_id>/songs`
  - Calls `services.notification_service.add_to_playlist`.
  - Adds a song and may notify the original sharer.

### Users: `routes/users.py`

- `GET /users/<user_id>`
  - Reads `User` directly.
- `GET /users/<user_id>/streak`
  - Calls `services.streak_service.get_streak`.
- `GET /users/<user_id>/notifications?unread_only=true|false`
  - Calls `services.notification_service.get_notifications`.
- `POST /users/notifications/<notification_id>/read`
  - Calls `services.notification_service.mark_as_read`.

### Feed: `routes/feed.py`

- `GET /feed/<user_id>/listening-now`
  - Calls `services.feed_service.get_friends_listening_now`.
  - Shows the most recent currently-listening entry per friend.
- `GET /feed/<user_id>/activity`
  - Calls `services.feed_service.get_activity_feed`.
  - Shows recent friend listening activity.

## Services

- `services/search_service.py`
  - `search_songs(query)`: searches songs by title/artist.
  - `get_song(song_id)`: returns one song dict.
- `services/playlist_service.py`
  - `create_playlist(...)`: validates creator and creates a playlist.
  - `get_playlist_songs(playlist_id)`: returns ordered playlist songs.
  - `get_playlist(playlist_id)`: returns playlist metadata.
  - `get_user_playlists(user_id)`: returns playlists created by a user.
- `services/streak_service.py`
  - `record_listening_event(user_id, song_id)`: creates a listening event and updates streak.
  - `update_listening_streak(user, now)`: applies streak rules.
  - `get_streak(user_id)`: reads a user's streak count.
- `services/feed_service.py`
  - `get_friends_listening_now(user_id)`: finds recent listening events from friends and deduplicates by friend.
  - `get_activity_feed(user_id, limit=20)`: returns friend listening history.
- `services/notification_service.py`
  - `create_notification(...)`: inserts a notification.
  - `add_to_playlist(...)`: adds a song and notifies the song sharer.
  - `rate_song(...)`: creates or updates a rating.
  - `get_notifications(...)`: returns notifications.
  - `mark_as_read(...)`: marks one notification read.