# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

For detailed domain-specific guidance, see:
- `AGENTS.md` — General development guide, commands, feature flags, PR rules
- `src/AGENTS.md` — Backend patterns, security, API development, architecture
- `tests/AGENTS.md` — Python testing patterns and best practices
- `static/AGENTS.md` — Frontend patterns, design system, React testing

## Essential Commands

**All Python commands must use the virtualenv** — prefix with `.venv/bin/` or `source .venv/bin/activate`.

### Backend

```bash
# Setup
devenv sync && direnv allow
devservices up          # Start dev dependencies (Postgres, Redis, Kafka, etc.)
devservices serve       # Start full dev server

# Testing (always use these flags)
.venv/bin/pytest -svv --reuse-db tests/sentry/path/test_file.py

# Linting (run on changed files before completing any task)
.venv/bin/pre-commit run --files src/sentry/foo.py tests/sentry/test_foo.py

# Migrations
sentry django makemigrations
sentry django migrate
./bin/update-migration <migration_name_or_number> <app_label>  # After rebase conflicts
```

### Frontend

```bash
pnpm run dev-ui                          # Dev server with hot reload
CI=true pnpm test path/to/file.spec.tsx  # Run tests (always use CI=true)
pnpm run lint:js path/to/file.tsx        # Lint specific files
pnpm run typecheck                       # Typecheck (whole project only)
```

## Architecture Overview

Sentry is a large-scale **Django + React** application for error tracking and performance monitoring.

### Backend (Python 3.13+, Django 5.2+)

- **Silo Architecture**: Two deployment silos — **Control** (user auth, billing, org management) and **Region** (project data, events, issues). Never join across silos; use outboxes for cross-silo updates. Endpoints must be decorated with `@region_silo_endpoint` or `@control_silo_endpoint`.
- **API Layer**: Django REST Framework endpoints in `src/sentry/api/endpoints/`. URLs in `src/sentry/api/urls.py`. Inherit from `OrganizationEndpoint` or `ProjectEndpoint` for proper scoping.
- **Task Queue**: Celery tasks use `@instrumented_task` decorator in `src/sentry/tasks/`.
- **Databases**: PostgreSQL (primary), Redis (cache/queue), ClickHouse via Snuba (event storage), Kafka (message queue).
- **Options System**: Centralized in `src/sentry/options/defaults.py`. Never pass default values to `options.get()` — defaults are already registered.

### Frontend (TypeScript, React 19)

- **Build**: Rspack. Package management via pnpm.
- **State**: React Query (TanStack Query) for server state. No new Reflux stores.
- **Styling**: Use core components from `@sentry/scraps/layout` (Flex, Grid, Container) and `@sentry/scraps/text` (Text, Heading). Avoid styled components for layout/typography.
- **Rules**: No class components. No CSS files. Always TypeScript. Lazy load routes.
- **Routes**: Defined in `static/app/routes.tsx`.
- **API calls**: Use `useApiQuery` from `sentry/utils/queryClient`.

## Critical Patterns

### Security: Always Scope Queries

```python
# WRONG — IDOR vulnerability
resource = Resource.objects.get(id=request.data["resource_id"])

# RIGHT — scoped to organization
resource = Resource.objects.get(id=request.data["resource_id"], organization_id=organization.id)
```

For project IDs from requests, always use `self.get_projects()` instead of trusting `request.data["project_id"]` directly.

### Serializers: Avoid N+1 Queries

Bulk-fetch data in `get_attrs()`, never query the database inside `serialize()`.

### Error Responses

Use `{"detail": "message"}` (DRF convention), not `{"error": "message"}`.

### API Design

- `snake_case` URL params, `camelCase` request/response bodies
- Return strings for numeric IDs
- Pagination via `cursor`

### Logging

Use `%`-style formatting (not f-strings) and `logger.exception()` inside exception handlers (not `logger.error()`).

### Python Typing

Abstract types for inputs (`Sequence`, `Mapping`), concrete types for returns (`list`, `dict`). Import from `collections.abc`, not `typing`.

## Testing

### Backend

- Test file location mirrors source: `src/sentry/foo/bar.py` → `tests/sentry/foo/test_bar.py`
- Use factory methods (`self.create_project()`, `self.create_organization()`) — never `Model.objects.create()`
- Use `pytest` style — no `unittest` assertions
- API tests extend `APITestCase` with `endpoint = "sentry-api-0-endpoint-name"`

### Frontend

- Import from `sentry-test/reactTestingLibrary`, not `@testing-library/react`
- Use `screen.getByRole()` as primary selector
- Use `userEvent` not `fireEvent`
- Use `MockApiClient.addMockResponse()` for API mocking — don't mock hooks
- Use fixtures (`ProjectFixture()`, `OrganizationFixture()`) — don't construct types manually

## Feature Flags

Register in `src/sentry/features/temporary.py`, check with `features.has("organizations:my-feature", organization, actor=user)`. Frontend: `organization.features.includes('my-feature')` (requires `api_expose=True`). Tests: `with self.feature("organizations:my-feature"):`.

## PR Rules

Frontend (`static/`) and backend (`src/`, `tests/`) are **not atomically deployed**. Split cross-cutting changes into separate PRs. Land backend first when frontend depends on new API changes.
