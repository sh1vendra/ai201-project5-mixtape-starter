# AI Usage

I used AI (Claude Code) mainly during orientation and to sanity-check hypotheses, not to find the bugs themselves.

- **Orientation:** I gave the AI `services/notification_service.py` and `services/playlist_service.py` and asked it to trace the "add song to playlist" call chain end-to-end (route → service → notification). This confirmed the flow documented in the Data Flow section below and saved me from manually cross-referencing `routes/playlists.py` and `models.py` line by line.
- **Issue #1 (streak):** After noticing the `today.weekday() != 6` condition myself while reading `update_listening_streak`, I asked the AI to explain what `datetime.weekday()` returns for each day (0=Monday...6=Sunday) to confirm my read of the comparison before changing it. I verified the fix by manually reasoning through a Saturday→Sunday and Sunday→Monday listen sequence.
- **Issue #4 (notifications):** The README hinted the missing notification followed the pattern of an existing working one, so I asked the AI to diff `add_to_playlist` (which notifies correctly) against `rate_song` (which didn't) line by line. It pointed out that `rate_song` committed the rating but never called `create_notification` at all, which I then confirmed by reading both functions directly.
- **Issue #5 (playlist duplicates/last song):** I read `get_playlist_songs` myself first and noticed the `songs[:-1]` slice looked suspicious given the docstring says "returns all songs." I asked the AI to explain what `songs[:-1]` evaluates to on a list of length 1 to double-check my read (an empty list) before committing to the fix.

In every case I read the relevant function myself before asking the AI to explain it, and verified each fix by re-reading the diff and reasoning through the boundary cases by hand rather than trusting the AI's explanation alone.

# Codebase Map

## File List

- `app.py` — Flask app factory (`create_app`), initializes `db`, registers the four blueprints, calls `db.create_all()`.
- `models.py` — SQLAlchemy schema only: `User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus `friendships`/`song_tags`/`playlist_entries` association tables. Each model has a `to_dict()` serializer, no business logic.
- `routes/feed.py` — exposes `GET /<user_id>/listening-now` and `GET /<user_id>/activity`, both pure pass-throughs to `feed_service`.
- `routes/playlists.py` — exposes playlist create/get/list-songs/add-song endpoints; delegates to `playlist_service` for CRUD and `notification_service.add_to_playlist` for the add-song side effect.
- `routes/songs.py` — exposes search, single-song lookup, rate, and listen endpoints, delegating to `search_service`, `notification_service.rate_song`, and `streak_service.record_listening_event`.
- `routes/users.py` — exposes profile, streak, and notification endpoints; queries `User` directly for the profile route, otherwise delegates to `streak_service`/`notification_service`.
- `services/feed_service.py` — builds the "friends listening now" (24h window, deduped) and general activity feeds from `ListeningEvent` rows.
- `services/notification_service.py` — creates and retrieves `Notification` rows; triggers a notification when a song is added to a playlist or rated, notifying the song's sharer.
- `services/playlist_service.py` — creates playlists and returns their songs in position order.
- `services/search_service.py` — case-insensitive title/artist search over `Song`, joined to tags.
- `services/streak_service.py` — records `ListeningEvent`s and updates a user's consecutive-day listening streak.

## Data Flow: Playlist Add-Song → Notification

1. Client calls `POST /playlists/<playlist_id>/songs` with `song_id`, `added_by` → `routes/playlists.py: add_song(playlist_id)`.
2. Route calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)`.
3. Inside `add_to_playlist`: `db.session.get(Song, song_id)`, `db.session.get(User, added_by_user_id)`, `db.session.get(Playlist, playlist_id)` — each raises `ValueError` if not found.
4. If `song not in playlist.songs`: `playlist.songs.append(song)` then `db.session.commit()`.
5. If `song.shared_by != added_by_user_id`: calls `create_notification(user_id=song.shared_by, notification_type="song_added_to_playlist", body=...)`.
6. `create_notification` builds a `Notification`, `db.session.add()`, `db.session.commit()`, returns the instance (return value unused by step 5's caller).
7. Route returns `{"message": "Song added to playlist"}, 201`.

## Pattern Noticed

Routes are thin request/response wrappers with no logic of their own — every route just validates input shape and delegates to exactly one service function, which is where all the real logic (and the bugs) lives.

# Root Cause Analysis

## Issue #1: Streak resets incorrectly on Sundays

**How I reproduced it:** In a Flask shell, I created a user with `last_listened_at` set to a Saturday and called `update_listening_streak(user, sunday_datetime)` where `sunday_datetime` was exactly one day later. Per the streak rules ("listened yesterday → increment by 1"), the streak should have gone from N to N+1, but it reset to 1 instead. Repeating the same one-day gap starting on any other weekday incremented correctly, isolating the bug to the Sunday case specifically.

**How I found the root cause:** `routes/users.py` → `streak_service.record_listening_event` → `update_listening_streak`, which is the only place streak math happens. Reading the function line by line, the branch that handles a one-day gap was `elif days_since_last == 1 and today.weekday() != 6:`. The `and today.weekday() != 6` clause only exists on this branch — the "no change" and "reset to 1" branches don't have it — which made it clear this extra condition was deliberately (and incorrectly) excluding Sundays from the increment path.

**The root cause:** Python's `datetime.weekday()` returns `6` for Sunday. The code added `today.weekday() != 6` to the "consecutive day" branch, intending presumably to handle a week-boundary case, but instead it excluded every Sunday from ever counting as a valid one-day increment. Any user whose current listening day was a Sunday and who had listened the day before (Saturday) fell through to the `else` branch and had their streak reset to 1, even though they listened on consecutive days.

**Fix and side-effect check:** Removed the `and today.weekday() != 6` clause so the branch is just `elif days_since_last == 1:`, matching the documented rule that a one-day gap always increments regardless of which weekday it lands on. I re-checked the `days_since_last == 0` (same-day, no-op) and `else` (multi-day gap, reset to 1) branches — neither references weekday, so they're unaffected. I also re-ran the Saturday→Sunday and Sunday→Monday sequences by hand: both now increment correctly, and a Friday→Sunday (2-day) gap still correctly resets to 1.

## Issue #4: Song sharer isn't notified when their song is rated

**How I reproduced it:** Using `flask shell`, I called `notification_service.rate_song(other_user_id, song_id, 5)` for a song shared by a different user, then queried `notification_service.get_notifications(sharer_id)`. No new notification appeared for the sharer, even though rating completed successfully and the `Rating` row was created. Doing the equivalent for `add_to_playlist` (add a song someone else shared to a playlist) did produce a `song_added_to_playlist` notification for the sharer, confirming the pattern exists elsewhere and is simply missing here.

**How I found the root cause:** Per the hint, I compared `add_to_playlist` and `rate_song` in `services/notification_service.py` line by line. `add_to_playlist` ends with an `if song.shared_by != added_by_user_id: create_notification(...)` block right after the state-changing `db.session.commit()`. `rate_song` also commits its state change (`db.session.commit()` after creating/updating the `Rating`) but the function just `return`s the rating immediately after — there is no equivalent notification block at all.

**The root cause:** The notification pattern (create a `Notification` for `song.shared_by` whenever someone other than the sharer acts on their song) was implemented for the "add to playlist" action but never implemented for the "rate song" action. It's not a typo or an off-by-one — the code path to build and send that notification simply doesn't exist in `rate_song`, so sharers never learn when their shared songs get rated.

**Fix and side-effect check:** Added the same notification block used in `add_to_playlist`, adapted for rating: after `db.session.commit()` and before `return rating`, `if song.shared_by != user_id: create_notification(user_id=song.shared_by, notification_type="song_rated", body=f"{rater.username} rated your song '{song.title}' {score}/5.")`. I checked that `rater` and `song` were already fetched earlier in the function (they are, for validation), so no extra queries were needed. I confirmed the self-rating case is still excluded (a user rating their own shared song does not generate a notification, matching the `add_to_playlist` behavior), and that re-rating an existing `Rating` (the `existing` branch) still fires the notification on every update, consistent with `add_to_playlist` notifying on every add.

## Issue #5: Last song in a playlist is missing from the song list

**How I reproduced it:** In `flask shell`, I created a playlist and added a single song to it, then called `playlist_service.get_playlist_songs(playlist_id)` — it returned `[]` instead of the one song. Adding a second song made the first song appear but not the second, confirming that the last song by position was always being dropped regardless of playlist length.

**How I found the root cause:** Traced `routes/playlists.py`'s list-songs endpoint into `playlist_service.get_playlist_songs`. The function queries `Song` joined to `playlist_entries`, ordered ascending by `position`, then returns `[song.to_dict() for song in songs[:-1]]`. The docstring directly above explicitly states "This function returns all songs in the playlist," which contradicted the `[:-1]` slice — that mismatch between documented behavior and actual code was the giveaway.

**The root cause:** `songs[:-1]` unconditionally drops the last element of the ordered song list. For a playlist with one song, `songs[:-1]` is `[]`, so the entire playlist appears empty; for any playlist, the last song by position is silently excluded from every response. This wasn't a filtering condition at all — it was an unconditional slice that should never have been there given the function is meant to return the complete ordered list.

**Fix and side-effect check:** Changed the return statement to `[song.to_dict() for song in songs]`, returning the full ordered list. I checked `get_playlist` (returns playlist metadata only, doesn't touch songs) and `add_to_playlist` in `notification_service.py` (uses `playlist.songs`, the ORM relationship, not `get_playlist_songs`) to confirm neither depended on the truncated behavior. I re-verified with one-song, two-song, and empty playlists in `flask shell`: all songs now appear in correct position order, and an empty playlist still correctly returns `[]`.
