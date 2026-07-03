# Quickstart & Validation Guide

**Feature**: 001-repository-registration | **Date**: 2026-07-03

How to set up, run, and validate the application shell + repository registration feature.
See [plan.md](./plan.md), [data-model.md](./data-model.md), and [contracts/](./contracts/) for
design detail — this guide is the runnable validation path only.

## Prerequisites

- Python 3.13, `uv` installed.
- PostgreSQL running locally (or reachable), with permission to create databases.
- Tailwind standalone CLI binary available (downloaded during setup; no Node required).
- Optional: a `GITHUB_TOKEN` (classic or fine-grained, public-repo read) to raise the GitHub
  rate limit. The feature works without one at 60 requests/hour.

## Setup

1. **Install dependencies** (managed via `pyproject.toml` / uv):
   ```bash
   uv sync
   ```

2. **Create the three databases**:
   ```bash
   createdb source_code_metrics
   createdb source_code_metrics_dev
   createdb source_code_metrics_test
   ```

3. **Configure environment** — copy `.env.example` to `.env` and set `APP_ENV` +
   `DATABASE_URL` (and optional `GITHUB_TOKEN`). For local development:
   ```
   APP_ENV=development
   DATABASE_URL=postgresql+psycopg://<user>:<pass>@localhost:5432/source_code_metrics_dev
   # GITHUB_TOKEN=ghp_xxx   # optional
   ```

4. **Run migrations** against the active database:
   ```bash
   uv run alembic upgrade head
   ```

5. **Build the stylesheet** (Tailwind standalone CLI → `app/static/css/app.css`):
   ```bash
   ./tailwindcss -i app/static/css/src.css -o app/static/css/app.css --minify
   # add --watch during development
   ```

6. **Run the app**:
   ```bash
   uv run uvicorn main:app --reload
   ```
   Open http://127.0.0.1:8000 — you are redirected to `/repositories`.

## Manual validation scenarios

Map to the spec's user stories and success criteria.

### Shell & navigation (US1 — FR-001..FR-008, SC-001, SC-003)
1. Load `/repositories`. **Expect**: two-column layout — left feature menu with "Repositories"
   marked active, right content area.
2. In the right column, **expect** a breadcrumb trail first ("Home / Repositories"), then the
   feature title "Repositories" directly below it.
3. While the table loads, **expect** a loading spinner in the table area (FR-006, SC-006).
4. Toggle dark/light mode in the navbar. **Expect** the theme switches and persists on reload.
5. Narrow the window to ~768px. **Expect** the sidebar collapses to a drawer; no horizontal
   overflow of primary content (SC-007).

### Register + list (US2 — FR-009, FR-010, FR-013, FR-016, FR-017; SC-002, SC-008)
6. With no repositories, **expect** an empty-state message inviting the first registration.
7. Click "Add repository". Submit with a blank `owner`. **Expect** rejection with a clear
   "owner is required" message (FR-013).
8. Add a real repository (e.g. owner `octocat`, name `Hello-World`, license `MIT`). **Expect**
   a submit spinner during GitHub verification, then the row appears in the table showing name,
   owner, and license (SC-004).
9. Add a non-existent repository (e.g. owner `octocat`, name `this-does-not-exist-xyz`).
   **Expect** rejection: "repository not found on GitHub"; nothing is saved (FR-017, SC-008).
10. Re-add `octocat/Hello-World`. **Expect** duplicate rejection (FR-014, SC-005).

### Edit (US3 — FR-011)
11. Edit an existing repository's license and save. **Expect** the updated value in the list
    (SC-004). Editing to duplicate another repo's owner+name is rejected (FR-014).

### Delete (US4 — FR-012, FR-019)
12. Click Delete on a row. **Expect** a confirmation modal; the row is removed only after
    confirming, and permanently (hard delete).

### GitHub verification unavailable (FR-018)
13. (Simulated) With GitHub unreachable/rate-limited, attempt to add a repository. **Expect**
    "verification could not be completed, please retry"; nothing saved.

## Automated test validation

- **Unit** (`tests/unit/`): `uv run pytest tests/unit`
- **Behavior** (`tests/behavior/`): `uv run pytest tests/behavior` — runs the `.feature` specs
  (shell, register, edit, delete, github_verification). GitHub is stubbed; DB uses
  `source_code_metrics_test` with per-test rollback.
- **Full suite + coverage**: `uv run pytest --cov=app`
- **Static analysis (gate)**: `uv run prospector` — must report zero offenses (Principle II).

**Definition of done for this feature**: all BDD scenarios and unit tests green, Prospector
clean, and the manual scenarios above verified in the browser in both light and dark modes.
