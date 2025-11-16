# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A2A Python SDK is a library for building agentic applications that follow the [Agent2Agent (A2A) Protocol](https://a2a-protocol.org). The SDK supports multiple transport protocols (HTTP/REST, gRPC, JSON-RPC) and provides both client and server implementations.

**Key Technologies:**
- Python 3.10+ with async/await
- `uv` for package management
- Pydantic v2 for data validation
- FastAPI/Starlette for HTTP servers
- gRPC for bidirectional streaming
- SQLAlchemy for optional database backends

## Development Commands

### Setup and Dependencies
```bash
# Install all dependencies including dev tools and all optional extras
uv sync --dev --extra all

# Install only core dependencies with specific extras
uv sync --dev --extra http-server --extra grpc

# Update lock file
uv lock
```

### Testing
```bash
# Run all tests
uv run pytest

# Run tests with coverage (CI requires 88% minimum)
uv run pytest --cov=a2a --cov-report term --cov-fail-under=88

# Run specific test file
uv run pytest tests/client/test_client.py

# Run specific test function
uv run pytest tests/client/test_client.py::test_function_name

# Run tests with specific markers
uv run pytest -m asyncio
```

**Note:** Integration tests requiring databases (PostgreSQL, MySQL) will be skipped unless database services are available. The CI environment provides these services.

### Code Quality

```bash
# Run linter with auto-fix
uv run ruff check --fix

# Run linter with unsafe fixes (more aggressive)
uv run ruff check --fix --unsafe-fixes

# Format code
uv run ruff format

# Format only changed files (recommended for large repos)
./scripts/format.sh

# Format all files
./scripts/format.sh --all

# Type checking with mypy
uv run mypy src

# Type checking with pyright
uv run pyright

# Run all pre-commit hooks manually
pre-commit run --all-files
```

### Code Generation

```bash
# Regenerate A2A types from specification (v0.3.0 default)
./scripts/generate_types.sh src/a2a/types.py

# Generate types from specific A2A spec version
./scripts/generate_types.sh --version v1.2.0 src/a2a/types.py

# Generate types from local JSON schema file
./scripts/generate_types.sh --input-file local/a2a.json src/a2a/types.py

# Regenerate gRPC code from proto definitions
buf generate
```

### Building and Publishing

```bash
# Build distribution packages
uv build

# Build wheel only
uv build --wheel

# Build sdist only
uv build --sdist
```

## Architecture Overview

The SDK is divided into client-side and server-side components with shared types and utilities.

### Client Architecture (`src/a2a/client/`)

**Core Components:**
- `Client`: Main async client for interacting with A2A agents
- `ClientFactory`: Factory for creating client instances with different transports
- `A2ACardResolver`: Resolves agent cards from URLs or objects

**Transports** (`client/transports/`):
- `rest.py`: HTTP/REST transport using httpx
- `grpc.py`: gRPC bidirectional streaming transport
- `jsonrpc.py`: JSON-RPC over HTTP transport

**Authentication** (`client/auth/`):
- `CredentialService`: Manages authentication credentials
- `AuthInterceptor`: Intercepts requests to add auth headers
- Support for API keys, OAuth, custom schemes

**Key Pattern:** Clients use a transport-agnostic interface. Transport selection happens at initialization via `ClientFactory` based on agent card interfaces.

### Server Architecture (`src/a2a/server/`)

**Agent Execution** (`server/agent_execution/`):
- `AgentExecutor`: Abstract base class - implement this to define your agent's logic
- `RequestContext`: Provides request metadata (task_id, message, user, etc.)
- `EventQueue`: Publish task updates, messages, and artifacts

**Request Handlers** (`server/request_handlers/`):
- `RequestHandler`: Abstract interface for A2A protocol methods
- `DefaultRequestHandler`: Production-ready implementation
- `jsonrpc_handler.py`, `rest_handler.py`, `grpc_handler.py`: Protocol-specific handlers

**Task Management** (`server/tasks/`):
- `TaskStore`: Persist task state (in-memory or database-backed)
- `TaskManager`: Orchestrates task lifecycle
- `PushNotificationSender`: Sends webhook notifications for task updates
- Database implementations support PostgreSQL, MySQL, SQLite

**Event System** (`server/events/`):
- `EventQueue`: Async queue for agent-generated events
- `EventConsumer`: Processes events and updates task state
- `QueueManager`: Manages multiple event queues

**Server Apps** (`server/apps/`):
- `rest/fastapi_app.py`: FastAPI integration for HTTP/REST
- `jsonrpc/fastapi_app.py`: FastAPI integration for JSON-RPC
- `jsonrpc/starlette_app.py`: Starlette integration for JSON-RPC
- These wrap request handlers and provide HTTP endpoints

**Key Pattern:** Implement `AgentExecutor.execute()` with your agent logic. The executor receives a `RequestContext` and publishes events to an `EventQueue`. The framework handles protocol translation, task persistence, and streaming responses.

### Shared Components

**Types** (`src/a2a/types.py`):
- Auto-generated from A2A specification JSON schema
- Contains all A2A protocol types (Task, Message, AgentCard, etc.)
- **DO NOT EDIT MANUALLY** - regenerate using `generate_types.sh`

**Utils** (`src/a2a/utils/`):
- `proto_utils.py`: Converters between Pydantic models and protobuf
- `errors.py`: A2A-specific exception types
- `telemetry.py`: OpenTelemetry integration helpers
- `message.py`, `parts.py`, `artifact.py`: Helper functions for constructing A2A objects

**gRPC** (`src/a2a/grpc/`):
- Auto-generated protobuf code (`*_pb2.py`, `*_pb2_grpc.py`, `*_pb2.pyi`)
- **DO NOT EDIT MANUALLY** - regenerate using `buf generate`
- Excluded from linting and formatting

## Code Style Guidelines

This project follows the **Google Python Style Guide** with enforcement via Ruff and pre-commit hooks:

- **Line length:** 80 characters
- **Indentation:** 4 spaces
- **Quotes:** Single quotes for inline strings, double quotes for docstrings
- **Docstrings:** Google style (see `tool.ruff.lint.pydocstyle` in pyproject.toml)
- **Imports:** Absolute imports only, sorted via isort
- **Type hints:** Required for all public functions and methods
- **Async:** Use async/await consistently; this is an async-first codebase

**Pre-commit hooks** run automatically on commit and enforce:
- Ruff linting and formatting
- No implicit Optional types
- Python 3.10+ syntax via pyupgrade
- Removal of unused imports (autoflake)
- Conventional commit messages
- Secret detection (gitleaks)

## Testing Conventions

- Test files mirror source structure: `tests/client/test_client.py` tests `src/a2a/client/client.py`
- Use `pytest-asyncio` for async tests with `@pytest.mark.asyncio`
- Use `pytest-mock` for mocking dependencies
- Use `respx` for mocking HTTP requests in client tests
- Integration tests live in `tests/integration/` and may require external services
- E2E tests live in `tests/e2e/`

**Coverage:** Maintain minimum 88% code coverage. Excluded from coverage:
- Generated code (`src/a2a/grpc/`)
- `__init__.py` files
- Abstract methods and NotImplementedError branches
- TYPE_CHECKING blocks

## Optional Dependencies

The SDK uses extras for optional features. When working on specific components:

- `http-server`: FastAPI, Starlette - needed for server HTTP apps
- `grpc`: gRPC libraries - needed for gRPC client/server
- `telemetry`: OpenTelemetry - needed for tracing integration
- `encryption`: Cryptography - needed for encrypted agent cards
- `postgresql`, `mysql`, `sqlite`: Database drivers - needed for database-backed task stores
- `sql`: All SQL drivers combined
- `all`: Everything above

**Development:** Always use `uv sync --dev --extra all` to ensure you have all optional dependencies for testing.

## Important Constraints

1. **Generated Files:** Never manually edit `src/a2a/types.py` or `src/a2a/grpc/*`. Regenerate using scripts.

2. **Backward Compatibility:** This is a published SDK. Breaking changes to public APIs require major version bumps.

3. **Protocol Compliance:** Changes must maintain A2A protocol compliance. Reference the [specification](https://a2a-protocol.org/latest/specification/).

4. **Async Consistency:** All I/O operations must be async. Do not introduce blocking calls.

5. **Optional Import Handling:** Components with optional dependencies must gracefully handle missing imports (see `client/__init__.py` for A2AGrpcClient pattern).

## Workflow Tips

**When adding a new feature:**
1. Check if it requires protocol changes (coordinate with A2A spec repo)
2. Add types to `types.py` if needed (regenerate from updated spec)
3. Implement in appropriate module (client/server/utils)
4. Add comprehensive tests with async support
5. Update docstrings with Google style
6. Run full test suite and linters before committing
7. Pre-commit hooks will auto-format your code

**When fixing a bug:**
1. Write a failing test that reproduces the issue
2. Fix the implementation
3. Verify the test passes and coverage is maintained
4. Check if documentation needs updates

**When updating dependencies:**
1. Modify `pyproject.toml` dependencies or dev dependencies
2. Run `uv lock` to update `uv.lock`
3. Run full test suite to ensure compatibility
4. Pre-commit hooks will validate the lock file is in sync
