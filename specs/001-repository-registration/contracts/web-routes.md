# Contract: Web Routes

**Feature**: 001-repository-registration | **Date**: 2026-07-03

Routes the FastAPI application exposes. HTML routes render Jinja2 templates; fragment routes
return HTML partials for progressive loading (spinner NFR); mutation routes accept form data and
return the updated table fragment or a form re-render with validation errors.

All feature routes are mounted under the v1 router. Static assets are served at `/static`.

---

## Navigation / shell

### `GET /`
- **Purpose**: Entry point.
- **Response**: `307` redirect to `/repositories`.

### `GET /repositories`
- **Purpose**: Render the Repositories page inside the app shell.
- **Response** `200 text/html`: full page — navbar + sidebar (Repositories active) + main area
  containing breadcrumb ("Home / Repositories"), page title ("Repositories"), the search +
  "Add repository" controls, and a **spinner placeholder** where the table will be injected.
- **Notes**: Satisfies FR-001..FR-005, FR-008. The table itself is loaded via the fragment
  route below so the component shows a loading state (FR-006).

---

## Repository data (fragments)

### `GET /api/v1/repositories/table`
- **Purpose**: Return the repository list as an HTML fragment.
- **Query**: `q?` (search filter over owner/name), `page?` (default 1), `page_size?` (default 10).
- **Response** `200 text/html`:
  - Table rows: each shows `repository_name`, `owner`, `license` (or "—" when blank), and row
    actions (Edit, Delete). Includes pagination controls ("Showing X–Y of N").
  - **Empty state** when no repositories exist (FR-016): a message inviting the first
    registration.
- **Maps to**: FR-010, FR-016, SC-004.

### `GET /api/v1/repositories/new`
- **Purpose**: Return the create form (modal body) fragment.
- **Response** `200 text/html`: form with empty `repository_name`, `owner`, `license` fields.

### `GET /api/v1/repositories/{id}/edit`
- **Purpose**: Return the edit form (modal body) fragment, pre-filled.
- **Response**: `200 text/html` pre-filled form; `404` if the repository does not exist.
- **Maps to**: FR-011.

---

## Repository mutations

### `POST /api/v1/repositories`
- **Purpose**: Create a repository.
- **Request**: form fields `repository_name` (required), `owner` (required), `license` (optional).
- **Behavior**: validate required/length → check uniqueness → **verify existence on GitHub** →
  persist.
- **Responses**:
  - `200/201 text/html`: success — returns the refreshed table fragment (client closes modal).
  - `422 text/html`: validation failure — returns the form fragment with field errors. Cases:
    - missing required field (FR-013)
    - duplicate owner+name (FR-014)
    - repository not found on GitHub → "repository not found on GitHub" (FR-017)
    - GitHub verification unavailable → "verification could not be completed, please retry"
      (FR-018)
- **Maps to**: FR-009, FR-013, FR-014, FR-015, FR-017, FR-018; SC-002, SC-005, SC-008.

### `POST /api/v1/repositories/{id}` (update)
- **Purpose**: Update an existing repository. (`PUT` acceptable; form posts use `POST` with the
  id in the path.)
- **Request**: same fields as create.
- **Behavior**: same validation → uniqueness (excluding self) → GitHub verify → persist.
- **Responses**: `200 text/html` refreshed table fragment on success; `422 text/html` form with
  errors; `404` if id not found.
- **Maps to**: FR-011, FR-013, FR-014, FR-017, FR-018.

### `POST /api/v1/repositories/{id}/delete`
- **Purpose**: Permanently delete a repository (hard delete) after user confirmation.
- **Behavior**: The delete confirmation modal gates this call client-side; server deletes the
  row.
- **Responses**: `200 text/html` refreshed table fragment on success; `404` if id not found.
- **Maps to**: FR-012, FR-019; SC-004.

---

## Cross-cutting response conventions

- Validation errors are rendered inline in the returned form fragment (no reliance on browser
  alerts), each tied to its field or a form-level banner.
- All list/table renders include the loading-state contract: the client shows the shared spinner
  partial until the fragment resolves (FR-006, SC-006).
- Content type is `text/html` for all UI routes (server-rendered Jinja fragments), consistent
  with the Jinja SSR mandate; no JSON API is introduced in this feature.
