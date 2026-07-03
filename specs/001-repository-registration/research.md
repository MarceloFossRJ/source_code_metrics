# Phase 0 Research: Application Shell & Repository Registration

**Feature**: 001-repository-registration | **Date**: 2026-07-03

This document resolves the open technical decisions for the feature. Each entry records the
decision, its rationale, and the alternatives considered.

---

## 1. Tailwind delivery without a Node toolchain

**Decision**: Build CSS with the **Tailwind CSS standalone CLI** (a single prebuilt binary).
Source at `app/static/css/src.css` (`@tailwind base/components/utilities` + a `@layer` of
Primer design tokens); output committed to `app/static/css/app.css`. `tailwind.config.js` sets
`darkMode: 'class'` and `content` globs over `app/templates/**/*.html` and `app/static/js/**`.

**Rationale**: Honors constitution NFR "vanilla Tailwind" and Principle V (minimal
dependencies) — no `npm`/`node_modules` in a Python project. Produces a purged, production-safe
static stylesheet served directly by FastAPI's static mount.

**Alternatives considered**:
- *Tailwind Play CDN*: zero build, but ships the full engine to the browser and is explicitly
  "not for production"; rejected.
- *npm-based Tailwind + PostCSS*: standard but adds a Node toolchain and dependency tree that
  violates minimal-dependency intent for a Python service; rejected.

---

## 2. GitHub-flavored theme (colors + fonts)

**Decision**: Define a Primer-derived token palette as CSS variables toggled by the `dark`
class, mapped into Tailwind theme colors:

| Token | Light | Dark |
|-------|-------|------|
| canvas (page bg) | `#ffffff` | `#0d1117` |
| canvas-subtle (panel bg) | `#f6f8fa` | `#161b22` |
| border | `#d0d7de` | `#30363d` |
| fg-default (text) | `#1f2328` | `#e6edf3` |
| fg-muted | `#656d76` | `#8b949e` |
| accent (links/primary) | `#0969da` | `#2f81f7` |
| success | `#1a7f37` | `#3fb950` |
| danger | `#cf222e` | `#f85149` |

Fonts: GitHub's system stack — sans `-apple-system, BlinkMacSystemFont, "Segoe UI", "Noto
Sans", Helvetica, Arial, sans-serif`; mono `ui-monospace, SFMono-Regular, "SF Mono", Menlo,
Consolas, monospace` (used for owner/repo identifiers).

**Rationale**: Reproduces GitHub's look with zero web-font downloads (system fonts) and a clean
light/dark switch via the `dark` class strategy. Tokens keep both themes in one source of truth.

**Alternatives considered**: Hardcoding hex values per element (rejected — unmaintainable, no
theming); importing Primer CSS (rejected — extra runtime dependency, conflicts with Tailwind).

---

## 3. Layout pattern (Windster/Flowbite "products" page)

**Decision**: Replicate the layout with our own Tailwind markup (no Flowbite runtime):
- **Top navbar** (fixed, full width): brand/logo + mobile hamburger on the left; dark/light
  toggle and a placeholder user avatar on the right.
- **Left sidebar** (`w-64`, fixed, below navbar): the feature menu. "Repositories" is the first
  and active item. Collapses to an off-canvas drawer under `md`.
- **Main content** (`md:ml-64`): breadcrumb bar first, then the page-header (title), then a
  panel card containing a search input + "Add repository" button, the data table, and
  pagination. Create/edit use a modal form; delete uses a confirmation modal.

**Rationale**: Directly satisfies FR-001..FR-005 (two columns; breadcrumb-then-title ordering;
active menu state) and matches the requested reference. Owning the markup avoids a component
library dependency (Principle V).

**Alternatives considered**: Adopting Flowbite JS/components (rejected — runtime dependency);
top-nav-only layout (rejected — spec mandates a left menu column).

---

## 4. Interactivity without a JS framework

**Decision**: ~40 lines of vanilla JS in `app/static/js/app.js` covering: dark-mode toggle
(persist to `localStorage`, apply `dark` class on `<html>`, honor `prefers-color-scheme` on
first load), mobile sidebar drawer, modal open/close, and disabling the submit button + showing
an inline spinner during form submission (which includes the GitHub round-trip).

**Rationale**: Constitution Principle V (minimal dependencies) and NFR "minimal front-end
dependencies". These interactions are small and well-understood; a framework is unwarranted.

**Alternatives considered**: Alpine.js / htmx (rejected — added dependency; fetch + small JS is
sufficient); server-only full-page reloads (rejected — cannot show per-component loading spinner
required by the NFR).

---

## 5. Satisfying the "loading spinner for server-rendered data" NFR

**Decision**: The Repositories page renders the shell immediately, then loads the repository
table via a client `fetch()` to a **fragment endpoint** (`GET /api/v1/repositories/table`),
showing a reusable spinner placeholder until the HTML fragment arrives. Create/edit/delete
submit via `fetch()`; the submit button shows a spinner during the request (notably during
GitHub verification), then the table fragment is re-fetched to reflect the change.

**Rationale**: Server-side Jinja rendering alone cannot show a per-component loading state; the
fragment approach keeps rendering server-side (templates render the fragments) while giving each
data component an observable loading state, satisfying the NFR and FR-006.

**Alternatives considered**: Full-page SSR with no spinner (rejected — violates NFR); a SPA
frontend (rejected — contradicts the Jinja SSR mandate and minimal-dependency goal).

---

## 6. GitHub existence verification (FR-017, FR-018)

**Decision**: `github_verification_service` uses **httpx** to call
`GET https://api.github.com/repos/{owner}/{repo}` with `Accept: application/vnd.github+json`,
a 10s timeout, and an optional `Authorization: Bearer <GITHUB_TOKEN>` when configured. Outcome
mapping:
- `200` → exists (proceed to save).
- `404` → not found → reject with "repository not found on GitHub" (FR-017).
- `403`/`429` with `X-RateLimit-Remaining: 0` → rate-limited → "verification could not be
  completed" (FR-018).
- Timeout / connection error → "verification could not be completed" (FR-018).

**Rationale**: httpx is the FastAPI-ecosystem standard HTTP client, supports timeouts and clean
test stubbing, and integrates with the same TestClient stack. A configurable token raises the
rate limit from 60/hr (unauthenticated) to 5000/hr without making a token mandatory.

**Alternatives considered**: `requests` (rejected — not async-friendly, extra dep vs. httpx
already fitting the stack); GitHub GraphQL API (rejected — heavier for a single existence
check); PyGithub SDK (rejected — large dependency for one endpoint, violates Principle V).

---

## 7. Configuration & three-database environment strategy

**Decision**: `core/config.py` uses **pydantic-settings** `BaseSettings` with `APP_ENV`
(`production` | `development` | `test`) and a `DATABASE_URL`. `.env.example` documents the three
targets:
- production → `postgresql+psycopg://.../source_code_metrics`
- development → `postgresql+psycopg://.../source_code_metrics_dev`
- test → `postgresql+psycopg://.../source_code_metrics_test`

Alembic reads the same settings so migrations target the active environment's DB. The pytest
suite forces `APP_ENV=test` (→ `source_code_metrics_test`) and wraps each test in a transaction
rolled back on teardown for isolation.

**Rationale**: Single typed settings source keeps app, migrations, and tests consistent (DIP);
per-env databases prevent test/dev data bleeding into production; transaction rollback gives
fast, isolated BDD/unit tests.

**Alternatives considered**: One shared DB with schema prefixes (rejected — weaker isolation);
SQLite for tests (rejected — behavioral divergence from PostgreSQL; constitution mandates
Postgres); env-specific hardcoded configs (rejected — not 12-factor, harder to override).

---

## 8. Duplicate detection & identity (FR-014)

**Decision**: A repository's identity is `(owner, repository_name)` compared case-insensitively
after trimming. Enforce at two levels: (a) service-layer pre-check for a friendly message, and
(b) a database **unique index on `lower(owner), lower(repository_name)`** as the source of
truth, catching races.

**Rationale**: The DB constraint guarantees correctness even under concurrent writes; the
service pre-check produces the user-facing message required by FR-014 without relying on parsing
DB errors as the primary path.

**Alternatives considered**: Service-only check (rejected — race condition, no hard guarantee);
storing a normalized `full_name` column with a plain unique index (viable alternative; deferred
in favor of a functional index to keep the original-case values intact for display).

---

## Resolved unknowns

All Technical Context items are resolved; no `NEEDS CLARIFICATION` markers remain. pgvector is
explicitly deferred (no semantic-search requirement in this feature).
