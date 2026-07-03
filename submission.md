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

```text
Searching for songs
GET /songs/search
→ routes/songs.py: search()
→ services/search_service.py: search_songs()
→ models.py: Song, Tag, song_tags
→ tests/test_search.py
```

```text
Viewing playlist songs
GET /playlists/<id>/songs
→ routes/playlists.py: get_songs()
→ services/playlist_service.py: get_playlist_songs()
→ models.py: Playlist, Song, playlist_entries
→ tests/test_playlists.py
```

```text
Recording a listen and updating a streak
POST /songs/<id>/listen
→ routes/songs.py: listen()
→ services/streak_service.py: record_listening_event()
→ services/streak_service.py: update_listening_streak()
→ models.py: User, ListeningEvent
→ tests/test_streaks.py
```

```text
Viewing friends listening now
GET /feed/<user_id>/listening-now
→ routes/feed.py: listening_now()
→ services/feed_service.py: get_friends_listening_now()
→ models.py: User, ListeningEvent, Song, friendships
```

```text
Rating a song
POST /songs/<id>/rate
→ routes/songs.py: rate()
→ services/notification_service.py: rate_song()
→ models.py: Rating, Song, User, Notification
```


## Feature Data Flow: Adding A Song To A Playlist Creates A Notification

This app does not currently have a dedicated "share song" route, but songs have a `shared_by` owner. The clearest notification flow is when one user adds another user's shared song to a playlist:

```text
POST /playlists/<playlist_id>/songs
    ↓
routes/playlists.py: add_song()
    - Reads JSON body fields: song_id and added_by
    - Calls add_to_playlist(playlist_id, song_id, added_by)
    ↓
services/notification_service.py: add_to_playlist()
    - Loads the Song, User who added it, and Playlist
    - Appends the song to playlist.songs if it is not already present
    - Commits the playlist change
    - If the adder is not the original sharer, creates a notification
    ↓
services/notification_service.py: create_notification()
    - Inserts a Notification row for song.shared_by
    - Uses notification_type = "song_added_to_playlist"
    - Commits the notification
    ↓
GET /users/<shared_by_user_id>/notifications
    ↓
routes/users.py: notifications()
    ↓
services/notification_service.py: get_notifications()
    - Returns notifications ordered newest first
```

The key data relationship is:

```text
Song.shared_by
    points to the user who originally shared the song

Notification.user_id
    points to the user who should receive the notification
```


## Bugs Fixes
### My listening streak keeps resetting

**Issue number and title:** Issue 1: My listening streak keeps resetting

**How I reproduced it:** I reproduced it by running the streak tests:

```bash
.venv/bin/python3 -m pytest tests/test_streaks.py -q
```

The test file had one failure:

```text
tests/test_streaks.py::test_streak_increments_on_sunday
```

That test creates a user, records a listen on Saturday, June 15, 2024, and then records another listen on Sunday, June 16, 2024. Since those dates are consecutive calendar days, the user's streak should increase from `1` to `2`. Instead, the streak stayed at `1`, which confirmed the bug.

**How I found the root cause:** I started from the failing test in `tests/test_streaks.py`, specifically `test_streak_increments_on_sunday`. That test calls `update_listening_streak(u, saturday)` and then `update_listening_streak(u, sunday)`, so I followed that function into `services/streak_service.py`.

Inside `update_listening_streak`, I looked at the code that calculates `days_since_last` and decides whether to increment or reset the streak. This condition made me confident I had found the exact cause:

```python
elif days_since_last == 1 and today.weekday() != 6:
```

The failure only happened for Sunday, and in Python, `weekday()` returns `6` for Sunday. That connected the failing input directly to the condition.

**The root cause:** The root cause is in the `update_listening_streak` function in `services/streak_service.py`. The specific faulty condition is:

```python
elif days_since_last == 1 and today.weekday() != 6:
```

The variable `days_since_last` correctly identifies that Saturday to Sunday is a one-day gap, so that part of the logic is working. The problem is the second comparison, `today.weekday() != 6`. In Python, Sunday is represented by `6`, so this condition says: increment the streak only if the user listened yesterday and today is not Sunday.

When a user listens on Saturday and then Sunday, the state is:

```text
days_since_last == 1
today.weekday() == 6
```

Because `today.weekday() != 6` is false on Sunday, the whole `elif` condition fails. The function then falls into the `else` branch and sets `user.listening_streak = 1`, which resets the streak instead of incrementing it.

The correct behavior requires something different because the app's streak rule is based on consecutive calendar days, not on the weekday name. If the gap is exactly one day, the streak should increment no matter whether the current day is Monday, Sunday, or any other day. The Sunday comparison adds an unrelated rule that contradicts the intended streak behavior.

**Fix and side-effect check:** The fix is to remove the Sunday exception and only check whether the user listened exactly one calendar day after their previous listen.

Change this:

```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
```

To this:

```python
elif days_since_last == 1:
    user.listening_streak += 1
```

This fixes the root cause because the streak now increments for every consecutive-day listen, including Saturday to Sunday.

The side-effect check is that the other streak behaviors should still pass: new users start at `1`, same-day listens do not double-count, normal consecutive weekdays increment the streak, skipped days reset the streak to `1`, and Saturday-to-Sunday now increments correctly.

### The same song keeps showing up twice in search

**Issue number and title:** Issue 3: The same song keeps showing up twice in search

**How I reproduced it:** I used the search tests because they already create the exact app state that triggers this issue: a song with multiple tags. The key test is `test_search_no_duplicates_multi_tag_song` in `tests/test_search.py`.

The test data creates a song called `"Crown Heights Anthem"` and connects it to three tags:

```text
rap
hip-hop
boom bap
```

Then it searches for `"Crown Heights"`:

```python
results = search_songs("Crown Heights")
matching = [r for r in results if r["title"] == "Crown Heights Anthem"]
```

The expected result is that `"Crown Heights Anthem"` appears once, even though it has multiple tags. The bug condition is a search result where the matching song has more than one row in the `song_tags` association table.

**How I found the root cause:** I started from `tests/test_search.py`, because that file describes the expected behavior for search results. The multi-tag test pointed me to the `search_songs` function in `services/search_service.py`.

Inside `search_songs`, I found that the query loads `Song` records but also joins through the `song_tags` table:

```python
db.session.query(Song)
    .outerjoin(song_tags, Song.id == song_tags.c.song_id)
```

That made me confident I had found the right area because the reported behavior only happens for songs with multiple tags, and `song_tags` is the table that creates one row per song/tag pair. The specific problem was that the query did not explicitly ask for distinct songs after joining through a table that can contain multiple rows for the same song.

**The root cause:** The root cause is in the `search_songs` function in `services/search_service.py`. The function joins `Song` to `song_tags`, but the query originally did not include a distinct result constraint.

The specific logic problem is this:

```text
One song can have multiple tag rows in song_tags.
The search query joins against song_tags.
Without distinct song results, the same Song can be returned once per matching joined row.
```

For example, `"Crown Heights Anthem"` has three tags. That means the join can produce three SQL rows for the same song ID. Search results should be based on unique songs, not on unique song/tag join rows. The correct behavior requires deduplicating by song because tags are extra metadata on a song, not separate search results.

**Fix and side-effect check:** I changed the query in `services/search_service.py` to explicitly return distinct songs:

```python
.distinct()
```

The fixed query now keeps the existing title/artist search behavior but prevents a multi-tag song from appearing more than once in the returned list.

I also removed the unused `Tag` import from `services/search_service.py`, since the service only needs `Song` and `song_tags`.

After the fix, I ran the focused search tests:

```bash
.venv/bin/python -m pytest tests/test_search.py -q
```

The result was:

```text
5 passed
```

For the side-effect check, I also ran the full test suite:

```bash
.venv/bin/python -m pytest tests/ -q
```

The search and streak tests passed. The remaining failures were in the playlist tests, which are for a separate issue about the last song in a playlist not showing up. That confirmed the search change did not break the existing search behavior.
