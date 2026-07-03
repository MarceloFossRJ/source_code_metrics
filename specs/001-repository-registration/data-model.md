# Phase 1 Data Model: Application Shell & Repository Registration

**Feature**: 001-repository-registration | **Date**: 2026-07-03

## Entity: Repository

Represents a source code repository the app is configured to track metrics for. Persisted in
PostgreSQL via SQLModel; schema created through an Alembic migration.

### Fields

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | integer | PK, auto-increment | Surrogate key. |
| `repository_name` | string(255) | required, non-empty (trimmed) | The repository's name (without owner). |
| `owner` | string(255) | required, non-empty (trimmed) | Organisation or user that owns the repository. |
| `license` | string(100) | optional, nullable | Free-text license (e.g. "MIT", "Apache-2.0"). May be blank/null. |
| `created_at` | timestamptz | required, default now | Audit: when the record was registered. |
| `updated_at` | timestamptz | required, default now, updated on change | Audit: last modification. |

### Constraints & Indexes

- **Primary key**: `id`.
- **Uniqueness**: functional unique index on `(lower(trim(owner)), lower(trim(repository_name)))`
  — enforces case-insensitive, whitespace-trimmed uniqueness of the owner+name pair (FR-014,
  research §8). Same repository name under a different owner is allowed.
- Values are trimmed of leading/trailing whitespace before persistence.

### Validation Rules (from spec)

| Rule | Source | Enforced at |
|------|--------|-------------|
| `repository_name` and `owner` are required and non-empty after trimming | FR-013 | Pydantic schema + service; NOT NULL in DB |
| `license` is optional and may be blank | FR-013, Clarifications | Pydantic schema (Optional) |
| Owner+name pair must be unique (case-insensitive, trimmed) | FR-014 | Service pre-check + DB functional unique index |
| Repository must exist on GitHub before create/edit is persisted | FR-017 | Service (calls GitHub verification) — see [contracts/github-verification.md](./contracts/github-verification.md) |
| Field length bounds: `repository_name` ≤255, `owner` ≤255, `license` ≤100 | Spec assumptions | Pydantic schema + DB column length |

### Lifecycle / State

The Repository has a simple lifecycle — no status field in this feature:

```
(none) --register(create + GitHub verify + uniqueness)--> Registered
Registered --edit(update + GitHub verify + uniqueness)--> Registered
Registered --remove(confirm)--> (deleted, hard delete, FR-019)
```

Hard delete only (FR-019, Clarifications) — removal permanently deletes the row after explicit
user confirmation. No soft-delete/archive column in this release.

## DTOs (Pydantic schemas — `app/schemas/repository_schema.py`)

Conceptual shapes (implementation in tasks phase):

- **RepositoryCreate**: `repository_name`, `owner`, `license?` — inbound create payload; trims
  and length-validates; rejects empty required fields.
- **RepositoryUpdate**: same shape as create — inbound edit payload.
- **RepositoryRead**: `id`, `repository_name`, `owner`, `license`, `created_at`, `updated_at` —
  outbound for list/detail rendering.
- **RepositoryValidationError**: field → message map, used to render form errors (missing field,
  duplicate, not-found-on-GitHub, verification-unavailable).

## Relationships

None in this feature. `Repository` is standalone. Future metric entities are expected to
reference `Repository.id` as a foreign key (out of scope here; the surrogate `id` is chosen now
to make those future relationships clean).

## Non-entity concept: Feature (menu option)

The left-menu "feature" is a navigation concept, not persisted data. For this release the menu
is a static list defined in the template/config with a single entry — "Repositories" (active).
No table is required; documented here to note the deliberate decision not to model it in the DB.
