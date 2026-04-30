---
trigger: always_on
---

# Documentation Requirements

## When to Create Documentation

You **must** create or update documentation in `docs/` for:

1. **New features** — Any new feature, endpoint, or capability added to the project.
2. **Major bug fixes** — Significant bug fixes that change behavior, fix race conditions, resolve data integrity issues, or affect user-facing functionality.

Minor cosmetic fixes, typo corrections, or trivial one-line changes do not require documentation.

## Before Making Changes

Before implementing a new feature or fixing a major bug, **always check existing documentation first**:

1. Read `docs/README.md` to understand the current documentation index.
2. Search `docs/` for any existing documentation related to the feature or area you are working on.
3. If related docs exist, plan to **update** them rather than creating duplicates.

## Documentation Format

### For New Features

Create `docs/<FEATURE_NAME>.md` with:

- **Title** — Clear feature name as H1
- **Overview** — What the feature does (1-2 sentences)
- **Implementation Details** — Key technical decisions, AI models used, data flow
- **Files Created/Modified** — List of backend, frontend, and admin files involved
- **API Endpoints** — New endpoints added (method, path, auth, description)
- **Database Changes** — New models/tables/migrations if any
- **Points Cost** — Service alias and point type if the feature consumes points

### For Major Bug Fixes

Create `docs/<BUG_NAME>_FIX.md` with:

- **Title** — Clear bug description as H1
- **Problem Summary** — What was broken and how it manifested
- **Root Cause** — Technical explanation of why the bug occurred
- **Solution** — What was changed and why
- **Files Modified** — List of files changed

## After Changes

1. Add the new doc to `docs/README.md` under the appropriate section (Feature Implementation, Bug Fix Logs, etc.).
2. If the feature introduces new API endpoints, also update `docs/API_REFERENCE.md`.
3. If the feature adds a major new capability, also update `docs/FEATURES.md`.
