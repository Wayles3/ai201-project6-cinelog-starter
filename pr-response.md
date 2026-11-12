# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude Code as an assistant on this project. I'm listing where it helped so it's clear what was AI-assisted:

- **Orientation:** Claude helped me read `models.py`, `services/collection_service.py`, and `tests/test_collection.py` and summarized the existing patterns (naming, duplicate check, test fixtures).
- **Code and git assistance:** Claude helped implement the code changes for Comments 1-3 and the stretch features, and helped with the rebase onto `origin/main`, the `models.py` conflict, and the commit history cleanup. It also ran the test suite and tried the endpoints with curl while I was writing the manual test steps.
- **Comments 4 and 5:** Claude suggested initial draft wording for both responses. For Comment 4 I rewrote the reasoning in my own words and wrote the tradeoff section myself, including the point that the `public` flag isn't enforced yet. For Comment 5 I added my own Letterboxd example and wrote the response to the maintainer's concern. Some supporting points (consistency with `get_collection()`, the `?sort=title` synthesis, and the sentence about keeping the public default after enforcement ships) started as Claude's suggestions, and I kept them because I agree with them.
- **Stress-testing:** I asked Claude to point out weak spots in the drafts. It flagged that the unenforced `public` flag undercuts the "opt-out" argument and that some claims were guesses without data, and the final text reflects those changes.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` so it matches the `verb_to_noun` convention in CONTRIBUTING.md (`add_to_collection()`, `remove_from_collection()`, `get_collection()`). I also updated the docstring wording from "Save" to "Add".
**How I verified:** Ran a project-wide `grep -rn "save_to_watchlist"` before and after. Before: three hits (the definition at `services/watchlist_service.py`, the import and the call in `routes/watchlist/watchlist.py`). After: zero hits. `pytest tests/ -v` still passed.

## Comment 2 — Deduplication
**What I did:** Added a duplicate check to `add_to_watchlist()`, following the pattern in `add_to_collection()` (`services/collection_service.py`). That function runs `CollectionEntry.query.filter_by(user_id=user_id, film_id=film_id).first()` after confirming the film exists. If a row comes back it raises `AlreadyInCollectionError` instead of inserting, and the route turns that into HTTP 409. I mirrored this exactly: `WatchlistEntry.query.filter_by(user_id=..., film_id=...).first()`, raising a new `AlreadyInWatchlistError`. The route now maps it to 409 (and `FilmNotFoundError` to 404, which previously surfaced as a 500).
**How I verified:** `test_add_to_watchlist_duplicate_raises` adds the same film twice, asserts `AlreadyInWatchlistError` on the second call, and asserts only one row exists.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with `test_add_to_watchlist_nonexistent_film_raises`, modelled on `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py` (same `app` / `sample_user` fixtures, same all-zero UUID, `pytest.raises(FilmNotFoundError)`). I also added a happy-path test and the duplicate test above, since CONTRIBUTING.md asks for all three for any new service function.
**How I verified:** `pytest tests/test_watchlist.py -v` and then `pytest tests/ -v` (7 passed).

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default, and let users choose privacy per film through the new optional `public` field on `POST /watchlist/<user_id>/add`. That field is stored and returned today but not yet enforced on reads (see tradeoff below).

**Reasoning:** CineLog is fundamentally a community film tracker: collections track what you've watched, and watchlists cover what's next. The whole point of a watchlist is visibility: it lets friends suggest films or gives a quick signal of your taste before you've logged much. Defaulting to public eliminates unnecessary UX friction. Most users never touch default settings, so a private-by-default rule would mean watchlists sit invisible forever, killing the social loop. Keeping it public also aligns with the backend, where `WatchlistEntry.public = True` is already the model default and `add_to_watchlist()` and its endpoint now default to the same value.

**Tradeoff acknowledged:** A private default optimizes for protecting users who might add something embarrassing to their queue without noticing the setting. However, public as the default makes sense given the current implementation: the `public` flag isn't enforced yet. Because `GET /watchlist/<user_id>` returns everything to anyone, `public: false` doesn't protect anyone at this stage, so a private default would not buy real privacy today. Public-by-default is honest about the current state of the codebase while optimizing for the app's social loop. I would keep this default once enforcement exists, because the cost of a wrongly public entry (a film title) is small next to the cost of a silent private default hiding most watchlists. When enforcement ships, I'd pair it with a way to change visibility on an existing entry, which this PR does not include.

## Comment 5 — Sort order
**My position:** I'm respectfully pushing back on alphabetical-by-title as the default and sorting newest-added first. I changed `get_watchlist()` from `Film.title.asc()` to `WatchlistEntry.date_added.desc()` and added `test_get_watchlist_returns_newest_first`. Alphabetical sorting stays available as a possible `?sort=title` option later (see synthesis).

**Reasoning:** A watchlist functions as a queue of intent. On Letterboxd, I add films the same day I hear about them, and I always scroll right to the top of my watchlist to check that they saved and to keep them front and center while my interest is fresh. Placing newest items first puts that recent intent at the top with zero scrolling.

Newest-first also makes `get_watchlist()` consistent with `get_collection()`, which already sorts by `date_added.desc()`, so both lists behave identically from a client perspective. Alphabetical order only solves the problem of hunting for a specific title in a long list, which is fundamentally a search or filtering problem rather than an optimal default sort order.

**Engagement with reviewer's point:** The maintainer's concern is that a specific film is hard to find unless the list is alphabetical. I disagree on the default, with one caveat: I don't have telemetry data for CineLog usage, so my position relies on common product expectations for queue-based lists rather than empirical measurement. The maintainer's point is strongest for a user with an old, extensive watchlist trying to find a specific title, where alphabetical sorting is objectively more helpful.

**Synthesis:** Ship newest-first as the default now. If users request alphabetical sorting, we can add an optional `?sort=title` query parameter later rather than forcing alphabetical as the default across the board. I left that parameter out of this PR to keep the review scope focused.

## Comment 6 — Rebase
**What conflicted:** `feature/watchlist` was branched before main's UUID refactor (57ec3f9). `git rebase origin/main` stopped with a content conflict in `models.py` on `WatchlistEntry.film_id`: main had `db.String(36)`, my branch had `db.Integer` plus `index=True`.
**How I resolved it:** Kept main's `String(36)` type and my `index=True`. After the rebase finished, I removed the remaining integer references on my side (the `add_to_watchlist` docstring said `film_id (int) ... pre-refactor`, and the route docstring documented `"film_id": <int>`) and committed that separately as `fix: update WatchlistEntry film_id to UUID after main branch refactor`.
**How I verified no conflict remains:** `git status` clean with no rebase in progress, no conflict markers (`grep -rn "<<<<<<<"`), `git log --merges` is empty, and `pytest tests/ -v` passes using UUID film IDs.

## Stretch features
**`remove_from_watchlist(user_id, film_id)`:** Follows `remove_from_collection()` exactly: it looks up the entry with `WatchlistEntry.query.filter_by(user_id=..., film_id=...).first()`, deletes and commits it, and returns `True`. If the film isn't on the watchlist it raises `NotInWatchlistError` (mirroring `NotInCollectionError`), which `DELETE /watchlist/<user_id>/remove` maps to 404; a missing `film_id` returns 400. The name follows the project's `verb_to_noun` convention. Tests: `test_remove_from_watchlist_deletes_entry` and `test_remove_from_watchlist_not_present_raises`.

**Second test (edge case beyond the review):** `test_add_to_watchlist_same_film_for_different_users_allowed`. I chose it because Comment 2's dedup check is the riskiest new logic: if I had written the `filter_by` with only `film_id` (forgetting `user_id`), the existing duplicate test would still pass, but the second user would be wrongly blocked from saving a film someone else already saved. This test pins the check to `(user_id, film_id)`. I also added `test_add_to_watchlist_public_defaults_true_and_can_be_private` for the visibility parameter below.

**Visibility toggle:** `add_to_watchlist(user_id, film_id, public=True)` takes a `public` argument that is stored on the new `WatchlistEntry`. `POST /watchlist/<user_id>/add` accepts an optional `"public"` field in the JSON body. It defaults to `true` (see Comment 4), and a non-boolean value (e.g. `"no"`) returns 400 rather than being silently coerced. Example: `{"film_id": "<uuid>", "public": false}` creates a private entry.

## Git history
Output of `git log --oneline origin/main..HEAD` on `feature/watchlist` (linear history, no merge commits; `git log --merges` is empty). This block was captured before the commit that adds it.

```
380976d docs: add pr-response.md with review responses
25aa11f docs: document watchlist endpoints and model in README
31fa30c test: add test that duplicate check is scoped per user
f6d1d65 test: add test for public default and private watchlist entries
54072d8 feat: add public parameter to add_to_watchlist endpoint
a325cf9 test: add tests for remove_from_watchlist
74eb6f6 feat: add remove_from_watchlist service and endpoint
6052ce0 test: add test for watchlist newest-first sort order
0ee0e05 feat: sort watchlist by date added, newest first
646ca54 fix: update WatchlistEntry film_id to UUID after main branch refactor
2e37a58 test: add happy path and duplicate tests for add_to_watchlist
5071e09 test: add test for nonexistent film_id in add_to_watchlist
01f9469 fix: add deduplication check to prevent duplicate watchlist entries
e9d69c6 fix: rename save_to_watchlist to add_to_watchlist per naming convention
a4a06ab fix: add .gitignore to keep venv, caches, and local database out of history
b80cae2 fix: update film retrieval method to use db.session.get in collection and watchlist services
d38be14 feat: add watchlist model and add-to-watchlist endpoint
```

## PR Description
**What it does:** Adds a watchlist to CineLog: a list of films a user wants to watch later, separate from their collection of films already watched. Users can add a film (`POST /watchlist/<user_id>/add`), view their list (`GET /watchlist/<user_id>`), and remove a film (`DELETE /watchlist/<user_id>/remove`). Adding a film that is already on the list, or one that doesn't exist, returns a clear 409 or 404 instead of a server error.

**Design decisions:**
- **Default visibility: public.** CineLog is a community app and watchlists are most useful when others can see them; privacy is a one-field opt-out (`"public": false`). Enforcing visibility for other viewers is deliberately left for a follow-up that includes auth.
- **Sort order: newest added first.** A watchlist is a queue of recent intent, and this matches `get_collection()`. Alphabetical remains a possible `?sort=title` option later.

**Review fixes in this PR:** renamed `save_to_watchlist` → `add_to_watchlist` (verb_to_noun), added duplicate protection modelled on `add_to_collection()`, added a nonexistent-film test, and rebased onto main's UUID refactor (no merge commits). Extras: `remove_from_watchlist`, the `public` parameter, and additional tests.

**How to test manually:**
1. `pip install -r requirements.txt`, then `python seed.py` (it prints the demo `user_id`), then `python app.py`. On macOS, port 5000 is often taken by AirPlay Receiver; if curl returns 403, run `python -c "from app import create_app; create_app().run(port=5055)"` and use that port below.
2. Get film ids: `curl localhost:5000/films/` and copy two `id` values (`FILM_A`, `FILM_B`).
3. Add a film: `curl -X POST localhost:5000/watchlist/<user_id>/add -H "Content-Type: application/json" -d '{"film_id": "FILM_A"}'`. Expect **201** with `"public": true`.
4. Repeat step 3 with the same film. Expect **409** "already on this user's watchlist".
5. Add `FILM_B` privately: same call with `-d '{"film_id": "FILM_B", "public": false}'`. Expect **201** with `"public": false`. Sending `"public": "no"` should return **400**.
6. Use a made-up id (`00000000-0000-0000-0000-000000000000`). Expect **404**.
7. `curl localhost:5000/watchlist/<user_id>`. Expect FILM_B first (added last), then FILM_A, with `public` shown on each.
8. Remove: `curl -X DELETE localhost:5000/watchlist/<user_id>/remove -H "Content-Type: application/json" -d '{"film_id": "FILM_A"}'`. Expect **200**; repeating it returns **404**.
9. Run `pytest tests/ -v`. Expect all 12 tests to pass.
