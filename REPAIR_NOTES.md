# REPAIR_NOTES.md

This pass reviewed the project by actually running it (backend server, full
test suite, both frontend builds) rather than just reading the code. Four
real, verified bugs were found and fixed. All fixes are in the code; this
file just explains what changed and why.

## 1. Publish was permanently blocked (critical)

`validation_report()` required every published episode - including Season 0
trailers - to have poster, banner, AND thumbnail artwork. But
`build_catalogue()` deliberately excludes Season 0 from the browsable
`seasons` list (trailers "are not a normal season," per reference.json), and
the supplied seed data only ever gives trailers a thumbnail. Net effect: the
seed data could never publish, full stop, even after fixing the two
deliberately-bad rows the exercise asks you to find.

**Fix:** `backend/app/main.py::validation_report()` now only requires a
thumbnail for Season 0 episodes.

Verified: after fixing the two known-bad seed rows, `POST
/admin/catalog/publish` now succeeds (7 shows, 63 episodes) where it
previously always returned 422.

## 2. A pre-existing duplicate could never be resolved through the CMS

`PATCH /admin/episodes/{id}` rejected an edit if *any other* episode shared
its `(content_group, language)` pair - even when the edit didn't touch
those fields. Since the seed data ships one such duplicate on purpose (that's
the point of the exercise), the one edit that actually fixes it - flipping
the duplicate to `draft` - was itself blocked by the same check, forever.
The only way out was to delete the episode outright.

**Fix:** the conflict check in `update_episode()` now only runs when the
edit would actually change `content_group` or `language`. Editing other
fields (status, duration, title) on an episode that's already part of a
pre-existing duplicate is allowed again.

Verified with a new regression-adjacent manual test: `PATCH` the duplicate's
`status` to `draft` while keeping the same `content_group`/`language` -
previously 409, now 200.

## 3. Show tile artwork was non-deterministic

`build_catalogue()` picked a show's poster/banner via `dict.update()` across
episode groups processed in ascending order, so whichever *later* episode
happened to carry poster/banner art silently won - the opposite of the
"apply to the first episode" behavior `upload_show_artwork()` (the CMS's
show-level convenience route) actually implements.

**Fix:** switched to `setdefault()` so the first group with artwork wins,
matching what the upload route does.

## 4. Catalogue `version` was hardcoded to `1`

Every publish wrote `"version": 1`, forever - already flagged as a known
issue in `CODE_REVIEW.md`/`VERIFICATION_REPORT.md` but never actually fixed.
Nothing (CMS run history, viewer, cache-busting) could tell publishes apart.

**Fix:** `build_catalogue()` now reads the current on-disk version and
increments it. Verified: two successive publishes now go `1 -> 2 -> 3`.

## 5. The test suite only worked under one exact invocation

`backend/tests/test_core.py` resolved `data/reference.json` relative to the
current working directory while also needing `app` importable - which only
lined up under the CI invocation (`pytest backend/tests` with
`PYTHONPATH=backend`, run from the repo root). The more natural local flow,
`cd backend && pytest tests`, failed with `FileNotFoundError`.

**Fix:** added `backend/conftest.py` to put `backend/` on `sys.path`
regardless of cwd, and made the test resolve `data/reference.json` relative
to the repo root via `Path(__file__).resolve().parents[2]` instead of a
bare relative path. Both invocations now pass (verified).

Added one new regression test,
`test_trailer_episodes_only_require_thumbnail_artwork`, covering fix #1.

## 6. `npm ci` / `docker compose up --build` would fail outside this sandbox

Both `cms/package-lock.json` and `viewer/package-lock.json` had every
package's `resolved` URL pointing at `http://package-firewall.replit.local/...`
- an internal proxy from wherever this project was originally generated.
That host doesn't exist anywhere else, so `npm ci` (and therefore the
Docker image build, which runs `npm ci`) would fail immediately for anyone
running this outside that specific environment - which is to say, everyone
who receives this zip.

**Fix:** rewrote every `resolved` URL in both lockfiles to point at
`https://registry.npmjs.org/`. Verified: `npm ci && npm run build` now
completes cleanly for both `cms` and `viewer` (confirmed here, not just
inferred).

## What was verified end-to-end, not just read

- `pytest` (4/4 passing) from both `backend/` and the repo root.
- The API booted locally against sqlite, and I drove it with real HTTP
  calls: login, validation report, artwork-blocked publish (422), fixing
  the two seed issues via the CMS's own endpoints, a clean validation
  report, a successful publish, a second publish (version incremented
  again), `/catalog`, and `/catalog/search`.
- `npm ci && npm run build` for both `cms` and `viewer`, from a clean
  `node_modules`.
