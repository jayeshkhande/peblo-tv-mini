# Peblo TV Mini

An end-to-end take-home implementation of the Peblo TV publishing flow:

```text
CMS (React) ──► FastAPI + PostgreSQL ──► atomic catalogue.json ──► Viewer (React)
```

## Run it

Requirements: Docker Desktop (or Python 3.12 + Node 20 for local development).

```bash
cp .env.example .env
docker compose up --build
```

Open:

- Viewer: http://localhost:5174
- CMS: http://localhost:5173
- API docs: http://localhost:8000/docs
- Health: http://localhost:8000/health

Demo credentials:

| Role | Email | Password |
| --- | --- | --- |
| Admin | `admin@example.com` | `admin-demo` |
| Editor | `editor@example.com` | `editor-demo` |

The editor can manage catalogue data but cannot publish. The admin can publish.

## Run the backend tests

```bash
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pytest tests
```

(`conftest.py` puts `backend/` on `sys.path` and the tests resolve `data/reference.json`
relative to the repo root, so this works the same whether you run `pytest tests` from
`backend/` or `pytest backend/tests` from the repo root, which is what CI does.)

## What is included

- FastAPI API with SQLAlchemy models for shows, seasons, episodes, artwork, users, and publish runs.
- PostgreSQL in Compose, with SQLite as the convenient local default.
- Server-side artwork validation for the exact target dimensions/aspect ratios and 200 KB ceiling.
- Local storage adapter with a small interface that can be replaced by an R2 adapter.
- Seed import from the supplied 95-row dataset and reference rules.
- Atomic, deterministic publication with content-group language collapsing and Season 0 exclusion.
- CMS list/search/filter/pagination, editor forms, upload previews, validation report, and publish history.
- Viewer home, hero, section rows, search filters, show details, loading/error/empty states.
- Focused pytest coverage for artwork validation, grouping, Season 0, auth, and atomic writes.
- Docker Compose and GitHub Actions CI.

## Data findings

The supplied seed has 95 episode rows across 8 shows. It contains 10 draft rows, 2 trailer rows in Season 0, 18 English/Hindi content groups, 1 episode with no artwork, and 8 draft episodes without a section under `Rhyme Rangers`. The CMS validation report exposes those issues instead of silently repairing them.

## Decisions and trade-offs

**Atomic publishing.** The publisher serializes a complete catalogue to a temporary file in the same directory, flushes and `fsync`s it, then uses `os.replace`. A reader sees either the previous complete file or the new complete file. If the process dies before the replace, the temporary file is cleaned up and the previous catalogue remains. The publish run is recorded with counts and outcome.

**Storage.** `LocalStorage` implements `put`, `url`, and `delete` behind a `Storage` protocol. Moving to Cloudflare R2 requires an R2 implementation plus environment-backed bucket credentials; API routes and artwork metadata do not change.

**Search.** The API searches the published in-memory catalogue snapshot for title, episode title, and category, then composes category, language, and section filters. This is intentionally simple and fast for a small pre-published catalogue. At hundreds of thousands of entries it should move to a search index (or PostgreSQL full-text/trigram search) and keep the same API contract.

**Why a catalogue file?** Viewer traffic does not need joins, auth, or database availability for every browse request. A complete immutable snapshot is cacheable and makes a publish a clear product boundary. The cost is eventual consistency: edits are not visible until an admin publishes, and the snapshot must be rebuilt when the schema changes.

**Skipped / kept small.** No real video playback, cloud deployment, audit log, rollback UI, or production SSO is included because those are outside the core scoring path. A deploy job is documented in CI rather than pointing at a fake cloud. The app keeps the seeded demo authentication deliberately explicit.

**AI usage.** AI assistance was used to accelerate scaffolding and review the requirements. The implementation was kept only where it matched the supplied reference data; the seed validation findings, publishing rules, permission checks, and file-write behavior were verified against the challenge rather than accepted blindly.

## Production notes

Use a managed secret store for `DATABASE_URL`, `JWT_SECRET`, and R2 credentials; never commit `.env`. Alert on `/health` failures and on publish failures because a stale catalogue is customer-visible while the database may still look healthy.

## Approximate time budget

Architecture/data inspection 30 min · API and publishing 2 h · CMS 90 min · viewer 60 min · tests/Compose/README 60 min.
