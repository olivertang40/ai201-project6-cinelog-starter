# PR Response Doc — CineLog Watchlist Feature

## AI Usage

Kiro (AI coding assistant) was used throughout this project in the following ways:

1. **Codebase orientation (Milestone 1):** Used AI to read and explain `models.py`, `collection_service.py`, and `watchlist_service.py` in Chinese, building a clear mental model of the relationships between models, services, and routes before touching any code.

2. **Bug discovery:** AI review of `watchlist_service.py` caught that `AlreadyInWatchlistError` was being raised without being defined or imported — a `NameError` that would only surface at runtime. This identified the need to define the exception class in the file directly (Plan A).

3. **Identifying missing backref:** AI review of `models.py` identified that `WatchlistEntry` had no `backref="film"` relationship defined on `Film`, which caused `entry.film` to fail with `AttributeError` at test time. This was fixed in `models.py` before tests ran.

4. **Stress-testing design arguments (Comments 4 and 5):** After drafting the Comment 4 and Comment 5 responses, I asked AI: "What counterarguments would a careful code reviewer raise against these positions? What tradeoffs am I not considering?" For Comment 4 (public default), AI raised the privacy concern for users who don't notice the UI state — this was already in my draft and I kept it. For Comment 5 (alphabetical sort), AI pointed out the "just heard about it, want to watch soon" recency use case — I incorporated this directly into the "Engagement with reviewer's point" section and explained why alphabetical sort doesn't prevent that use case.

5. **Commit message validation:** Asked AI to review `git log --oneline` output and check for conventional commit format compliance — AI identified 3 commits missing `fix:` / `test:` prefixes, which were corrected by rebuilding the branch via cherry-pick.

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

### Feature Overview

This PR adds a **watchlist feature** to CineLog — allowing users to save films they want to watch later. The watchlist complements the existing collection feature: the collection tracks films a user has already watched (with optional ratings), while the watchlist tracks films they plan to watch.

**What was added:**
- `WatchlistEntry` model with `user_id`, `film_id`, `date_added`, and `public` fields
- `add_to_watchlist()` and `get_watchlist()` service functions in `services/watchlist_service.py`
- `AlreadyInWatchlistError` exception for duplicate prevention
- API endpoints: `GET /watchlist/<user_id>` and `POST /watchlist/<user_id>/add`
- Full test coverage in `tests/test_watchlist.py`
- Aligned `WatchlistEntry.film_id` with main branch's UUID migration (was `Integer`, now `String(36)`)

### Design Decisions

**1. Default visibility (`public=True`):**  
Watchlists default to public because the feature's primary value is social discovery and sharing. Defaulting to private would leave most lists invisible (users rarely change defaults), killing the social dimension. This follows common social platform patterns (Letterboxd, Goodreads). Tradeoff: users who don't notice the UI state may not realize their list is public — production UI must clearly indicate visibility status.

**2. Sort order (alphabetical by title):**  
`get_watchlist()` returns films sorted A→Z by title, not by date added. Rationale: a watchlist is a browsable set ("what's on my list?"), not a chronological log. Alphabetical sort makes the list scannable and stable. The collection feature uses date-added descending because it's a history; watchlist uses alphabetical because it's a queue. This serves different use cases with appropriate defaults.

### Manual Testing Instructions

**Prerequisites:**
```bash
# Activate venv and ensure dependencies are installed
.venv\Scripts\activate  # Windows
pip install -r requirements.txt
```

**1. Start the Flask app:**
```bash
python app.py
```
The app runs at `http://127.0.0.1:5000`.

**2. Add a user and films (via database or existing API):**
Use the existing `/films` endpoints to create films, or add test data directly to the SQLite database.

**3. Add a film to a user's watchlist:**
```bash
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_uuid>"}'
```
Expected: `201 Created` with the watchlist entry JSON.

**4. Try adding the same film again (duplicate check):**
```bash
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<same_film_uuid>"}'
```
Expected: Error response (the deduplication logic prevents duplicate entries).

**5. Get the user's watchlist:**
```bash
curl http://127.0.0.1:5000/watchlist/<user_id>
```
Expected: JSON array of films, sorted alphabetically by title. Each film includes `date_added` and `public` fields from the watchlist entry.

**6. Verify alphabetical sort:**
Add multiple films with different titles (e.g., "Alien", "Blade Runner", "Casablanca"). Confirm `GET /watchlist/<user_id>` returns them in A→Z order, not by the order they were added.

**7. Run automated tests:**
```bash
pytest tests/test_watchlist.py -v
```
All 4 watchlist tests should pass.

---

### Final Commit History

```
1d6c2c5 fix: update WatchlistEntry film_id to UUID after main branch refactor
f2a75e0 docs: add visibility and sort order design decisions to pr-response.md
81ddff8 test: add watchlist tests and fix missing backref relationships in models
a6c880b fix: add deduplication check to prevent duplicate watchlist entries
e4ceac0 fix: rename save_to_watchlist to add_to_watchlist per naming convention
f0a6ba7 fix: update film retrieval method to use db.session.get in collection and watchlist services
3e4b117 feat: add watchlist model and add_to_watchlist service endpoint
```

All commits follow conventional commit format. No merge commits.
