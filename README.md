# Mixtape

A Flask social music app where friends share songs, build collaborative playlists, and track listening stats.

I picked up this codebase with five known bugs in its service layer, traced each one from its route to the failing service, and fixed three: the listening-streak reset boundary, duplicate songs in search, and the last song missing from playlists. The other two are still open. The commit history has the details of each fix.

## Known issues (3 of 5 fixed)

| # | Issue | Service | Status |
|---|-------|---------|--------|
| 1 | My listening streak keeps resetting | `streak_service.py` | Fixed |
| 2 | Friends Listening Now shows people from yesterday | `feed_service.py` | Open |
| 3 | The same song keeps showing up twice in search | `search_service.py` | Fixed |
| 4 | I got notified when a friend added my song to a playlist but not when they rated it | `notification_service.py` | Open |
| 5 | The last song in a playlist never shows up | `playlist_service.py` | Fixed |

## App structure

```
MixTape/
├── app.py                      # Flask app factory and DB setup
├── models.py                   # SQLAlchemy models for all entities
├── routes/
│   ├── songs.py                # Song sharing, search, and rating routes
│   ├── playlists.py            # Playlist creation and song management
│   ├── users.py                # User profiles, streaks, notifications
│   └── feed.py                 # Friends listening now, activity feed
├── services/
│   ├── streak_service.py       # Listening streak logic
│   ├── feed_service.py         # Friends listening now feed logic
│   ├── search_service.py       # Song search logic
│   ├── notification_service.py # Notification creation and retrieval
│   └── playlist_service.py     # Playlist retrieval logic
├── tests/
│   ├── test_streaks.py
│   ├── test_search.py
│   └── test_playlists.py
├── seed_data.py                # Populates DB with test data
├── requirements.txt
└── .gitignore
```

Routes call services, and the bugs live in `services/`. To debug a broken endpoint, follow it to the service it calls. For example:

- Rating a song: `POST /songs/<song_id>/rate` → `routes/songs.py` → `notification_service.rate_song()`
- Viewing a playlist: `GET /playlists/<id>/songs` → `routes/playlists.py` → `playlist_service.get_playlist_songs()`

## Setup

```bash
python -m venv .venv

# macOS / Linux
source .venv/bin/activate
# Windows (Command Prompt)
.venv\Scripts\activate.bat
# Windows (Git Bash)
source .venv/Scripts/activate

pip install -r requirements.txt
python seed_data.py              # seed the database with test data
FLASK_APP=app:create_app flask run
pytest tests/                    # run the tests
```

On macOS, if requests hang or get "connection refused", use `http://127.0.0.1:5000` instead of `http://localhost:5000`. `localhost` can resolve to an IPv6 address that Flask isn't listening on.
