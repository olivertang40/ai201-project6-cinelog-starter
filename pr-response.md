# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` naming convention (consistent with `add_to_collection` in collection_service). Updated the single call site in `routes/watchlist/watchlist.py` (both the import and the function call).

**How I verified:** Searched the entire project for any remaining references to `save_to_watchlist` — none found. Confirmed the route's import and call both use the new name.

## Comment 2 — Deduplication
**What I did:** Added `AlreadyInWatchlistError` exception class directly in `watchlist_service.py`. Added a duplicate-check in `add_to_watchlist()` — queries `WatchlistEntry` by `user_id` + `film_id` before inserting; raises `AlreadyInWatchlistError` if a record already exists. Follows the same pattern as `add_to_collection()` in `collection_service.py`.

**How I verified:** Ran `pytest tests/ -v` — all 8 tests pass, including `test_add_to_watchlist_duplicate_raises` which directly tests this logic.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with 4 tests mirroring `test_collection.py` structure:
- `test_add_to_watchlist_creates_entry` — basic add
- `test_add_to_watchlist_duplicate_raises` — deduplication check
- `test_add_to_watchList_nonexistent_film_raises` — handles nonexistent film (the required test per Comment 3)
- `test_get_watchlist_returns_alphabetical` — sort order verification

Also fixed missing `backref` relationships in `models.py`: added `watchlist_entries` relationships to both `User` and `Film` models so `entry.film` and `entry.user` work correctly.

**How I verified:** Ran `pytest tests/test_watchlist.py -v` — all 4 tests pass.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default.

**Reasoning:** The watchlist is a discovery and sharing feature — its primary value is visible when other users can see it. A new user adding their first film to a watchlist is likely exploring the app, not yet thinking about privacy settings. Defaulting to public means they immediately get the social benefit (others can see their taste, they can share it) without any extra action. If we defaulted to `public=False`, most watchlists would stay private simply because users never changed the default — which would kill the social dimension of the feature entirely.

This follows the same principle as social platforms (Twitter, Letterboxd, Goodreads): content is public by default, and users who want privacy can opt out. The friction is placed on the privacy use case, not the sharing use case, because sharing is the feature's core purpose.

**Tradeoff acknowledged:** The main downside is that users who add a film without reading the UI may not realize their watchlist is public. This is a real privacy concern — especially for users who are more privacy-conscious. A production-ready implementation should show a clear indicator in the UI when a list is public, and make the toggle easy to find. The default is appropriate here, but the UI needs to make the visibility state obvious at a glance.

## Comment 5 — Sort order
**My position:** Keep alphabetical sort (A → Z by title) for the watchlist, rather than switching to date-added descending.

**Reasoning:** A watchlist is fundamentally a queue of films a user *hasn't seen yet* — it's not a log. Date-added order (newest first) makes sense for a collection/log, where you want to see your most recent activity. But for a "films I want to watch" list, the more useful question is "what's on my list?" not "what did I add most recently?" Alphabetical sort makes the list scannable and consistent — users can find a specific film quickly, and the order doesn't shift every time they add something new.

Collection (`get_collection`) sorts by `date_added DESC` because it's a history — recency matters. Watchlist is a set — browsability matters. These are different use cases and deserve different sort defaults.

**Engagement with reviewer's point:** The reviewer's argument for date-added order is valid in one scenario: if a user adds a film because they just heard about it and want to watch it soon, date-added order surfaces that intent. That's a legitimate use case. However, alphabetical sort doesn't prevent that — the user can still watch films in any order they choose. The sort only affects how the list is *displayed*, not what they can do with it. If CineLog adds a priority or queue feature later, that would be the right place to handle "watch next" ordering. For now, alphabetical is the more neutral and scannable default.

## Comment 6 — Rebase
**What conflicted:** Two things conflicted during `git rebase origin/main`:
1. `.gitignore` — both branches added a `.gitignore` independently (add/add conflict). Resolved by keeping all entries from both sides, including `.pytest_cache/` from main.
2. `models.py` — main branch commit `07ca580` migrated `Film.id` from `Integer` to `String(36)` UUID, and also removed `WatchlistEntry` entirely (since that feature wasn't on main yet). After rebase, `WatchlistEntry` was gone and our `watchlist_entries` backref relationship on `Film` was also missing.

**How I resolved it:** For `.gitignore`, merged both sets of entries into one clean file and used `git rebase --skip` to drop the now-redundant tracking commit. For `models.py`, after rebase completed, manually restored `WatchlistEntry` with `film_id` updated from `Integer` to `String(36)` to match the UUID migration on main. Confirmed `watchlist_entries` relationships were intact on both `User` and `Film` models.

**How I verified no conflict remains:** Ran `pytest tests/ -v` — all 8 tests pass. Confirmed `git log --oneline` shows a linear history with no merge commits, branching cleanly from `origin/main`.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
