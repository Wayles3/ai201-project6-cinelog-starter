# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude Code (an AI coding agent) throughout this project, so I'm being specific about where:

- **Orientation:** Claude read `models.py`, `services/collection_service.py`, and `tests/test_collection.py` before touching anything, and summarized the naming, dedup, and fixture patterns. I checked its summary of `add_to_collection()` against the actual code (film lookup, then `filter_by(...).first()`, then raise `AlreadyInCollectionError`).
- **Implementation and git hygiene:** Claude made the code changes for Comments 1–3, ran the rebase onto `origin/main`, resolved the `models.py` conflict, reworded the original "added watchlist model…" commit with an interactive rebase, and implemented the three stretch features. I reviewed each diff and the test output.
- **Comments 4 and 5:** Claude drafted the first versions of these two arguments from the code and README context. I reviewed them against CineLog's behavior and they reflect my position, but the initial wording is AI-generated, so treat them as AI-assisted rather than independently written. A reviewer should weigh them accordingly.
- **Verification:** Claude ran the pytest suite and exercised every endpoint with curl against a scratch database (201/409/404/400/200 paths) before I wrote the manual test steps below.

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
**My position:** Keep `public=True` as the default, and make privacy an explicit opt-out through the new `public` field on `POST /watchlist/<user_id>/add`.

**Reasoning:** CineLog is described as a *community* film tracking app: the collection already exists so people can log, rate, and show what they've watched. A watchlist is the forward-looking half of the same profile ("what I'm planning to watch"), and its main social value is being seen: friends can recommend a film from it, and it shows a user's taste before they've logged much. The default should serve the typical user, who is adding a film like *Spirited Away* or *Alien* with no privacy concern, and who will never open a settings screen. If the default were private, most watchlists would sit invisible forever because almost nobody changes defaults, and the community half of the feature would never get used. Defaulting to public also matches the existing `WatchlistEntry.public = True` column default, so the model, service, and endpoint all agree.

**Tradeoff acknowledged:** Private-by-default is the safer choice for privacy. A watchlist can say something personal (a film someone is embarrassed to be interested in, or something sensitive), and a public default means a user who doesn't notice the setting exposes that. If a user adds a film and only later learns it was public, that is a real cost. I'm accepting it for three reasons: the exposure is limited to film titles, not contact info or ratings; the opt-out is a single boolean on the same request (`"public": false`); and the response echoes `public` back so clients can show the current state. Two follow-ups I'd want before launch: (1) a PATCH-style endpoint to flip visibility on an existing entry, and (2) enforcing `public` when someone other than the owner reads a watchlist. Today `GET /watchlist/<user_id>` has no concept of who is viewing, so the flag is stored and returned but not yet filtered on. That enforcement belongs in a separate PR with an auth story, and I'd say so explicitly rather than let the flag imply protection that doesn't exist yet. If the team prefers privacy-first, flipping the default is a one-line change in the model and `add_to_watchlist()`, so this is a cheap decision to revisit.

## Comment 5 — Sort order
**My position:** I agree with the maintainer: sort the watchlist by date added, newest first. I changed `get_watchlist()` from `Film.title.asc()` to `WatchlistEntry.date_added.desc()`, and added `test_get_watchlist_returns_newest_first`.

**Reasoning:** A watchlist is a queue of intent, and intent is freshest right after the user adds something. The common session is: you hear about a film, add it, and then you (or a friend viewing your profile) want to confirm it landed and see what you most recently got excited about. Newest-first puts that at the top with no scrolling. It also makes `get_watchlist()` consistent with `get_collection()`, which already sorts `date_added.desc()`, so the two lists in the same app behave the same way and clients can treat them the same. Alphabetical order only helps when someone is hunting for a specific title in a long list, and that is a search problem, not a default-order problem.

**Engagement with reviewer's point:** The maintainer's claim is "most users want to see what they added recently." I agree, with one honest caveat: I don't have usage data for CineLog, so this rests on how queue-like lists behave elsewhere, not on measurement. The case where the maintainer's argument is weakest is a user with a long, old watchlist who wants to find a particular film, where alphabetical is genuinely better. My synthesis: ship newest-first as the default now, and if users ask for it, add an optional `?sort=title` parameter later rather than making alphabetical the default. I did not add that parameter in this PR because nobody has asked for it and it would widen the review.

## Comment 6 — Rebase
**What conflicted:** `feature/watchlist` was branched before main's UUID refactor (57ec3f9). `git rebase origin/main` stopped with a content conflict in `models.py` on `WatchlistEntry.film_id`: main had `db.String(36)`, my branch had `db.Integer` plus `index=True`.
**How I resolved it:** Kept main's `String(36)` type and my `index=True`. After the rebase finished, I removed the remaining integer references on my side (the `add_to_watchlist` docstring said `film_id (int) ... pre-refactor`, and the route docstring documented `"film_id": <int>`) and committed that separately as `fix: update WatchlistEntry film_id to UUID after main branch refactor`.
**How I verified no conflict remains:** `git status` clean with no rebase in progress, no conflict markers (`grep -rn "<<<<<<<"`), `git log --merges` is empty, and `pytest tests/ -v` passes using UUID film IDs.

## Stretch features
**`remove_from_watchlist(user_id, film_id)`:** Follows `remove_from_collection()` exactly: it looks up the entry with `WatchlistEntry.query.filter_by(user_id=..., film_id=...).first()`, deletes and commits it, and returns `True`. If the film isn't on the watchlist it raises `NotInWatchlistError` (mirroring `NotInCollectionError`), which `DELETE /watchlist/<user_id>/remove` maps to 404; a missing `film_id` returns 400. The name follows the project's `verb_to_noun` convention. Tests: `test_remove_from_watchlist_deletes_entry` and `test_remove_from_watchlist_not_present_raises`.

**Second test (edge case beyond the review):** `test_add_to_watchlist_same_film_for_different_users_allowed`. I chose it because Comment 2's dedup check is the riskiest new logic: if I had written the `filter_by` with only `film_id` (forgetting `user_id`), the existing duplicate test would still pass, but the second user would be wrongly blocked from saving a film someone else already saved. This test pins the check to `(user_id, film_id)`. I also added `test_add_to_watchlist_public_defaults_true_and_can_be_private` for the visibility parameter below.

**Visibility toggle:** `add_to_watchlist(user_id, film_id, public=True)` takes a `public` argument that is stored on the new `WatchlistEntry`. `POST /watchlist/<user_id>/add` accepts an optional `"public"` field in the JSON body. It defaults to `true` (see Comment 4), and a non-boolean value (e.g. `"no"`) returns 400 rather than being silently coerced. Example: `{"film_id": "<uuid>", "public": false}` creates a private entry.

## Git history
<!-- TODO (you): paste your `git log --oneline` screenshot here after the final history is settled -->

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
