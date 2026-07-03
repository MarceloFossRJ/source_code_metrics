<!--
SYNC IMPACT REPORT
==================
Version change: [TEMPLATE / unversioned] → 1.0.0
Rationale: Initial ratification. Template placeholders replaced with concrete,
project-specific governance for source_code_metrics (git_metrics tool).

Modified principles: N/A (initial population)
Added principles:
  - I. Behavior-Driven & Test-First Development (BDD + TDD)
  - II. Clean Code & SOLID Design
  - III. Layered Architecture (FastAPI)
  - IV. Persistent, Migrated Data
  - V. Minimal Dependencies & Prescribed Stack
Added sections:
  - Technology Stack & Non-Functional Requirements
  - Development Workflow & Git Discipline
  - Governance

Templates requiring updates:
  - .specify/templates/plan-template.md ✅ (generic "Constitution Check" gate — no change needed)
  - .specify/templates/spec-template.md ✅ (no principle-specific coupling — no change needed)
  - .specify/templates/tasks-template.md ✅ (test tasks optional/generic — no change needed)
  - .specify/templates/checklist-template.md ✅ (no change needed)

Follow-up TODOs: None. (Input's "Python 3.43" resolved to Python 3.13 by user.)
-->

# source_code_metrics Constitution

`source_code_metrics` extracts engineering metrics from a team's GitHub repositories into a
local PostgreSQL database so engineering managers, teams, and stakeholders can prepare
operational reviews and track delivery/quality trends over time. These principles are
non-negotiable and govern every contribution.

## Core Principles

### I. Behavior-Driven & Test-First Development (BDD + TDD)

Behavior MUST be described in human-readable language before implementation. Every
feature MUST have BDD specifications (pytest-bdd-ng) expressing functional and
non-functional requirements, and code-centric correctness MUST be driven by unit tests
(pytest) written test-first following Red-Green-Refactor.

- BDD scenarios (Given/When/Then) define WHAT the system must do; they validate
  business requirements.
- TDD unit tests define HOW the code behaves; tests are written before implementation,
  fail first, then pass.
- The full test suite MUST pass before any commit. No task is complete with failing or
  skipped tests.

**Rationale**: BDD keeps the system anchored to business value; TDD keeps the code
correct and refactorable. Together they prevent regressions and untested behavior.

### II. Clean Code & SOLID Design

All code MUST follow Clean Code practices and the SOLID principles:

- **S** — Single Responsibility: each module/class has one reason to change.
- **O** — Open/Closed: open for extension, closed for modification.
- **L** — Liskov Substitution: subtypes are substitutable for their base types.
- **I** — Interface Segregation: no client depends on methods it does not use.
- **D** — Dependency Inversion: depend on abstractions, not concretions.

Prospector MUST be run as the static-analysis gate, and ALL reported offenses MUST be
fixed before commit.

**Rationale**: Readable, decoupled code with a clean static-analysis baseline keeps the
codebase maintainable as metrics coverage grows.

### III. Layered Architecture (FastAPI)

The application MUST follow the prescribed layered structure with clear separation of
concerns. Dependencies flow inward toward business logic:

- **api/** — controllers/routes (presentation logic only); versioned under `api/v1`.
- **services/** — business logic.
- **repositories/** — data access.
- **models/** — SQLModel database models.
- **schemas/** — Pydantic DTOs for validation and serialization.
- **factories/** — object creation (app factory pattern).
- **core/** — global configuration (env, security).
- **templates/** + **static/** — Jinja2 server-side rendering and assets.

Routes MUST NOT contain business logic; services MUST NOT perform raw data access
outside repositories. Pydantic MUST be used for all type management, JSON serialization,
and validation at boundaries.

**Rationale**: A strict layered boundary enforces SRP and DIP at the architectural
level and keeps presentation, logic, and persistence independently testable.

### IV. Persistent, Migrated Data

Base data MUST be stored in the relational database (PostgreSQL). SQLModel is the ORM.
Every schema change MUST be applied through an Alembic migration — no manual or
out-of-band schema edits. pgvector MUST be used for semantic-search storage only where
a semantic-search need exists; it MUST NOT replace relational storage for base data.

**Rationale**: Migrations make schema evolution reproducible and reviewable; keeping
base data relational preserves query integrity and auditability of metrics over time.

### V. Minimal Dependencies & Prescribed Stack

The project MUST use the minimum set of dependencies necessary. New third-party
dependencies MUST be justified against existing capabilities in the prescribed stack.
The prescribed stack (see Technology Stack section) is authoritative; substitutions
require a constitution amendment.

**Rationale**: A small, deliberate dependency surface reduces security exposure,
maintenance burden, and build fragility.

## Technology Stack & Non-Functional Requirements

**Language & Frameworks** (authoritative; changes require amendment):

- Python 3.13.
- FastAPI web framework with Jinja2 server-side templating.
- SQLModel (ORM), Alembic (migrations), Pydantic (types/validation/serialization).
- PostgreSQL with pgvector extension for semantic search.
- pytest (unit) and pytest-bdd-ng (BDD); Prospector (static analysis).

**Non-Functional Requirements**:

- Modern web interface built with vanilla Tailwind CSS.
- Simple UX, responsive design, minimal front-end dependencies.
- Every page that renders server-side data MUST display a loading-state spinner for the
  component being rendered.

## Development Workflow & Git Discipline

**Branching** — NEVER commit directly to `main` or `develop`. Every feature, fix, or
task MUST be developed in its own branch named `feature/<desc>`, `fix/<desc>`, or
`chore/<desc>`. One pull request per feature branch.

**Commits** — Conventional Commits (https://www.conventionalcommits.org/) with clear,
imperative messages. Group logically related changes into a single commit.

**Per-task workflow** (each task in a spec plan):

1. Ensure you are on the correct feature branch.
2. Implement the task.
3. Write or update unit tests and BDD specs for the task.
4. Run Prospector and fix all offenses.
5. Run the full test suite — all specs MUST pass.
6. Update the version in the project `pyproject.toml`.
7. Update `changelog.md`.
8. Stage and commit.
9. Tag the version in git.

**Versioning & Changelog** — Releases follow Semantic Versioning
(https://semver.org/). Every merge to `main` bumps the version and records changes in
`changelog.md` following Common Changelog (https://common-changelog.org/).

## Governance

This constitution supersedes all other development practices. When guidance conflicts,
the constitution wins.

- **Amendments** MUST be proposed via pull request documenting the change and rationale,
  and MUST update the version and this document's amendment date.
- **Versioning policy** (this document): MAJOR for backward-incompatible governance or
  principle removals/redefinitions; MINOR for a new principle/section or materially
  expanded guidance; PATCH for clarifications and non-semantic refinements.
- **Compliance** — Every PR and review MUST verify adherence to these principles
  (BDD+TDD gate, Prospector clean, layered boundaries, migrations for schema changes,
  minimal dependencies). Any added complexity MUST be justified in the PR.
- Runtime and agent development guidance lives alongside this constitution; where such
  guidance exists it MUST NOT contradict these principles.

**Version**: 1.0.0 | **Ratified**: 2026-07-03 | **Last Amended**: 2026-07-03
