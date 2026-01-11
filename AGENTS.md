# AGENTS.md — Dify Development Guide for Agentic Coding

## Project Overview

Dify is an open-source LLM application platform with three core layers:

- **Backend API** (`/api`): Python 3.11+ Flask application using Domain-Driven Design (DDD)
- **Frontend Web** (`/web`): Next.js 15 with React 19 and TypeScript 5.9+
- **Docker** (`/docker`): Containerized deployment and middleware orchestration

## Build, Lint & Test Commands

### Backend (Python)

**Package Manager**: `uv` (replaces poetry from v1.3.0+)

```bash
# Setup
cd api
uv sync --dev
uv run flask db upgrade

# Development server
uv run flask run --host 0.0.0.0 --port 5001 --debug

# Async worker (Celery)
uv run celery -A app.celery worker -P threads -c 2 --loglevel INFO -Q dataset,priority_dataset,priority_pipeline,pipeline,mail,ops_trace,app_deletion,plugin,workflow_storage,conversation,workflow,schedule_poller,schedule_executor,triggered_workflow_dispatcher,trigger_refresh_executor,retention

# Beat scheduler (for scheduled tasks)
uv run celery -A app.celery beat

# Linting & formatting
uv run ruff check --fix ./api      # Fix linting issues
uv run ruff format ./api            # Format code
make lint                           # (from root) Comprehensive: ruff + imports + dotenv-linter
make type-check                     # Type checking with basedpyright
make format                         # Format only

# Testing
uv run pytest tests/unit_tests/     # Unit tests only
uv run pytest tests/integration_tests/  # Integration tests (CI-only)
uv run pytest tests/unit_tests/path/to/test_file.py  # Single test file
uv run pytest tests/unit_tests/path/to/test_file.py::test_function_name  # Single test
make test                           # (from root) Run all unit tests via script
```

### Frontend (Node.js)

**Package Manager**: `pnpm@10.x` (enforced via `preinstall` hook)

```bash
cd web

# Setup
pnpm install

# Development server
pnpm run dev              # Runs with Turbopack, inspect flag for debugging

# Production build
pnpm run build

# Linting & formatting
pnpm lint:fix             # ESLint with auto-fix (preferred)
pnpm lint                 # ESLint without changes
pnpm lint:quiet           # ESLint quiet mode
pnpm type-check:tsgo      # TypeScript check (preferred, faster)
pnpm type-check           # tsc type checking

# Testing
pnpm test                 # Run all tests (Vitest)
pnpm test:watch           # Watch mode
pnpm test:coverage        # Coverage report
pnpm test path/to/file.spec.tsx  # Single test file

# Component Analysis
pnpm analyze-component app/components/YourComponent/index.tsx
pnpm analyze-component app/components/YourComponent/index.tsx --json
pnpm analyze-component app/components/YourComponent/index.tsx --review
```

---

## Code Style Guidelines

### Python Backend

**Type Annotations**
- Always annotate function parameters and return types
- Avoid `Any` and `Unknown`—use explicit union types or Protocol-based generics
- Example: `def fetch_user(user_id: str, session: Session) -> User | None:`

**Imports**
- Group imports: stdlib → third-party → local (sorted alphabetically within groups)
- Use absolute imports from package root: `from core.plugin.entities import Plugin`
- No relative imports (e.g., avoid `from ..utils import helper`)

**Class & Function Naming**
- Classes: `PascalCase` (e.g., `PluginService`, `WorkflowExecutor`)
- Functions: `snake_case` (e.g., `get_plugin_by_id`, `run_workflow`)
- Constants: `UPPER_SNAKE_CASE`
- Private methods/attributes: leading underscore (e.g., `_validate_config`)

**Error Handling**
- Use domain-specific exceptions (e.g., `PluginNotFoundError`, `WorkflowExecutionError`)
- Never suppress errors silently; log with context using standard logger
- Example: `logger.error("Failed to load plugin", extra={"plugin_id": plugin_id, "error": str(e)})`
- Avoid bare `except:` or generic `Exception` catches

**Special Methods**
- Implement `__repr__` and `__str__` for all domain models
- Use Pydantic `@field_validator` for input validation (not `__init__` logic)
- SQLAlchemy session usage: `with Session(db.engine) as session: ...` pattern

**File & Package Size**
- Keep files under 500 lines; split large modules
- Use service layer (`services/`) for business logic, avoid fat utils
- Organize by domain: `core/plugin/`, `core/workflow/`, `services/trigger/`, etc.

**Logging**
- Use structured logging with context: `logger.info("msg", extra={"key": value})`
- Avoid `.format()` in log messages; pass data via `extra=` dict
- No print statements in production code

### TypeScript/React Frontend

**Type Annotations**
- Strict mode enabled; no `any` types (use `unknown` if needed, then narrow)
- Explicit return types for all functions: `const getValue = (): string => { ... }`
- Use discriminated unions and type guards instead of type assertions

**Imports**
- Use path aliases (e.g., `@/components/Button` from `tsconfig.json`)
- Group imports: React/external libs → local app code → styles
- Example:
  ```typescript
  import React from 'react'
  import { useMutation } from '@tanstack/react-query'
  import { Button } from '@/components/Button'
  import styles from './Page.module.css'
  ```

**Component Naming**
- Components: `PascalCase` (e.g., `UserCard.tsx`)
- Hooks: `useXxx` naming convention (e.g., `useUserData.ts`)
- Utilities: `camelCase` (e.g., `formatDate.ts`, `classnames.ts`)

**State Management & Hooks**
- Prefer React hooks + React Query for data fetching
- Use `zustand` for global state (if needed); avoid prop drilling
- Always clean up side effects in `useEffect` cleanup functions

**Styling**
- Use Tailwind CSS classes (primary approach)
- CSS Modules for component-scoped styles when needed
- No inline `style={{}}` props; use classes instead

**Component Structure**
- Single responsibility: one component = one behavior
- Extract complex logic into custom hooks
- Group related state variables together

**Strings & i18n**
- **CRITICAL**: All user-facing strings must use i18n
- Path: `web/i18n/en-US/` (never hardcode text)
- Import: `import { useTranslation } from 'react-i18next'`
- Usage: `const { t } = useTranslation(); <span>{t('key.name')}</span>`

**Error Handling**
- Throw domain-specific error classes (e.g., `ValidationError`, `APIError`)
- Use React error boundaries for UI errors
- Never suppress errors in catch blocks without logging

---

## Testing Practices

### Backend (Python)

**Framework**: `pytest` with Arrange-Act-Assert pattern

```bash
# Run unit tests (fast, local, mocked dependencies)
uv run pytest tests/unit_tests/

# Run specific test file
uv run pytest tests/unit_tests/path/to/test_file.py

# Run specific test function
uv run pytest tests/unit_tests/path/to/test_file.py::TestClass::test_method

# With coverage
uv run pytest tests/unit_tests/ --cov=api --cov-report=html

# Integration tests (CI-only, requires full Docker stack)
uv run pytest tests/integration_tests/
```

**Guidelines**
- Organize: `tests/unit_tests/` mirrors the `api/` structure
- Mocking: Use `pytest-mock` for fixtures and patches
- Each test function tests one scenario; use `test_xxx_when_yyy` naming
- Follow AAA pattern: Arrange (setup) → Act (call function) → Assert (verify)
- Example:
  ```python
  def test_plugin_service_load_when_plugin_exists(mock_plugin_repo):
      # Arrange
      plugin_id = "test-plugin"
      mock_plugin_repo.get_by_id.return_value = Plugin(id=plugin_id)
      
      # Act
      result = PluginService(plugin_repo=mock_plugin_repo).load(plugin_id)
      
      # Assert
      assert result.id == plugin_id
      mock_plugin_repo.get_by_id.assert_called_once_with(plugin_id)
  ```

### Frontend (TypeScript)

**Framework**: Vitest 4.0.16 + React Testing Library 16.0

```bash
# Run all tests
pnpm test

# Watch mode for active development
pnpm test:watch

# Run specific file
pnpm test path/to/file.spec.tsx

# Generate coverage
pnpm test:coverage
```

**File Naming & Structure**
- Format: `ComponentName.spec.tsx` in same directory as component
- Example: `web/app/components/Button/index.tsx` → `web/app/components/Button/index.spec.tsx`
- Global setup: `web/vitest.setup.ts` (mocks, env)
- Integration tests: `web/__tests__/` directory

**Guidelines** (see `web/testing/testing.md` for complete spec)
- Single behavior per test: one assertion focus
- Black-box testing: assert observable outputs, avoid implementation details
- Naming: `should <behavior> when <condition>`
- Query order: `getByRole` > `getByText` (semantic) > `getByTestId` (last resort)
- Use `test.each()` for data-driven tests
- Example:
  ```typescript
  describe('Button', () => {
    it('should show loading state when isLoading=true', () => {
      // Arrange
      render(<Button isLoading={true}>Click me</Button>)
      
      // Act & Assert
      expect(screen.getByRole('button')).toHaveAttribute('aria-busy', 'true')
    })
  })
  ```

**Component Complexity Analysis**
- Before writing tests, run: `pnpm analyze-component <path>`
- `Complexity > 50`: Refactor first, then write integration tests
- `Complexity 30–50`: Multiple describe blocks, integration scenarios
- `Linecount > 500`: Consider splitting into smaller components

**Testing Library Setup**
- Configuration: `web/vitest.config.ts` (jsdom, path aliases, presets)
- Global mocks: `web/vitest.setup.ts` (i18n, next/image, ResizeObserver)
- Shared factories: `web/__mocks__/` (reusable mock modules)

---

## General Practices

### Architecture & DDD

**Backend**
- Organize by domain: `core/plugin/`, `core/workflow/`, `services/`, `extensions/`
- Inject dependencies via constructors (no global state)
- Separate concerns: domain logic → service layer → API routes
- Use Pydantic for input validation and serialization
- Store multi-tenant config in `configs.dify_config`, passed through `tenant_id`

**Frontend**
- Component hierarchy: pages → features → components → primitives
- State flows: React Query for server state, Zustand for UI state
- Custom hooks for reusable logic; avoid fat components (use analyzer)
- Never mutate Zustand/React Query state directly; use immutable patterns

### Documentation & Comments

- Prefer self-documenting code over comments
- Comments only for "why" decisions, not "what" the code does
- Docstrings: Module and complex function level only
- Keep i18n keys organized; avoid long-lived magic strings

### Git & Submission

- Before pushing:
  - Backend: `make lint`, `make type-check`, `make test`
  - Frontend: `pnpm lint:fix`, `pnpm type-check:tsgo`, `pnpm test`
- Commit messages: Atomic changes, clear intent
- PRs: Reference issue numbers; summarize test coverage

### Environment Setup

**Backend**
- Copy `.env.example` → `.env` and set `SECRET_KEY` (use `openssl rand -base64 42`)
- Middleware via Docker: `cd docker && docker compose -f docker-compose.middleware.yaml up -d`
- Database: `uv run flask db upgrade`

**Frontend**
- Copy `.env.example` → `.env.local`
- Set `NEXT_PUBLIC_API_PREFIX` to backend URL
- Set `NEXT_PUBLIC_COOKIE_DOMAIN` if frontend and backend are on different subdomains

---

## Additional Resources

- **Backend arch overview**: `api/AGENTS.md` (infrastructure, plugins, workflows)
- **Frontend testing**: `web/testing/testing.md` (complete test specification)
- **Frontend AGENTS**: `web/AGENTS.md` (automated test generation)
- **Integration tests**: CI-only; run locally only with full Docker stack
