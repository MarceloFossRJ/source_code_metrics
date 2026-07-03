# Implementation Plan: Application Shell & Repository Registration

**Branch**: `001-repository-registration` | **Date**: 2026-07-03 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/001-repository-registration/spec.md`

## Summary

Deliver the first vertical slice of "Source code metrics": a responsive application shell
(fixed left feature menu + right content area that always renders breadcrumbs then the feature
title) and the "Repositories" feature — full CRUD for the repositories the app will track
(`repository_name`, `owner`, `license`). Every create/edit verifies the repository exists on
GitHub before persisting. The UI uses a custom Tailwind theme built on GitHub's Primer color
tokens and system font stack, laid out like the Windster/Flowbite admin "products" page, with
light and dark modes. Data lives in PostgreSQL across three environment databases
(`source_code_metrics`, `source_code_metrics_dev`, `source_code_metrics_test`), evolved with
Alembic. Behavior is specified with pytest-bdd-ng and correctness driven by pytest (TDD).

## Technical Context

**Language/Version**: Python 3.13 (`requires-python >=3.13.13`)

**Primary Dependencies**:
- Runtime: FastAPI, Uvicorn, Jinja2, SQLModel (+ SQLAlchemy), Pydantic v2, pydantic-settings,
  Alembic, psycopg (v3, sync driver), httpx (GitHub existence verification).
- UI build (dev-only, not a runtime dependency): Tailwind CSS **standalone CLI** (single
  binary — no Node/npm). No Flowbite/Alpine; interactions use ~40 lines of vanilla JS.
- Tooling/tests: pytest, pytest-bdd-ng, pytest-cov, Prospector, FastAPI `TestClient`,
  respx or httpx MockTransport for GitHub stubbing.

**Storage**: PostgreSQL. Three databases: `source_code_metrics` (production),
`source_code_metrics_dev` (development), `source_code_metrics_test` (automated tests). Schema
managed exclusively via Alembic migrations. pgvector is **not** used in this feature (no
semantic-search need yet).

**Testing**: pytest (unit), pytest-bdd-ng (BDD `.feature` files + step defs). GitHub calls are
stubbed in tests; the DB layer runs against `source_code_metrics_test` with per-test rollback.

**Target Platform**: Linux server (containerable), served by Uvicorn; browsers on desktop and
tablet (≈768px and up).

**Project Type**: Server-rendered web application (FastAPI + Jinja2), layered per constitution.

**Performance Goals**: Single-user / single-tenant internal tool. Page/fragment renders in
<300ms p95 excluding the external GitHub call; GitHub verification bounded by a 10s timeout.

**Constraints**: Minimal dependencies (constitution V); vanilla Tailwind (no component-library
runtime dep); every server-data-rendering component shows a loading spinner (NFR); light + dark
mode; responsive from ≈768px up; layered architecture with no logic in routes and no raw data
access outside repositories.

**Scale/Scope**: One feature (Repositories) + shell. ~1 entity, ~7 routes, tens–hundreds of
repository rows expected. Menu designed to accept future features without restructuring.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Assessment | Status |
|-----------|------------|--------|
| I. BDD + Test-First (BDD/TDD) | BDD `.feature` specs per user story (shell nav, create+list, edit, delete, GitHub-verify); unit tests written before implementation; full suite green before commit. | PASS |
| II. Clean Code & SOLID + Prospector | Layered separation enforces SRP/DIP; `.prospector.yaml` added and run as a gate; all offenses fixed pre-commit. | PASS |
| III. Layered Architecture (FastAPI) | Exact `app/{api,services,repositories,models,schemas,factories,core,db,templates,static}` layout; routes = presentation only, services = logic, repositories = data access, Pydantic DTOs at boundaries. | PASS |
| IV. Persistent, Migrated Data | PostgreSQL via SQLModel; all schema changes via Alembic; base data relational. pgvector intentionally deferred (no semantic search). | PASS |
| V. Minimal Dependencies & Prescribed Stack | Only stack-aligned additions (httpx, pydantic-settings, psycopg, uvicorn). Tailwind via standalone CLI = dev tool, not runtime dep. No Flowbite. Each new dep justified in research.md. | PASS |
| NFRs (Tailwind UI, responsive, loading spinners) | Custom Primer-based Tailwind theme; fragment-loaded lists + submit spinners satisfy the loading-state rule; responsive Windster-style layout; dark/light mode added per user request. | PASS |

**Result**: PASS — no violations. Complexity Tracking left empty.

## Project Structure

### Documentation (this feature)

```text
specs/001-repository-registration/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   ├── web-routes.md         # HTTP routes the app exposes (pages + fragments + form posts)
│   └── github-verification.md # GitHub REST contract consumed for existence checks
└── tasks.md             # Phase 2 output (/speckit-tasks — NOT created here)
```

### Source Code (repository root)

```text
app/
├── api/
│   └── v1/
│       ├── endpoints/
│       │   └── repositories.py     # routes: pages, table fragment, create/edit/delete
│       └── router.py               # aggregates v1 routers
├── services/
│   ├── repository_service.py       # CRUD business logic, duplicate + validation rules
│   └── github_verification_service.py  # existence check against GitHub
├── repositories/
│   └── repository_repo.py          # data access for Repository
├── models/
│   └── repository.py               # SQLModel table model
├── schemas/
│   └── repository_schema.py        # Pydantic DTOs (create/update/read/validation errors)
├── factories/
│   └── app_factory.py              # create_app(): wiring, templates, static, routers
├── core/
│   └── config.py                   # pydantic-settings: APP_ENV, DATABASE_URL, GITHUB_*
├── db/
│   └── database.py                 # engine + session dependency
├── static/
│   ├── css/
│   │   ├── src.css                 # @tailwind directives + Primer @layer tokens
│   │   └── app.css                 # built output (Tailwind standalone CLI)
│   ├── js/
│   │   └── app.js                  # dark-mode toggle, modal + sidebar drawer, submit spinner
│   └── images/
└── templates/
    ├── base.html                   # shell: navbar + sidebar + content slot
    ├── partials/
    │   ├── sidebar.html            # feature menu (Repositories = first item)
    │   ├── breadcrumb.html
    │   ├── page_header.html        # breadcrumb + title block
    │   ├── spinner.html            # reusable loading spinner
    │   ├── repository_table.html   # table fragment (or empty state)
    │   ├── repository_form.html    # create/edit modal form
    │   └── repository_delete.html  # delete confirmation modal
    └── repositories/
        └── index.html             # Repositories page (extends base, spinner placeholder)

alembic/
├── env.py
├── script.py.mako
└── versions/
    └── <ts>_create_repository_table.py

tests/
├── unit/
│   ├── test_repository_service.py
│   ├── test_repository_repo.py
│   ├── test_github_verification_service.py
│   └── test_repository_schema.py
└── behavior/
    ├── features/
    │   ├── application_shell.feature
    │   ├── register_repository.feature
    │   ├── edit_repository.feature
    │   ├── delete_repository.feature
    │   └── github_verification.feature
    ├── steps/
    └── conftest.py

main.py                 # entrypoint: app = create_app()
alembic.ini
tailwind.config.js      # darkMode:'class'; Primer tokens; content globs over templates
.prospector.yaml
.env.example            # APP_ENV + DATABASE_URL per environment + GITHUB_TOKEN
changelog.md
pyproject.toml          # deps managed with uv (replaces the template's requirements.txt)
```

**Structure Decision**: Adopt the constitution's prescribed FastAPI layered layout verbatim
(`app/api|services|repositories|models|schemas|factories|core|db|templates|static`) with
`tests/{unit,behavior}`. Alembic lives at repo root (`alembic/` + `alembic.ini`). The
constitution template's `requirements.txt` and `app/db/*.db` (sqlite) slots are superseded by
`pyproject.toml` (uv) and PostgreSQL respectively — deliberate, stack-consistent deviations.

## Complexity Tracking

> No constitution violations. Section intentionally empty.
