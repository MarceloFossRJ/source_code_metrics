# Feature Specification: Application Shell & Repository Registration

**Feature Branch**: `001-repository-registration`

**Created**: 2026-07-03

**Status**: Draft

**Input**: User description: "This is an application that should be listing source code repositories metrics. The app is called 'Source code metrics'. As a first step I want to have the initial page setup. The page should have two columns - a left one where the menu options (features) should reside, and a right one where the pages from the selected menu option should display. The right page should always show breadcrumbs first, and below the feature title. The first menu option should be the registration of repositories the app should track metrics. It should be a CRUD functionality with the following fields: repository_name, owner (organisation/user that owns the repository), license. Design: follow GitHub design, colors and fonts."

## Clarifications

### Session 2026-07-03

- Q: How should the "license" field be captured for a repository? → A: Optional free-text (may be left blank; no enforced list of values)
- Q: When a user removes a registered repository, what should happen to the record? → A: Hard delete (record permanently removed after explicit confirmation)
- Q: What validation should apply to repository_name and owner values? → A: Verify the repository exists on GitHub before saving (in addition to light non-empty/length checks)

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Navigate the application shell (Priority: P1)

As a user of "Source code metrics", I open the application and see a consistent two-column
layout: a left column listing the available features (menu), and a right column that
displays the page for the currently selected feature. The right column always shows a
breadcrumb trail at the top, and the feature title directly below it, so I always know
where I am and can move between features.

**Why this priority**: The shell is the foundation every other feature renders inside. Without
it there is no place to present repository registration or any future metric feature. It is
the first thing a user sees and establishes the product's navigation model.

**Independent Test**: Can be fully tested by opening the application, confirming the left menu
lists at least one feature, selecting it, and verifying the right column shows the breadcrumb
trail followed by the feature title above the feature content — delivering a usable, navigable
product frame.

**Acceptance Scenarios**:

1. **Given** the application is open, **When** the initial page loads, **Then** a left column
   with a feature menu and a right content column are both visible.
2. **Given** the shell is displayed, **When** a feature is selected in the left menu, **Then**
   the right column updates to that feature's page and the selected menu item is visually
   marked as active.
3. **Given** a feature page is displayed in the right column, **When** the page renders,
   **Then** a breadcrumb trail appears first and the feature title appears immediately below it.
4. **Given** the right column is loading server-provided data, **When** the data has not yet
   arrived, **Then** a loading-state indicator is shown for the component being rendered.

---

### User Story 2 - Register and view tracked repositories (Priority: P1)

As a user, I select the "Repositories" feature from the left menu and register the source code
repositories I want the app to track metrics for. I provide the repository name, the owner
(the organisation or user that owns it), and its license, and I can see the list of all
repositories I have registered so far.

**Why this priority**: Registering repositories is the entry point for the entire product —
no metrics can be collected or reviewed until the user has declared which repositories to
track. Combined with the shell, create + list is the minimum viable slice that delivers value.

**Independent Test**: Can be fully tested by opening the Repositories feature, adding a new
repository with name, owner, and license, and confirming it appears in the list of registered
repositories.

**Acceptance Scenarios**:

1. **Given** the Repositories feature is open, **When** I submit a new repository with a
   repository name, an owner, and a license, **Then** the repository is saved and appears in
   the list of registered repositories.
2. **Given** the Repositories feature is open, **When** I view the list, **Then** each entry
   displays its repository name, owner, and license.
3. **Given** I am registering a repository, **When** I omit a required field, **Then** the
   submission is rejected and I see a clear message indicating what is missing.
4. **Given** a repository with the same owner and repository name already exists, **When** I
   try to register it again, **Then** the system prevents the duplicate and informs me.
5. **Given** I submit a repository whose owner/name does not exist on GitHub, **When** I try
   to save it, **Then** the system rejects the submission and tells me the repository could
   not be found on GitHub.

---

### User Story 3 - Edit a registered repository (Priority: P2)

As a user, I correct or update the details of a repository I previously registered — for
example fixing a misspelled name, changing the owner, or updating the license.

**Why this priority**: Data entered by hand will contain mistakes and repositories change
ownership or license over time. Editing keeps the tracked set accurate, but the product is
still usable for an initial round without it.

**Independent Test**: Can be fully tested by selecting an existing repository, changing one or
more of its fields, saving, and confirming the updated values are reflected in the list.

**Acceptance Scenarios**:

1. **Given** a registered repository exists, **When** I open it for editing and change its
   name, owner, or license and save, **Then** the updated values are persisted and shown in
   the list.
2. **Given** I am editing a repository, **When** I change its owner and name to match another
   existing repository, **Then** the system prevents the change and informs me of the conflict.
3. **Given** I am editing a repository, **When** I clear a required field and save, **Then** the
   change is rejected with a clear message.

---

### User Story 4 - Remove a registered repository (Priority: P3)

As a user, I remove a repository I no longer want the app to track, so the list reflects only
the repositories that are currently relevant.

**Why this priority**: Housekeeping matters for a clean, accurate list, but it is the least
urgent CRUD operation for an initial release.

**Independent Test**: Can be fully tested by selecting a registered repository, confirming its
removal, and verifying it no longer appears in the list.

**Acceptance Scenarios**:

1. **Given** a registered repository exists, **When** I choose to remove it and confirm,
   **Then** it is deleted and no longer appears in the list.
2. **Given** I choose to remove a repository, **When** I am asked to confirm, **Then** the
   deletion only happens after I explicitly confirm it.

---

### Edge Cases

- What happens when no repositories have been registered yet? The list shows an empty state
  that invites the user to register the first repository.
- How does the system handle leading/trailing whitespace or differing letter case in owner and
  repository name when checking for duplicates? Names are trimmed and compared case-insensitively
  so "Owner/Repo" and "owner/repo" are treated as the same repository.
- What happens when the user submits an overly long value for a field? The system rejects values
  that exceed the allowed length with a clear message.
- How does the system behave while server data is still loading? A loading-state indicator is
  displayed for the affected component until data arrives.
- What happens when the user navigates directly to a feature page via its breadcrumb path? The
  correct feature page renders with its breadcrumb trail and title.
- What happens when the owner/repository name does not exist on GitHub? The submission is
  rejected with a clear "repository not found on GitHub" message and nothing is saved.
- How does the system behave if GitHub cannot be reached (network error, timeout, or rate
  limit) during verification? The submission is not saved, and the user is told verification
  could not be completed and can retry.

## Requirements *(mandatory)*

### Functional Requirements

#### Application Shell

- **FR-001**: The application MUST present a two-column layout with a left column for the
  feature menu and a right column for feature content.
- **FR-002**: The left menu MUST list the available features and visually indicate which
  feature is currently selected.
- **FR-003**: Selecting a feature in the left menu MUST display that feature's page in the
  right column.
- **FR-004**: Every feature page in the right column MUST render a breadcrumb trail as the
  first element and the feature title immediately below the breadcrumb trail.
- **FR-005**: The breadcrumb trail MUST reflect the user's current location within the
  application (e.g., the app followed by the current feature).
- **FR-006**: Any feature page that renders server-provided data MUST display a loading-state
  indicator for the component being rendered until the data is available.
- **FR-007**: The interface MUST follow the visual style of GitHub — its color palette,
  typography, and general layout conventions — and MUST be responsive across common desktop
  and tablet screen widths.
- **FR-008**: "Repositories" MUST appear as the first feature in the left menu.

#### Repository Registration (CRUD)

- **FR-009**: Users MUST be able to register a repository by providing a repository name, an
  owner (the organisation or user that owns the repository), and a license.
- **FR-010**: Users MUST be able to view a list of all registered repositories, with each
  entry showing repository name, owner, and license.
- **FR-011**: Users MUST be able to edit the repository name, owner, and license of an existing
  registered repository.
- **FR-012**: Users MUST be able to remove a registered repository, subject to an explicit
  confirmation step before deletion.
- **FR-013**: The system MUST require repository name and owner for every repository and reject
  submissions missing either, presenting a clear message identifying the missing field. The
  license field is optional and MAY be left blank.
- **FR-014**: The system MUST prevent registering two repositories with the same owner and
  repository name (compared after trimming whitespace and ignoring letter case) and inform the
  user of the conflict.
- **FR-015**: The system MUST persist registered repositories so they remain available across
  sessions and page reloads.
- **FR-016**: The system MUST present an empty-state message when no repositories have been
  registered.
- **FR-017**: On registration and on edit, the system MUST verify that a repository with the
  given owner and repository name exists on GitHub before saving; if it does not exist, the
  system MUST reject the submission with a clear "repository not found on GitHub" message and
  MUST NOT persist the record.
- **FR-018**: If GitHub cannot be reached to perform the existence check (network error,
  timeout, or rate limit), the system MUST NOT save the record and MUST inform the user that
  verification could not be completed, allowing them to retry.
- **FR-019**: Removing a repository MUST permanently delete the record (hard delete) after the
  user explicitly confirms the removal.

### Key Entities *(include if feature involves data)*

- **Repository**: A source code repository the user wants the app to track metrics for. Key
  attributes: repository name, owner (the organisation or user that owns the repository), and
  license. Identified uniquely by the combination of owner and repository name.
- **Feature (menu option)**: A navigable area of the application shown in the left menu and
  rendered in the right column. For this release the only feature is "Repositories".

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: From the initial page, a user can locate and open the Repositories feature in a
  single interaction (one click/selection from the left menu).
- **SC-002**: A user can register a new repository in under 1 minute, providing name, owner,
  and license.
- **SC-003**: 100% of feature pages display a breadcrumb trail followed by the feature title,
  in that order, before any feature content.
- **SC-004**: 100% of repository create, edit, and delete actions are reflected in the
  repository list immediately after the action completes.
- **SC-005**: Attempts to register a duplicate repository (same owner and name) are prevented
  100% of the time, with a message explaining why.
- **SC-006**: Every page that loads server-provided data shows a loading-state indicator until
  the data appears, verified across all feature pages.
- **SC-007**: The layout remains usable and readable, with no horizontal overflow of primary
  content, on screen widths from tablet (≈768px) to desktop.
- **SC-008**: Attempts to register or save a repository that does not exist on GitHub are
  rejected 100% of the time, with no record persisted.

## Assumptions

- This release delivers the application shell plus the full CRUD for repository registration,
  including verifying that each repository exists on GitHub at registration/edit time. Actual
  metric collection and any deeper GitHub integration (fetching commits, pull requests, or
  other metric data) are out of scope and handled by later features.
- Verifying a repository's existence on GitHub relies on GitHub being reachable; access
  credentials/rate-limit handling for that check are a design detail resolved during planning.
- "Repositories" is the only menu feature in this release; the menu is designed to accommodate
  additional features later without restructuring.
- The license field is optional free-text describing the repository's license (e.g., "MIT",
  "Apache-2.0"); no enforced list of allowed licenses is required for this release.
- The application is single-user / single-tenant for this release; user accounts, authentication,
  and per-user separation of repositories are out of scope.
- The GitHub visual reference (design, colors, fonts) applies to the look and feel; exact assets
  are chosen during design/implementation and are not part of this specification.
- Duplicate detection is based on the owner + repository name pair; the same repository name
  under different owners is allowed.
- Reasonable field length limits are applied to text inputs to prevent abuse; exact limits are a
  design detail.
