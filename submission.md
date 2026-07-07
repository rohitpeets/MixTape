## app.py
This file is responsible for creating a Flask App instance and creating a database using SQLAlchemy ( Which uses python objects to represent a Table/ORM).
In this file,we also:
Configure the App.
Connect the database.
map the routes and blureprints for the services.
## models.py

This file defines the SQLAlchemy models (tables) for all data in the app, plus their relationships and serialization.

** Association Tables (Used for linking tables) for:
- friendships (User - User)
- song_tags (Song - Tag)
- playlist_entries (Playlist - Song)

** User Model
- id, username, email
- listening_streak
- last_listened_at
- created_at

** Relationships (With back-referencing)
- shared_songs -> Song (backref: shared_by_user)
- ratings -> Rating (backref: rater)
- listening_events -> ListeningEvent (backref: listener)
- notifications -> Notification (backref: recipient)
- playlists -> Playlist (backref: creator)
- friends -> User (self-referential)

** Tag Model
- id, name (unique)

** Song Model
- id, title, artist, album, genre
- shared_by, shared_at, share_note
- Sub Relationships: ratings, listening_events, tags (through song_tags)

** ListeningEvent Model
- id, user_id, song_id, listened_at
- one row per "user listened to song" event

** Rating Model
- id, user_id, song_id, score (1-5)
- UniqueConstraint on (user_id, song_id) -> one rating per user per song

** Playlist Model
- id, name, created_by, created_at, is_collaborative
- songs -> Song (through playlist_entries) -- NOT ordered by position automatically

** Notification Model
- id, user_id, notification_type, body, created_at, read

** to_dict() methods
- Every model has one, used to serialize for JSON API responses
## Data Flow Trace
## Routes/

** The Feed_bp manages two Routes: **

- @feed_bp.route("/<user_id>/listening-now")
calls for get_friends_listening_now(user_id) from the services.feed_service.py to return a feed object containing a list of friends who have listened to something recently along with the song they are listening too.
The get_friends_listening_now(user_id) has checks for valid users,valid friends,and arranges the elements in a given order before returning them.
The feed object is jasonified and presented to the user.

- @feed_bp.route("/<user_id>/activity")
calls for get_activity_feed(user_id) to return a list of general listening activity from all friends which is converted to json to present to the user.

**  The playlists_bp manages 4 Routes: **

- @playlists_bp.route("/", methods=["POST"])
Executes a create() function that reads name, created_by, and an optional is_collaborative flag from the JSON body. If name or created_by is missing, it returns a 400 error immediately without calling the service. Otherwise it calls create_playlist() from services.playlist_service.py, which checks the user exists before inserting the new playlist. The created playlist is jsonified and returned with a 201 status.

- @playlists_bp.route("/<playlist_id>")
Executes a get_detail() function that calls get_playlist(playlist_id) from services.playlist_service.py to fetch the playlist's metadata only (no songs included). Returns the playlist as json, or a 404 error if the playlist doesn't exist.

- @playlists_bp.route("/<playlist_id>/songs")
Executes a get_songs() function that calls get_playlist_songs(playlist_id) from services.playlist_service.py to fetch the ordered list of songs in the playlist. Returns the songs plus a count as json, or a 404 error if the playlist doesn't exist.

- @playlists_bp.route("/<playlist_id>/songs", methods=["POST"])
Executes an add_song() function that reads song_id and added_by from the JSON body. If either is missing, it returns a 400 error. Otherwise it calls add_to_playlist() — notably imported from services.notification_service.py instead of playlist_service.py, since this function both adds the song to the playlist and notifies the song's original sharer. Returns a success message with a 201 status.

** The songs_bp manages 4 Routes: **

- @songs_bp.route("/search")
Executes a search() function that reads the query string parameter q. If q is missing or empty, it returns a 400 error immediately. Otherwise it calls search_songs(query) from services.search_service.py, which matches songs by title or artist (case-insensitive) and returns them along with their tags. The results plus a count are jsonified and returned.

- @songs_bp.route("/<song_id>")
Executes a get_song_detail() function that calls get_song(song_id) from services.search_service.py to fetch a single song by its ID. Returns the song as json, or a 404 error if the song doesn't exist.

- @songs_bp.route("/<song_id>/rate", methods=["POST"])
Executes a rate() function that reads user_id and score from the JSON body. If either is missing, it returns a 400 error. Otherwise it calls rate_song() — imported from services.notification_service.py rather than a dedicated ratings service — which validates the score is between 1 and 5, then creates a new rating or updates an existing one for that user/song pair. Returns the rating as json with a 201 status.

- @songs_bp.route("/<song_id>/listen", methods=["POST"])
Executes a listen() function that reads user_id from the JSON body. If missing, it returns a 400 error. Otherwise it calls record_listening_event() from services.streak_service.py, which logs the listening event and updates the user's listening streak. Returns the event as json with a 201 status.

** The users_bp manages 4 Routes: **

- @users_bp.route("/<user_id>")
Executes a get_user() function that queries the User model directly with db.session.get() — no service layer involved here, unlike the other routes. If no user is found, returns a 404 error. Otherwise returns the user as json via to_dict().

- @users_bp.route("/<user_id>/streak")
Executes a streak() function that calls get_streak(user_id) from services.streak_service.py to fetch the user's current listening streak. Returns the user_id and streak count as json, or a 404 error if the user doesn't exist.

- @users_bp.route("/<user_id>/notifications")
Executes a notifications() function that reads an optional unread_only query string parameter (defaults to "false", compared case-insensitively). It calls get_notifications() from services.notification_service.py, passing that flag through, to fetch either all notifications or just unread ones for the user. Returns the notifications plus a count as json, or a 404 error if the user doesn't exist.

- @users_bp.route("/notifications/<notification_id>/read", methods=["POST"])
Executes a read_notification() function that calls mark_as_read(notification_id) from services.notification_service.py to flip a notification's read flag to true. Returns a success message as json, or a 404 error if the notification doesn't exist.

## pattern noticed :
every route delegates to a service function, routes just do parsing/formatting.

** Solving bugs **

## Reproduce And Fix
## Issue #2: Last song in playlist never shows up

**File:** services/playlist_service.py

**How I reproduced it:**
playlist_entries for "Late Night Vibes" (b7df8851-e583-439b-b6f4-afd4dbec1ca1)
has 7 rows, positions 1-7, song e8c25e28-c464-4639-aa97-8adbd3cf7dc9 at
position 7. GET /playlists/b7df8851.../songs returned count: 6, position 7
missing.

**How I found the root cause:**
Opened get_playlist_songs() (README named this file). Query was correct —
join, filter by playlist_id, order by position. Found it in the return
line: `return [song.to_dict() for song in songs[:-1]]`.

**The root cause:**
Query returns songs ordered by position correctly, but `[:-1]` drops the
last item — so the highest-position song is always excluded, regardless of
playlist length.

**My fix and side-effect check:**
Removed `[:-1]`, return `songs`. Reran GET — count 7, song e8c25e28-... now
appears last, in order. Checked get_playlist() and add_to_playlist() —
neither depends on the truncated list. Committed as 6491a87.


## Issue #4: Friends Listening Now shows people from yesterday

**File:** services/feed_service.py

**How I reproduced it:**
nova's friends all had listening events from 07-05/07-06, older than 24h
vs real time (2026-07-07 11:31). GET /feed/60233a52.../listening-now
returned count: 0. Inserted a ListeningEvent for kenji (14577a2b-...)
timestamped 20h ago — kenji then appeared in the feed.

**How I found the root cause:**
Read get_friends_listening_now() — query and timestamp comparisons were
correct. Found it in the constant: `RECENT_THRESHOLD = timedelta(hours=24)`.

**The root cause:**
Filter logic is correct, but the threshold is 24 hours — too wide for
"listening now." Anything from the entire previous day still counts as
current.

**My fix and side-effect check:**
Changed RECENT_THRESHOLD to timedelta(minutes=30). Reran test — kenji's
20h-old event disappeared (count: 0). Inserted a fresh event for kenji —
he reappeared with the new timestamp, confirming the filter works both
ways. get_activity_feed() doesn't use this constant — unaffected.
Committed separately.


## Issue #5: No notification when a friend rates my song

**File:** services/notification_service.py

**How I reproduced it:**
nova had 1 notification (song_added_to_playlist) before any action. darius
POSTed a 5-star rating on nova's song "Midnight Drive" via
/songs/b7dca080.../rate — rating saved successfully. Checked nova's
notifications after — still 1, no new entry.

**How I found the root cause:**
Compared add_to_playlist() (works) to rate_song() (doesn't) line by line,
per the project hint. add_to_playlist() calls create_notification()
targeting song.shared_by after its commit. rate_song() commits and
returns — no create_notification() call anywhere in it.

**The root cause:**
rate_song() never calls create_notification() — never implemented, not
broken logic. The rating saves correctly; it's just not wired to the
notification system.

**My fix and side-effect check:**
Added create_notification() in rate_song() after the commit, same
shared_by guard as add_to_playlist(), type "song_rated". Reran test —
darius rating nova's song produced a new notification: count 1 → 2, entry
reads "darius rated your song 'Midnight Drive' 4 stars." Rating itself
still updates correctly (same id, no duplicate). get_notifications() and
mark_as_read() unchanged.