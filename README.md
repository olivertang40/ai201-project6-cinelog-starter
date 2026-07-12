# CineLog

A community film tracking app. Users log films they've watched, rate them, and build collections.

This repository is the starting point for **Project 6: Simulated Code Review**.

---

## Setup

```bash
pip install -r requirements.txt
python app.py
```

The app starts on `http://localhost:5000` and uses a local SQLite database (`cinelog.db`).

---

## Project Structure

```
ai201-project6-cinelog-starter/
├── app.py                     # Flask app factory
├── models.py                  # SQLAlchemy models (Film, User, CollectionEntry, WatchlistEntry)
├── services/
│   ├── collection_service.py  # Business logic for collections
│   └── watchlist_service.py   # Business logic for watchlist
├── routes/
│   ├── films.py               # Film browsing endpoints
│   ├── collection.py          # Collection endpoints
│   └── watchlist/
│       └── watchlist.py       # Watchlist endpoints
├── tests/
│   ├── test_collection.py     # Tests for collection service
│   └── test_watchlist.py      # Tests for watchlist service
├── CONTRIBUTING.md            # Commit conventions and PR guidelines
├── pr-response.md             # Code review response documentation
└── requirements.txt
```

---

## API Overview

### Films

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/films/` | List all films (supports `?genre=` and `?year=` filters) |
| GET | `/films/<film_id>` | Get a single film by UUID |

### Collection

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/collection/<user_id>` | Get a user's collection (newest first) |
| POST | `/collection/<user_id>/add` | Add a film to the collection |
| DELETE | `/collection/<user_id>/remove` | Remove a film from the collection |

### Watchlist

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/watchlist/<user_id>` | Get a user's watchlist (alphabetical by title) |
| POST | `/watchlist/<user_id>/add` | Add a film to the watchlist |

---

## Data Models

**Film** — A film in the catalog. IDs are UUIDs.

**User** — A registered user. IDs are UUIDs.

**CollectionEntry** — Links a user to a film they've watched. Stores rating and date added. A user can only have one entry per film.

**WatchlistEntry** — Links a user to a film they want to watch. Stores date added and visibility (public/private). A user can only have one entry per film.

---

## Naming Conventions

Service functions follow a `verb_to_noun` pattern. See `CONTRIBUTING.md` for full details.

---

## Running Tests

```bash
pytest tests/
```

---

## Your Task

This is the completed `feature/watchlist` branch with a fully implemented watchlist feature. The PR has been reviewed and all six maintainer comments have been addressed:

1. ✅ Renamed `save_to_watchlist` to `add_to_watchlist` for naming consistency
2. ✅ Added deduplication logic to prevent duplicate entries
3. ✅ Added comprehensive test coverage
4. ✅ Documented design decision: `public=True` default visibility
5. ✅ Documented design decision: Alphabetical sort order
6. ✅ Rebased onto main and resolved UUID migration conflicts

See `pr-response.md` for complete documentation of all changes and design decisions.
