# PR Response Doc — CineLog Watchlist Feature

## AI Usage

I used Cursor AI to orient myself in the codebase before reading the review comments — specifically to summarize how `add_to_collection()` handles deduplication and how `tests/test_collection.py` structures fixtures and assertions. I verified every explanation against the actual source files before implementing anything.

For Comments 4 and 5, I drafted my visibility and sort-order positions first, then asked AI to play devil's advocate ("what counterargument would a careful reviewer raise?"). The AI surfaced the privacy risk of default-public watchlists for guilt-pleasure viewing; I kept my public-default position but strengthened the tradeoff section to acknowledge that case explicitly. The AI also suggested alphabetical sort helps large lists — I agreed with the maintainer's date-added preference instead, because CineLog's collection endpoint already uses newest-first and watchlists are action queues, not catalogs.

I also used AI to verify my final `git log --oneline` output followed conventional commit format before pushing.

## Comment 1 — Rename

**What I did:**

Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` and updated the single route call site in `routes/watchlist/watchlist.py` (`add_film()` handler). I ran a project-wide search for `save_to_watchlist` to confirm no remaining references before committing.

**How I verified:**

- `grep`/project search returned zero matches for `save_to_watchlist` after the rename.
- Ran `pytest tests/ -v` to confirm imports and the route wiring still work.

## Comment 2 — Deduplication

**What I did:**

Added an explicit deduplication check to `add_to_watchlist()` before creating a new `WatchlistEntry`. If a `(user_id, film_id)` pair already exists, the function raises `AlreadyInWatchlistError` instead of inserting a duplicate row. I also added a `UniqueConstraint` on the model to enforce this at the database level.

**How I verified:**

I modeled the logic directly on `add_to_collection()` in `services/collection_service.py`:
1. Validate the film exists first (`FilmNotFoundError` if not).
2. Query for an existing entry with `WatchlistEntry.query.filter_by(user_id=..., film_id=...).first()`.
3. Raise a domain error if found; otherwise create and commit.

Re-ran `pytest tests/ -v` after the change. Added `test_add_to_watchlist_duplicate_raises` (stretch) to prove only one row remains after a duplicate add attempt.

## Comment 3 — Missing test

**What I did:**

Created `tests/test_watchlist.py` with `test_add_to_watchlist_nonexistent_film_raises`, modeled after `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py`. The test uses the same fixture pattern (`app`, `sample_user`) and asserts that a fake UUID film ID raises `FilmNotFoundError` rather than a database integrity error.

**How I verified:**

```bash
pytest tests/test_watchlist.py -v
pytest tests/ -v
```

Both pass (10 tests total).

## Comment 4 — Default visibility

**My position:**

Keep the default as **`public=True`** for newly added watchlist entries.

**Reasoning:**

CineLog is a community film-tracking app — users log what they've watched, build collections, and share taste with friends. A watchlist is most valuable as a lightweight social signal: "what are you excited to watch next?" Default-public optimizes for the common low-friction workflow (save a film → it shows up for friends without an extra toggle step). This aligns with CineLog's discovery-oriented identity: watchlists are a curation surface, not a private notes app.

**Tradeoff acknowledged:**

Default-public creates privacy risk for users who treat watchlists as personal intent lists (guilty-pleasure picks, surprise plans, or films they don't want others to see). Default-private would optimize for privacy-by-default but adds friction for the many users who *do* want to share. I'm keeping public as the default because CineLog's community value outweighs the privacy case at launch — and callers can already pass `"public": false` in the POST body (stretch feature) until a dedicated visibility toggle UI ships.

## Comment 5 — Sort order

**My position:**

Change watchlist sort order to **date-added, newest first** (matching `get_collection()`).

**Reasoning:**

When someone opens their watchlist, the most useful question is "what did I recently decide to watch?" — not "what comes first alphabetically?" Newest-first puts the current intent queue at the top and matches how users mentally prioritize backlog items.

**Engagement with reviewer's point:**

I agree with the maintainer's argument that most users care about what they added recently. Alphabetical sorting helps scanning a very large static list, but that need is better served by client-side search/filter once a UI exists. At the service layer, newest-first provides higher default utility and keeps the API consistent with `get_collection()`, which already returns `date_added` descending. I implemented `.order_by(WatchlistEntry.date_added.desc())` in `get_watchlist()` and added `test_get_watchlist_returns_newest_first` to lock in the behavior.

## Comment 6 — Rebase

**What conflicted:**

While the PR was open, `main` refactored `Film.id` (and related foreign keys) from integers to UUID strings. After rebasing `feature/watchlist` onto `origin/main`, the branch's original `WatchlistEntry` model (with `Integer` `film_id`) conflicted with the UUID-based `Film` model on main. There was also a minor `.gitignore` add/add conflict (`.pytest_cache/` vs local entries).

**How I resolved it:**

```bash
git fetch origin
git rebase origin/main
```

- Resolved `.gitignore` by keeping both `.pytest_cache/` and virtualenv entries.
- Re-added `WatchlistEntry` with `film_id = db.Column(db.String(36), db.ForeignKey("film.id"))` to match main's UUID schema.
- Updated watchlist service docstrings, route comments, and tests to treat `film_id` as a UUID string.

**How I verified no conflict remains:**

- Rebase completed with no remaining conflict markers.
- `git log --oneline origin/main..HEAD` shows a linear history with no merge commits.
- `pytest tests/ -v` passes on the rebased branch.

## Commit History

Screenshot of `git log --oneline` on `feature/watchlist` (conventional commits, no merge commits):

```
f2242c6 docs: add pr-response.md with visibility and sort order decisions
5f96a3e test: add watchlist deduplication, remove, and sort order tests
ed39712 feat: add remove_from_watchlist endpoint and optional public parameter
3f1be35 fix: update WatchlistEntry film_id to UUID after main branch refactor
0b6ccac test: add test for nonexistent film_id in add_to_watchlist
0a4f7ae fix: add deduplication check to prevent duplicate watchlist entries
690f259 fix: rename save_to_watchlist to add_to_watchlist per naming convention
e0392cf fix: update film retrieval method to use db.session.get in collection and watchlist services
df33f0b feat: add watchlist model and add_to_watchlist endpoint
```

## PR Description

This PR implements CineLog's **watchlist** feature — a way for users to save films they want to watch later.

**What's included:**
- `WatchlistEntry` model and service functions in `services/watchlist_service.py`
- REST endpoints:
  - `GET /watchlist/<user_id>` — returns the user's watchlist (with `date_added` and `public` metadata)
  - `POST /watchlist/<user_id>/add` — adds a film by `film_id` (optional `public` field)
  - `DELETE /watchlist/<user_id>/remove` — removes a film from the watchlist
- Deduplication: adding the same film twice raises `AlreadyInWatchlistError` (409)
- Tests in `tests/test_watchlist.py`

**Design decisions:**
- **Default visibility:** `public=True` — optimizes for CineLog's community discovery use case; callers can set `"public": false` explicitly.
- **Sort order:** newest-first by `date_added` — matches `get_collection()` and surfaces recently saved films at the top.

**Manual testing:**

1. Start the app:
   ```bash
   pip install -r requirements.txt
   python app.py
   ```
   Server runs at `http://127.0.0.1:5000`.

2. Create a test user and film (copy the printed UUIDs):
   ```bash
   python - <<'PY'
   from app import create_app, db
   from models import User, Film

   app = create_app()
   with app.app_context():
       db.create_all()
       user = User(username="watchlist_user", email="watchlist@example.com")
       film = Film(title="Watchlist Test Film", year=2026, genre="Drama")
       db.session.add_all([user, film])
       db.session.commit()
       print(user.id)
       print(film.id)
   PY
   ```

3. Add a film to the watchlist:
   ```bash
   curl -X POST "http://127.0.0.1:5000/watchlist/USER_ID/add" \
     -H "Content-Type: application/json" \
     -d "{\"film_id\":\"FILM_ID\"}"
   ```
   Expect `201` with `"public": true`.

4. Fetch the watchlist:
   ```bash
   curl "http://127.0.0.1:5000/watchlist/USER_ID"
   ```
   Verify the film appears with `date_added` and `public` fields.

5. Verify deduplication — repeat step 3; expect `409` with an error message.

6. Remove from watchlist:
   ```bash
   curl -X DELETE "http://127.0.0.1:5000/watchlist/USER_ID/remove" \
     -H "Content-Type: application/json" \
     -d "{\"film_id\":\"FILM_ID\"}"
   ```
   Expect `200`. Re-fetch in step 4 — list should be empty.

## Stretch — Remove from watchlist

**What I did:**

Added `remove_from_watchlist(user_id, film_id)` following the same pattern as `remove_from_collection()`, plus a `DELETE /watchlist/<user_id>/remove` route. Raises `NotInWatchlistError` if the film isn't on the list.

**How I verified:**

`test_remove_from_watchlist_removes_entry` and `test_remove_from_watchlist_not_on_watchlist_raises` in `tests/test_watchlist.py`.

## Stretch — Second test

**What I did:**

Added `test_add_to_watchlist_duplicate_raises` — verifies duplicate adds raise `AlreadyInWatchlistError` and only one DB row exists.

**Why this case:**

Comment 3 covered nonexistent `film_id`. Duplicate adds are the next most common failure mode (double-click save, retry after slow network) and the deduplication fix should be proven at the test layer, not just by reading service code.

## Stretch — Visibility toggle

**What I did:**

Added an optional `public` parameter to `add_to_watchlist(user_id, film_id, public=True)` and wired it through the POST endpoint via `data.get("public", True)`. Callers can send `"public": false` to create a private entry.

**How a caller uses it:**

```bash
curl -X POST "http://127.0.0.1:5000/watchlist/USER_ID/add" \
  -H "Content-Type: application/json" \
  -d "{\"film_id\":\"FILM_ID\", \"public\": false}"
```

Verified with `test_add_to_watchlist_respects_public_false`.
