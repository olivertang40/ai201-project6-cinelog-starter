# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` naming convention (consistent with `add_to_collection` in collection_service). Updated the single call site in `routes/watchlist/watchlist.py` (both the import and the function call).

**How I verified:** Searched the entire project for any remaining references to `save_to_watchlist` — none found. Confirmed the route's import and call both use the new name.

## Comment 2 — Deduplication
**What I did:** Added `AlreadyInWatchlistError` exception class directly in `watchlist_service.py`. Added a duplicate-check in `add_to_watchlist()` — queries `WatchlistEntry` by `user_id` + `film_id` before inserting; raises `AlreadyInWatchlistError` if a record already exists. Follows the same pattern as `add_to_collection()` in `collection_service.py`.

**How I verified:**

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with 4 tests mirroring `test_collection.py` structure:
- `test_add_to_watchlist_creates_entry` — basic add
- `test_add_to_watchlist_duplicate_raises` — deduplication check
- `test_add_to_watchList_nonexistent_film_raises` — handles nonexistent film (the required test per Comment 3)
- `test_get_watchlist_returns_alphabetical` — sort order verification

Also fixed missing `backref` relationships in `models.py`: added `watchlist_entries` relationships to both `User` and `Film` models so `entry.film` and `entry.user` work correctly.

**How I verified:** Ran `pytest tests/test_watchlist.py -v` — all 4 tests pass.

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
