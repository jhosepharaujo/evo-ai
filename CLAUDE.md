# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Evo AI is an open-source platform for creating and managing AI agents. It's a full-stack app with a Python/FastAPI backend and a Next.js/React frontend. It supports multiple agent frameworks (Google ADK, CrewAI, LangGraph) and implements Google's Agent-to-Agent (A2A) protocol for agent interoperability.

## Common Commands

### Backend
```bash
make run                  # Dev server with hot reload (port 8000)
make run-prod             # Production with 4 workers
make lint                 # Flake8 (src/ and tests/)
make format               # Black formatter
make install-dev          # Install with dev dependencies
make alembic-upgrade      # Apply pending DB migrations
make alembic-revision message="desc"  # Create new migration
make alembic-migrate message="desc"   # Create + apply migration
make seed-all             # Seed DB (admin, client, mcp_servers, tools)
```

### Frontend (run from `frontend/`)
```bash
pnpm dev                  # Dev server (port 3000)
pnpm build                # Production build
pnpm lint                 # ESLint
```

### Docker
```bash
make docker-build && make docker-up   # Start all services
make docker-seed                      # Seed DB in container
```

### Testing
```bash
pytest                    # Run all tests
pytest tests/api/         # Run specific module
pytest -v --cov=src       # With coverage
```
Tests use SQLite in-memory DB via fixtures in `tests/conftest.py`.

## Architecture

### Backend (`src/`)

**Layered architecture**: Routes -> Services -> Models/DB

- **`src/main.py`** - FastAPI app init. All routes registered under `API_PREFIX = "/api/v1"`.
- **`src/api/`** - Route handlers (auth, agents, chat, sessions, clients, mcp_servers, tools, admin, a2a). Async functions with JWT protection.
- **`src/services/`** - Business logic. Key file: `agent_service.py` (~1300 lines, agent CRUD + validation).
- **`src/models/models.py`** - All SQLAlchemy ORM models in a single file. Agent types constrained to: `llm`, `sequential`, `parallel`, `loop`, `a2a`, `workflow`, `crew_ai`, `task`.
- **`src/schemas/`** - Pydantic models for request/response validation.
- **`src/config/settings.py`** - Pydantic `BaseSettings` pulling from `.env`. Singleton via `settings = get_settings()`.
- **`src/config/database.py`** - SQLAlchemy engine + session setup (PostgreSQL).
- **`src/core/jwt_middleware.py`** - JWT auth middleware. `get_current_user()`, `get_current_admin_user()`, `verify_user_client()`.

**Agent framework layer** (`src/services/adk/` and `src/services/crewai/`):
- `agent_builder.py` - Constructs agent instances from DB config
- `agent_runner.py` - Executes agents, manages sessions
- `custom_agents/` - Custom agent type implementations (A2A, workflow, task)
- Engine selected via `AI_ENGINE` env var ("adk" or "crewai")

**`src/services/service_providers.py`** - Wires up ADK or CrewAI session/artifact/memory services based on `AI_ENGINE`. Imported at startup by `main.py`.

### Frontend (`frontend/`)

Next.js 15 App Router with TypeScript, Tailwind CSS, shadcn/ui components.

- `app/(auth)/` - Login, register, forgot-password, verify-email pages
- `app/(dashboard)/` - Agents, workflows, chat, API keys, clients, MCP servers
- `services/` - API client modules (axios-based)
- `components/WorkflowEditor/` - ReactFlow-based visual workflow builder
- `middleware.ts` - JWT validation on requests

### Database

PostgreSQL with Alembic migrations in `migrations/versions/`. Key tables: `clients`, `users`, `agents`, `agent_folders`, `mcp_servers`, `tools`, `api_keys`, `audit_logs`, `sessions`.

Multi-tenant via `client_id` on most tables - users can only access resources belonging to their client.

## Code Conventions

- **Language**: ALL code, comments, docstrings, logs, error messages, and commits must be in English.
- **Formatting**: Black (line-length 88). Flake8 ignores E203, W503, E501.
- **Commits**: Conventional Commits format - `<type>(<scope>): <description>` (feat, fix, docs, style, refactor, perf, test, chore).
- **Schemas**: Pydantic `BaseModel` with `Config.from_attributes = True` for ORM models. Use `Optional` for nullable, `EmailStr` for emails.
- **Routes**: Use appropriate status codes (201 create, 204 delete). Async functions. JWT protection on protected routes. Pagination for lists.
- **Services**: Handle errors with `SQLAlchemyError`, use transactions, rollback on error, strong typing.

## Key Environment Variables

`AI_ENGINE` (adk|crewai), `POSTGRES_CONNECTION_STRING`, `REDIS_HOST`/`REDIS_PORT`, `JWT_SECRET_KEY`, `ENCRYPTION_KEY` (for API key storage), `EMAIL_PROVIDER` (sendgrid|smtp), `LANGFUSE_PUBLIC_KEY`/`LANGFUSE_SECRET_KEY`/`OTEL_EXPORTER_OTLP_ENDPOINT` (observability). See `.env.example` for full list.

## API Docs

Swagger UI at `http://localhost:8000/docs`, ReDoc at `http://localhost:8000/redoc`.
