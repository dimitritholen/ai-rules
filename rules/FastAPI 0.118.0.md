# FastAPI 0.118.0 Production-Ready Coding Rules

> Comprehensive best practices for building enterprise-grade FastAPI applications (Mid-2025)
>
> These rules synthesize current industry standards from OWASP, major tech companies (Microsoft, Netflix, Uber), official FastAPI documentation, and production deployments.

---

## 1. Project Structure & Architecture

### Project Organization

**MUST organize domain logic inside a single root package** (typically `app/` or `src/`), with each feature module containing consistent files: `router.py`, `schemas.py`, `models.py`, `service.py`, `dependencies.py`, and `exceptions.py`.

**MUST use relative imports between modules within the same package** (e.g., `from ..dependencies import get_db`) to maintain portability and avoid coupling to specific deployment structures.

**NEVER mix infrastructure concerns with domain logic** - business logic in service layer must be expressible in plain Python without importing FastAPI or ORM libraries.

### Layered Architecture

**MUST enforce unidirectional dependency flow**: Presentation → Application/Service → Domain → Infrastructure. Inner layers (domain) cannot import from outer layers (routers, dependencies). Use dependency inversion with abstract interfaces at layer boundaries.

**ALWAYS separate Commands (writes) from Queries (reads)** - command operations use rich domain entities and value objects, while query operations can bypass service layer for optimized read-only data transfer objects.

**ALWAYS keep controllers (routers) thin with zero business logic** - routers should only handle HTTP concerns (extracting request data, calling service methods, formatting responses). Move all conditional logic, loops, and calculations to service layer.

### Dependency Injection

**ALWAYS use constructor injection over FastAPI's `Depends()` for core business logic** - reserve `Depends()` exclusively for HTTP layer concerns (authentication, database sessions, request context). Framework-agnostic DI containers enable easier testing and framework migration.

**MUST implement singleton pattern for shared resources using lifespan events** - database connection pools, cache clients, and external service clients should be initialized once during application startup, not per-request.

**ALWAYS chain dependencies to avoid duplication** - FastAPI automatically caches dependency results within a single request scope, so dependencies requiring the same sub-dependency will reuse the computed value.

### Router Organization

**MUST configure router metadata (prefix, tags, dependencies, responses) at instantiation or inclusion time** - keep individual route handlers thin, moving shared configuration to `APIRouter()` initialization for consistency.

**NEVER duplicate validation logic between dependencies and schemas** - use Pydantic schemas for data shape validation, reserve dependencies exclusively for business rule validation requiring database queries or external service calls.

**MUST use explicit module-level imports with descriptive aliases** when importing cross-domain services (e.g., `from src.auth import constants as auth_constants`) to prevent naming collisions and improve code readability in large codebases.

---

## 2. Security & Authentication

### Authentication & Authorization

**ALWAYS hash passwords using Argon2 or bcrypt with salts** - Never store plaintext passwords. Use pwdlib with Argon2 (recommended) or passlib with bcrypt. Generate secure secret keys with `openssl rand -hex 32`.

**MUST implement JWT token expiration with refresh tokens** - Set access tokens to expire within 30 minutes, refresh tokens within 7 days. Use HS256 algorithm and store JWT secrets in environment variables, never in code.

**MUST validate object-level authorization on every endpoint** - For any endpoint receiving an object ID and performing actions, explicitly check that the authenticated user has permission for that specific object. BOLA (API1:2023) is the #1 OWASP API security risk.

**ALWAYS use dependency injection (Depends) for authentication** - Leverage HTTPBearer with Depends() for token validation. Never manually parse Authorization headers. This enables reusable, testable auth logic.

**MUST implement role-based access control with explicit permission checks** - Use FastAPI dependencies to enforce permissions at the route level. Check user roles/permissions before resource access, not just authentication status.

### Input Validation & Injection Prevention

**ALWAYS use Pydantic models for request/response validation** - Never accept raw dictionaries or strings. Define explicit schemas with type annotations. Pydantic automatically validates and rejects malformed data.

**MUST use SQLAlchemy ORM with parameterized queries** - Never construct SQL queries with string concatenation or f-strings. Use SQLAlchemy's ORM or Core with bound parameters to prevent SQL injection.

**NEVER return full objects without explicit field filtering** - Use Pydantic response models with explicit field inclusion to prevent excessive data exposure (API3:2023). Don't rely on client-side filtering.

**MUST validate and sanitize file uploads** - Check file types, sizes, and content. Use allowlists for extensions. Store uploads outside the web root. Implement antivirus scanning for production.

### Security Headers & CORS

**MUST configure restrictive CORS in production** - Never use `allow_origins=["*"]` or `allow_credentials=True` with wildcard origins. Explicitly list trusted domains. When credentials are enabled, specify exact origins, methods, and headers.

**ALWAYS implement security headers middleware** - Use fastapi-armor or Secweb to set: Content-Security-Policy, Strict-Transport-Security (HSTS), X-Frame-Options: DENY, X-Content-Type-Options: nosniff, and Referrer-Policy.

**MUST enforce HTTPS/TLS in production** - Set HSTS headers with max-age of at least 1 year. Configure Strict-Transport-Security with includeSubDomains. Never serve APIs over HTTP in production.

### Rate Limiting & Monitoring

**ALWAYS implement rate limiting on authentication endpoints** - Use slowapi or similar to limit login attempts (e.g., 5/minute per IP). Protect against credential stuffing and brute force attacks.

**MUST implement API-wide rate limiting** - Apply tiered rate limits based on authentication status. Authenticated users get higher limits. Track by user ID or API key, not just IP address.

**NEVER log sensitive data** - Exclude passwords, tokens, API keys, PII, and financial data from logs. Implement log scrubbing. Use structured logging with sanitization filters before production deployment.

---

## 3. Database & ORM Patterns

### Session Management & Dependencies

**ALWAYS use `expire_on_commit=False` when creating async session factories** - Expired attributes cannot be lazy-loaded in async contexts after commit, causing detached instance errors when objects are accessed outside their session scope.

**ALWAYS close async sessions using `async with` context managers or explicit `try/finally` blocks with `await session.close()`** - FastAPI's dependency injection with `yield` guarantees cleanup execution, but explicit session closure prevents connection leaks.

**NEVER reuse sessions across requests** - Create one session per request via dependency injection. Session reuse causes connection pool exhaustion and transaction state corruption.

**NEVER use sync database operations in async route handlers** - Blocking I/O in async routes prevents the event loop from processing other tasks, causing request timeouts under load.

### Connection Pooling

**ALWAYS configure `pool_size` and `max_overflow` based on your concurrency requirements** - Total concurrent connections = `pool_size + max_overflow`. For async applications, set `pool_size` to match expected concurrent requests (typical: 20-50) and `max_overflow` for burst capacity (typical: 10-20).

**ALWAYS enable `pool_pre_ping=True` for production deployments** - Pre-ping validates connections on checkout, preventing stale connection errors. Note this doesn't protect against mid-transaction disconnects.

### Query Optimization & N+1 Prevention

**ALWAYS use explicit eager loading (`selectinload()`, `joinedload()`) instead of lazy loading for async relationships** - Set `lazy="raise"` or `lazy="raise_on_sql"` on relationships to catch accidental lazy loads. Use `selectinload()` for one-to-many/many-to-many, `joinedload()` for many-to-one relationships.

**NEVER access relationship attributes without eager loading in async contexts** - Async sessions cannot perform implicit lazy loads. Either use eager loading strategies or the `AsyncAttrs` mixin (SQLAlchemy 2.0.13+) with `awaitable_attrs`.

### Transaction Management

**ALWAYS use `async with session.begin()` for explicit transaction control** - This context manager automatically commits on success and rolls back on exceptions. Manual commit/rollback should only be used when transaction boundaries span multiple operations with conditional logic.

**NEVER perform blocking operations inside async transactions** - Database transactions hold connections from the pool. Blocking operations (file I/O, sync HTTP calls) during transactions cause connection starvation.

### Migration & Deployment

**ALWAYS run Alembic migrations before application startup, not during FastAPI startup events** - Long-running migrations can trigger gunicorn worker timeouts. Run migrations in your Docker entrypoint or deployment pipeline before starting the application server.

**ALWAYS use `disable_existing_loggers=False` in Alembic's `env.py` when using FastAPI** - Change `fileConfig(config.config_file_name)` to `fileConfig(config.config_file_name, disable_existing_loggers=False)` to prevent logging conflicts between Alembic and FastAPI.

---

## 4. REST API Design

### Resource Naming & Structure

**ALWAYS use plural nouns for resource collections** - Use `/users`, `/orders`, not `/user`, `/getUsers`. HTTP methods already convey the action.

**MUST organize URIs hierarchically to reflect resource relationships** - Use `/users/{userId}/orders/{orderId}` to show ownership. Keep hierarchies shallow (max 2-3 levels).

**NEVER use verbs in endpoint paths** - Use `POST /orders` not `/createOrder`. The HTTP method defines the action.

### HTTP Methods & Status Codes

**MUST use semantically correct HTTP status codes** - Use `201 Created` for POST success, `204 No Content` for DELETE, `400 Bad Request` for validation errors, `404 Not Found` for missing resources, `401` for authentication failures, `403` for authorization failures.

**ALWAYS raise HTTPException with descriptive messages for expected errors** - Provide clear, actionable error messages to help clients debug issues.

### Response Models & Validation

**MUST separate request models from response models** - Create distinct Pydantic models (e.g., `UserCreate`, `UserResponse`) for security and clarity. Response models should include additional fields like `id`, timestamps.

**ALWAYS use `response_model` parameter explicitly** - Declare `response_model` in route decorators even when type hints exist. This ensures proper validation, filtering, and OpenAPI documentation.

**MUST implement pagination for collection endpoints** - Use limit/offset or cursor-based pagination. Include metadata in responses: `total`, `page`, `page_size`, `items`. Consider using `fastapi-pagination` library.

**MUST filter response data through response_model** - Even if your function returns more data, `response_model` ensures only declared fields are exposed, preventing accidental data leakage.

### Versioning & Documentation

**ALWAYS version APIs using URL path prefixes** - Use `/v1/users`, `/v2/users` with APIRouter organization. URL versioning is clearest for consumers and maintains backward compatibility.

**MUST leverage Pydantic Field for constraints and OpenAPI documentation** - Use `Field(max_length=100, description="...")` to auto-generate detailed Swagger UI docs and enforce validation.

### Architecture & Dependencies

**ALWAYS use async dependencies for non-blocking operations** - Prefer `async def` for dependencies involving I/O. FastAPI caches dependency results within request scope—leverage this for repeated validations.

---

## 5. Performance & Scalability

### Async/Await Patterns

**NEVER use blocking I/O inside `async def` routes** - Even one blocking call will stall the entire event loop and prevent other requests from being processed. Use `async def` only when all I/O operations are non-blocking with `await`.

**ALWAYS offload blocking operations to ThreadPoolExecutor** - When using legacy libraries or synchronous DB drivers in async routes, wrap blocking calls with `run_in_executor()` to prevent event loop blocking.

**ALWAYS use async database drivers** - Use asyncpg for PostgreSQL, motor for MongoDB, or async SQLAlchemy instead of synchronous drivers. This delivers measurably higher RPS and lower latency under concurrent load.

### Database Optimization

**MUST implement connection pooling with proper sizing** - Configure `pool_size` and `max_overflow` in SQLAlchemy/asyncpg. Reusing connections eliminates per-request connection overhead, with typical improvements showing 3-10x reduction in database query latency.

**MUST configure pool lifecycle parameters** - Set `pool_recycle` (typically 3600 seconds) and `pool_timeout` to prevent stale connections and connection exhaustion under sustained load.

**ALWAYS use `asyncio.gather()` for concurrent database queries** - When fetching independent data sources, execute queries concurrently rather than sequentially to reduce total latency by the number of parallel operations.

### Caching Strategies

**MUST implement Redis caching with appropriate TTLs for repeated queries** - Recent benchmarks show 3x faster response times for cached endpoints. Use cache-aside pattern with TTLs based on data volatility (seconds for real-time, minutes for semi-static, hours for static).

**ALWAYS use FastAPI background tasks for cache warming after responses** - Return the response immediately, then update cache outside the request cycle to avoid blocking the client while maintaining fresh cache data.

### Response Streaming

**MUST use StreamingResponse with chunked generators for files >10MB** - Loading entire files into memory causes memory spikes. Chunked streaming (8KB-64KB chunks) maintains constant memory usage regardless of file size.

**NEVER use `yield from file` in streaming responses** - Use explicit chunk reading with `file.read(chunk_size)` in a generator. Benchmarks show measurably faster performance than `yield from`.

### Worker and Deployment

**MUST run multiple Uvicorn workers behind Gunicorn** - Single-worker deployments are CPU-bound. Use `(2 × CPU cores) + 1` workers as baseline. This directly scales throughput linearly with available cores.

**ALWAYS enable uvloop for production deployments** - Uvloop provides 2-4x performance improvement over asyncio's default event loop with zero code changes, particularly for high-concurrency scenarios (1000+ concurrent connections).

---

## 6. Testing

### Test Infrastructure

**ALWAYS clean up dependency_overrides after each test** - Use `app.dependency_overrides.clear()` in fixture teardown or use pytest-fastapi-deps context manager to prevent test contamination across modules.

**MUST use httpx.AsyncClient with ASGITransport for testing async endpoints** - TestClient's sync magic breaks inside async test functions; instead use `AsyncClient(transport=ASGITransport(app=app), base_url="http://test")`.

**ALWAYS use nested transaction rollback pattern for database tests** - Set up a transaction with SAVEPOINT support, let application code call `session.commit()` (which only ends nested transaction), then rollback the entire transaction after the test to maintain hermetic isolation.

**NEVER use scope="session" or scope="module" for client or database fixtures** - Use scope="function" to guarantee test isolation; the performance gain isn't worth the risk of test contamination.

### Test Execution

**MUST mark async tests with @pytest.mark.anyio** - This ensures proper event loop management and prevents "RuntimeError: This event loop is already running" errors.

**ALWAYS override dependencies at the fixture level, not at module level** - Setting overrides at the root of test files creates global state that bleeds into subsequent tests; confine overrides to fixture scope with proper cleanup.

**MUST use async context manager pattern for AsyncClient fixtures** - Yield within `async with AsyncClient()` block to ensure proper socket closure and resource cleanup.

### Test Strategy

**ALWAYS test at the API seam using TestClient or AsyncClient** - API-level tests catch integration issues (auth headers, serialization, validation) that users actually encounter; avoid testing internal functions directly.

**MUST use polyfactory (not pydantic-factories) for test data generation** - The library was renamed and expanded; it generates valid Pydantic models respecting validators in one line: `user: User = UserFactory()`.

**ALWAYS run tests with pytest-cov and enforce minimum 80% coverage** - Use `pytest --cov=app --cov-report=term-missing` to identify untested code paths; anything below 80% indicates insufficient test rigor.

**NEVER allow real network calls, real time, or real randomness in tests** - Make tests hermetic by mocking external services, freezing time with freezegun, and seeding random generators to ensure reproducible test runs.

**MUST create separate test database or use in-memory SQLite for each test run** - Never point tests at production or shared development databases; use isolated test DBs that are created fresh and torn down after tests complete.

---

## 7. Error Handling & Validation

### Validation & Error Response Structure

**ALWAYS override RequestValidationError handler to standardize validation error responses** - Default Pydantic validation errors return nested structures with `loc`, `msg`, and `type` fields. Override with `@app.exception_handler(RequestValidationError)` to provide consistent, user-friendly error formats across your API.

**MUST include both user-friendly messages and structured error details in validation responses** - Return a dual-layer structure: simplified error messages for client consumption and detailed error arrays for debugging. Example: `{"message": "Validation failed", "errors": {...}, "detail": [...]}`.

**ALWAYS use PydanticCustomError for domain-specific validation errors with context** - Instead of generic `ValueError`, use `PydanticCustomError('error_type', 'message template with {variable}', {'variable': value})` to provide rich, contextual error messages with proper error typing.

### Exception Handling Architecture

**MUST create domain-specific custom exceptions inheriting from HTTPException** - Define exceptions like `InventoryException`, `PostNotFoundException` with pre-configured status codes and contextual detail messages. This provides type-safe, semantic error handling across your application.

**ALWAYS implement global exception handlers for both HTTP and unexpected exceptions** - Use `@app.exception_handler()` for centralized error handling. Handle `HTTPException`, `RequestValidationError`, and catch-all `Exception` to ensure no error goes unformatted.

**NEVER expose internal exception details or stack traces in production responses** - For unexpected exceptions (500 errors), return generic messages to clients while logging full details. Use environment-based configuration to show stack traces only in development.

### Validation Patterns

**MUST use field_validator with mode='after' for type-safe validations** - After validators run post-Pydantic parsing, ensuring type safety. Use `mode='before'` only when handling raw input transformation. Always return the validated value from validators.

**ALWAYS use model_validator for cross-field validation logic** - When validation depends on multiple fields, use `@model_validator(mode='after')` to access the full model instance. This prevents redundant checks and maintains validation order clarity.

**MUST use dependencies for database-dependent validations, not Pydantic validators** - Pydantic validators cannot access request context or perform async operations. Use FastAPI dependencies for validations requiring database checks, authentication verification, or external service calls.

### Logging & Monitoring

**ALWAYS log validation and HTTP errors with request context** - Include request method, path, headers (excluding sensitive data), and correlation IDs in error logs. Use different log levels: WARNING for client errors (4xx), ERROR for server errors (5xx).

**MUST use structured JSON logging with consistent fields for production environments** - Configure structured logging with fields like `timestamp`, `level`, `message`, `request_id`, `user_id`, `error_type`. This enables effective log aggregation and monitoring in centralized systems.

**ALWAYS capture error context before raising HTTPException in business logic** - Log the error with full context (parameters, state, user info) before raising the exception. The exception handler may not have access to this context, making debugging difficult without prior logging.

---

## 8. Production Deployment & Operations

### Worker & Process Management

**NEVER run Gunicorn with multiple workers inside Kubernetes containers** - Use single Uvicorn process per container and let Kubernetes handle horizontal scaling through pod replication. Gunicorn's multi-worker pattern conflicts with container orchestration.

**MUST calculate workers using `(2 × CPU_cores) + 1` for VM/bare-metal deployments** - This is the proven formula for Gunicorn with Uvicorn workers. Measure and tune for your specific workload; too many workers degrades performance.

**ALWAYS use `gunicorn.workers.UvicornWorker` class in production** - Run `gunicorn app:app --worker-class uvicorn.workers.UvicornWorker` for VM deployments. Never use standalone Uvicorn without a process manager outside containers.

### Health Checks & Graceful Shutdown

**MUST implement separate `/health/live` and `/health/ready` endpoints** - Liveness checks basic app health (returns 200 if running). Readiness checks all critical dependencies (database, cache, external APIs). Configure appropriate Kubernetes probe timeouts to prevent zombie containers.

**ALWAYS handle SIGTERM signals with lifespan events** - Use FastAPI's `@asynccontextmanager` lifespan pattern to cleanup resources (close DB connections, flush caches, finish background tasks) before shutdown. Configure `terminationGracePeriodSeconds` in Kubernetes to match your longest request timeout plus cleanup time.

**NEVER allow health checks to receive production traffic** - Health check endpoints must bypass authentication, rate limiting, and heavy middleware. Place them on separate paths excluded from observability metrics to avoid skewing latency percentiles.

### Environment & Configuration

**MUST use Pydantic BaseSettings for all configuration** - Load environment variables through typed Pydantic models with validation. Never hardcode secrets or use unvalidated `os.getenv()`. For production, integrate secrets managers (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault) rather than `.env` files.

**ALWAYS use multi-stage Docker builds with non-root users** - Build stage installs dependencies (use `python:3.12-slim` as base), runtime stage copies only virtual environment and application code. Create and run as non-root user (`USER appuser`) to reduce attack surface. Keep images under 200MB for faster cold starts.

### Timeout & Connection Management

**MUST configure `--timeout-graceful-shutdown` to exceed max request duration** - Set uvicorn's graceful shutdown timeout to your longest expected request time plus 5-10 seconds buffer. Default 5s causes request termination during rolling deployments.

**ALWAYS set `--timeout-keep-alive` based on load balancer settings** - Configure uvicorn keep-alive timeout to match or slightly exceed your load balancer's idle timeout (typically 60-75s for AWS ALB). Mismatched timeouts cause client connection errors.

### Observability

**MUST expose Prometheus metrics at `/metrics` with OpenTelemetry instrumentation** - Use `prometheus-fastapi-instrumentator` or OpenTelemetry SDK for traces, metrics, and logs. Avoid high-cardinality labels (no user IDs, full URLs). Set up alerts on p99 latency, error rate, and saturation metrics.

**NEVER log sensitive data or use blocking I/O in request handlers** - Configure structured JSON logging with correlation IDs. Use async logging handlers to prevent blocking the event loop. Scrub PII, tokens, and passwords from logs before they hit collectors. Set up centralized logging (Loki, CloudWatch) with appropriate retention.

---

## Summary

These 115+ rules represent distilled patterns from production FastAPI systems at scale, emphasizing:

- **Security First**: OWASP API Security Top 10 compliance, proper authentication/authorization
- **Performance**: Async patterns, connection pooling, caching strategies
- **Maintainability**: Clean architecture, separation of concerns, testability
- **Reliability**: Graceful shutdown, health checks, proper error handling
- **Observability**: Structured logging, metrics, distributed tracing

Following these rules ensures your FastAPI application is production-ready, secure, performant, and maintainable at enterprise scale.

---

*Generated from industry best practices as of mid-2025*
*Sources: FastAPI Official Docs, OWASP, SQLAlchemy 2.0 Docs, Production Deployments at Scale*
