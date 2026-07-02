# submission.md

## Milestone 1 Notes

### `streak_service.py`

`record_listening_event(user_id, song_id)` — the entry point. Given a user and a song, it:

- Looks up the user (raises if not found)
- Creates a ListeningEvent row with the current UTC timestamp
- Calls update_listening_streak() to update the user's streak
- Commits everything to the DB and returns the event

`update_listening_streak(user, now)` — the actual streak logic. Per the docstring, the intended rules are:

- No prior listen → streak = 1
- Already listened today → no change
- Listened yesterday → streak += 1
- Gap of more than a day → streak resets to 1
  Computes days_since_last as the difference in calendar dates between now and user.last_listened_at, then branches on that:

== 0 → no-op
== 1 and today.weekday() != 6 → increment
else → reset to 1

`get_streak(user_id)` — a simple read accessor, just returns user.listening_streak.

### `feed_sercice.py`

handles two related but distinct feed features: a "who's listening right now" snapshot and a general activity log

`feed_service.py` handles two related but distinct feed features: a "who's listening right now" snapshot and a general activity log.

**Module-level constant:**
`RECENT_THRESHOLD = timedelta(hours=24)` — defines how far back "listening now" looks.

**`get_friends_listening_now(user_id)`**
Builds the "Friends Listening Now" feed — a snapshot of what each friend is _currently/recently_ playing.

1. Looks up the user (raises if not found)
2. Computes a cutoff time 24 hours ago
3. Gets the user's friend IDs; returns `[]` immediately if there are none
4. Queries all `ListeningEvent` rows belonging to those friends where `listened_at >= cutoff`, ordered newest-first
5. Walks through those events and **deduplicates by friend** — using a `seen_friends` set, it keeps only the first (i.e. most recent, since the query is already sorted descending) event per friend
6. Returns a list of dicts: `{friend, song, listened_at}`, one entry per friend, showing only their latest recent song

**`get_activity_feed(user_id, limit=20)`**
Builds a general activity feed — a rolling log of recent listens across all friends, explicitly _not_ deduplicated and _not_ time-windowed.

1. Looks up the user (raises if not found)
2. Gets friend IDs; returns `[]` if none
3. Queries the most recent `limit` `ListeningEvent` rows across all friends (no cutoff filter at all — could include events from months ago if a friend hasn't been active)
4. Returns each event as a `{friend, song, listened_at}` dict, one per event — so the same friend can appear multiple times if they logged several listens

**Key difference between the two:** `get_friends_listening_now` is time-bounded (24h) and deduplicated (one entry per friend, latest only); `get_activity_feed` is count-bounded (top N) and un-deduplicated (every event, same friend can repeat).

### `notification_service.py`

`notification_service.py` handles creating/reading notifications, and also — a bit unexpectedly — owns two pieces of unrelated business logic (playlist additions and song ratings) that happen to trigger notifications.

**`create_notification(user_id, notification_type, body)`**
The low-level building block. Just constructs a `Notification` row with a type and message body, saves it, and returns it. Every other notification-producing function in this file calls into this one rather than constructing `Notification` objects directly.

**`add_to_playlist(playlist_id, song_id, added_by_user_id)`**
Handles the "add a song to a playlist" action:

1. Looks up the song, the adder, and the playlist (raises if any are missing)
2. Appends the song to the playlist's songs (skips if it's already there), commits
3. If the person adding the song isn't the same person who originally shared it, calls `create_notification()` to notify the original sharer, with a body like `"{adder} added your song '{title}' to the playlist '{name}'."`

**`rate_song(user_id, song_id, score)`**
Handles submitting/updating a rating:

1. Validates `score` is 1–5
2. Looks up the song and rater (raises if missing)
3. Checks for an existing `Rating` by that user for that song — updates its score if found, otherwise creates a new `Rating`
4. Commits and returns the rating

**`get_notifications(user_id, unread_only=False)`**
Read accessor — queries all notifications for a user, optionally filtered to `read=False`, ordered newest-first, and returns them as dicts via `to_dict()`.

**`mark_as_read(notification_id)`**
Looks up a notification by ID (raises if missing), sets `read = True`, commits.

### `playlist_service.py`

`playlist_service.py` handles creating playlists and reading back their data — playlist metadata, song contents, and per-user playlist lists.

**`create_playlist(name, created_by_user_id, is_collaborative=True)`**

1. Looks up the creating user (raises if not found)
2. Constructs a new `Playlist` row with the given name, creator, and collaborative flag
3. Commits and returns it

**`get_playlist_songs(playlist_id)`**
Meant to return the full ordered list of songs in a playlist:

1. Looks up the playlist (raises if not found)
2. Joins `Song` against the `playlist_entries` association table, filters to this playlist, and orders ascending by `position` — so songs come back in the order they were added
3. Converts to dicts and returns

**`get_playlist(playlist_id)`**
Read accessor for playlist metadata only (name, creator, collaborative flag, etc.) — no song list. Looks up the playlist, raises if missing, returns `to_dict()`.

**`get_user_playlists(user_id)`**
Returns all playlists a given user created, as a list of dicts. No existence check on the user — an unknown `user_id` just yields an empty list rather than an error, which is a bit inconsistent with the other functions in this file that all raise `ValueError` on a missing user/playlist.

### `search_service.py`

`search_service.py` handles finding songs by keyword and looking up a single song by ID.

**`search_songs(query)`**

1. Builds a query joining `Song` to `song_tags` via an `outerjoin` (so songs come along even if they have no tags)
2. Filters to songs where `title` or `artist` case-insensitively contains the query string (`ilike`)
3. Returns `.to_dict()` for each matching song

**`get_song(song_id)`**
Straightforward lookup by primary key — raises `ValueError` if the song doesn't exist, otherwise returns `song.to_dict()`.
