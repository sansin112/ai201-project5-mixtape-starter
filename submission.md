# Project 5 Submission: Mixtape Bug Hunt

## AI Usage Section
During this project, AI was utilized as a pair-programming partner focused heavily on codebase navigation, tracing data flows, and verifying edge-case logic rather than blindly generating blocks of code. 

* **Codebase Navigation & Tracing:** I used the AI to map out how the Flask application factory patterns interacted with SQLAlchemy's association tables (`playlist_entries` and `song_tags`). This helped trace the application flow from routes (e.g., `POST /songs/<song_id>/listen`) back into the underlying service files inside `services/`.
* **What it Helped Me Understand:** The AI clearly explained how the Python list slice syntax `[:-1]` in `playlist_service.py` was stripping away rows from query outputs and how many-to-many relationship joins without deduplication rules implicitly generate row duplication.
* **Verification & Discrepancies:** I manually ran `flask shell` loops to query records and check active data model values. At one point, the AI incorrectly assumed that the seed script was generating 8 songs per playlist, but I verified by reading `seed_data.py` and checking the browser JSON outputs that the seed script itself was initially hard-capped to populate exactly 7 rows. I had to manually use `flask shell` to dynamically inject an 8th song entity into the database to definitively prove that the slice boundary fix was operating flawlessly.

---

## Codebase Map
### Main Files and Roles
* `app.py`: Flask application factory that initializes configuration patterns, registers blueprints (`songs`, `playlists`, `users`, `feed`), sets up the SQLAlchemy instance (`db`), and triggers the generation of database tables via `db.create_all()`.
* `models.py`: Defines the database schemas using SQLAlchemy ORM entities (`User`, `Song`, `Tag`, `Playlist`, `ListeningEvent`, `Rating`, `Notification`) along with many-to-many join tables (`friendships`, `song_tags`, `playlist_entries`).
* `routes/`: Contains standard Flask routing decorators that extract request data, validate payload variables, handle errors, and delegate core application mechanics directly to backend services.
* `services/`: The core business logic layer where queries are executed and state mutations take place.

### Data Flow Example: Rating a Song
1. **Client Action:** A client triggers a `POST` request to `http://127.0.0.1:5000/songs/<song_id>/rate` passing a JSON payload with keys `user_id` and `score`.
2. **Routing Blueprint:** `routes/songs.py` catches the endpoint, extracts the parameters, validates that both keys are provided, and passes them cleanly to `services.notification_service.rate_song()`.
3. **Service Layer Execution:** `rate_song` queries the database to verify the `Song` and `User` entities exist, checks if a previous `Rating` record exists to perform an upsert/insert, writes a notification row into the database table, commits the transaction session via `db.session.commit()`, and returns the model instance to the blueprint for JSON serialization.

---

## Root Cause Analysis (RCA)

### Bug 1: Listening Streak Resetting on Sundays (`services/streak_service.py`)
1. **The symptoms you observed:** The user's listening streak would reset back down to `1` when logging consecutive listening actions on Sundays instead of incrementing.
2. **The exact root cause:** In `update_listening_streak()`, an explicit constraint condition `and today.weekday() != 6` was hardcoded onto the consecutive-day conditional branch check. Since Python handles weekday index offsets starting from Monday as `0`, day index `6` accurately targets Sunday, triggering a false fallback into the `else` block which forces the streak counter back to `1`.
3. **The fix you applied:** Removed the conditional constraint clause entirely, simplifying the branch check to `elif days_since_last == 1:`.
4. **Why that fix solves the problem:** Stripping the weekday checking rule allows any listen event executed exactly 1 day after the previous event date to increment the streak smoothly, completely unblocking Sunday boundaries.
5. **How you verified the fix:** Queried user streak endpoints (`/users/<user_id>/streak`) and verified the database attributes remained preserved instead of triggering a reset.

### Bug 2: Friends Listening Now shows historical data (`services/feed_service.py`)
1. **The symptoms you observed:** The feed endpoint populated friend listening metrics spanning across multiple separate calendar days instead of isolating records to today's active window.
2. **The exact root cause:** The `cutoff` calculation used a rolling window duration window of 24 hours (`cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`). This pulled in older events from yesterday if they occurred within the prior 24-hour bracket.
3. **The fix you applied:** Modified the `cutoff` variable to pin directly to today's midnight boundary using `.replace(hour=0, minute=0, second=0, microsecond=0)`.
4. **Why that fix solves the problem:** Pinned midnight truncation alters the underlying SQLAlchemy query filter condition to evaluate records matching exclusively from `00:00:00` of the current calendar day onward, ignoring prior dates.
5. **How you verified the fix:** Evaluated feed arrays (`/feed/<user_id>/listening-now`) and confirmed that events belonging to prior calendar dates were dropped from the feed list.

### Bug 3: Row Duplication in Song Search (`services/search_service.py`)
1. **The symptoms you observed:** When searching for specific query strings, individual song entities appeared duplicated multiple times inside the resulting JSON list.
2. **The exact root cause:** The query combined an `.outerjoin(song_tags)` alongside filter matching parameters. Because individual song records are connected to many rows in the tags association mapping table, SQL joins inherently produce duplicate base product rows for every corresponding tag entry match.
3. **The fix you applied:** Chained a `.distinct()` call to the SQLAlchemy ORM query generation structure before executing `.all()`.
4. **Why that fix solves the problem:** Injecting the `DISTINCT` modifier instructs the database engine to collapse duplicate matching identities, ensuring only completely unique `Song` model items are loaded into the results array.
5. **How you verified the fix:** Checked query matches (`/songs/search?q=Anthem`) and verified that the array length matched only unique song entities.

### Bug 4: Missing Song Rating Notifications (`services/notification_service.py`)
1. **The symptoms you observed:** Users were not getting notified when their shared tracks received star rating interactions from friends.
2. **The exact root cause:** While an isolated utility routine `create_notification()` was written into the codebase, it was never called or wired into the `rate_song` service workflow.
3. **The fix you applied:** Added a condition inside `rate_song` checking if the song has an associated sharer (`song.shared_by`) who is not the rater themselves, and instantiated a new `Notification` entry before running `db.session.commit()`.
4. **Why that fix solves the problem:** It hooks directly into the database transaction loop, dynamically generating a notification record whenever an external rating POST request modifies song rating metrics.
5. **How you verified the fix:** Evaluated notifications lists (`/users/<user_id>/notifications`) to confirm instances of song rating alerts appeared properly.

### Bug 5: Playlist Missing Final Track Entry (`services/playlist_service.py`)
1. **The symptoms you observed:** Playlists returned via the endpoint consistently omitted the very last song entry in their position ordering.
2. **The exact root cause:** The return block inside `get_playlist_songs()` concluded with the list-slice index `songs[:-1]`. This explicitly chopped off the final item from the list array before returning the serialized payload dictionary.
3. **The fix you applied:** Removed the slice operation completely, changing the return line to return all serialized elements in the `songs` list.
4. **Why that fix solves the problem:** Removing the array-chopping slice allows the complete matching dataset of database-retrieved playlist songs to pass through intact.
5. **How you verified the fix:** Manually added an extra track into a playlist via `flask shell`, re-queried the endpoint route, and verified that the item count correctly incremented from `7` to `8`.