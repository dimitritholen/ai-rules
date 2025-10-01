# Python 3.12+ Production Rules

## 1. Type System & Modern Syntax

### Type Parameter Syntax (PEP 695)
- **MUST** use PEP 695 syntax for generics in Python 3.12+: `class Container[T]:` not `TypeVar`
- **MUST** use `type` statement for type aliases: `type ListOrSet[T] = list[T] | set[T]`
- **MUST** use built-in generics: `list[int]`, `dict[str, int]` not `List[int]`, `Dict[str, int]`
- **MUST** import collection types from `collections.abc`, NOT `typing` (deprecated, removal Oct 2025)

### Type Annotations
- **MUST** use modern union syntax: `str | None` not `Optional[str]` (Python 3.10+)
- **MUST** use `TypeIs` for type narrowing over `TypeGuard` in Python 3.13+
- **MUST** use `Self` for methods returning instance: `def method(self) -> Self:` (Python 3.11+)
- **NEVER** use `runtime_checkable` unless genuinely needed at runtime (performance cost)
- **MUST** annotate public APIs; use strict type checking in production (mypy --strict)

### Type Checking Configuration
```toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true
```

---

## 2. Exception Handling

### Exception Groups (Python 3.11+)
- **MUST** use `TaskGroup` for concurrent operations where one failure should cancel others
- **MUST** use `except*` syntax for handling `ExceptionGroup`
- **NEVER** swallow `asyncio.CancelledError` - always re-raise after cleanup

### Exception Chaining
- **MUST** use `raise ... from e` when re-raising to preserve traceback chain
- **MUST** use `add_note()` to enrich exceptions with context without changing type (Python 3.11+)
- **NEVER** use bare `except:` - catches `KeyboardInterrupt`, `SystemExit`

### Try/Except Patterns
- **MUST** minimize try block scope - only wrap code that can raise expected exception
- **MUST** catch specific exception types, not broad `Exception`
- **MUST** log full tracebacks: use `log.exception()` or `exc_info=True`
- **MUST** inherit custom exceptions from `Exception`, never `BaseException`
- **MUST** organize exception hierarchies by how callers handle them, not by module

### Structured Logging
```python
import structlog
log = structlog.get_logger()
log.error("operation_failed", order_id=order_id, exc_info=True)
```

---

## 3. Async/Await & Concurrency

### Structured Concurrency
- **MUST** use `asyncio.TaskGroup` for all-or-nothing concurrent operations (Python 3.11+)
- **MUST** use `asyncio.timeout()` for external I/O operations
- **NEVER** block event loop with sync operations >100ms - use `run_in_executor()`
- **ALWAYS** await coroutines or use `create_task()` - never call async function without await

### Async Patterns
- **MUST** handle task exceptions explicitly - background tasks fail silently if unchecked
- **MUST** use `asyncio.gather(..., return_exceptions=True)` for independent operations
- **MUST** implement cleanup in `except asyncio.CancelledError` blocks, then re-raise
- **NEVER** use `time.sleep()` in async code - use `asyncio.sleep()`
- **NEVER** use `eval()` or `exec()` with untrusted input

### Concurrency Model Selection
- **I/O-bound tasks**: Use `asyncio` (best performance)
- **CPU-bound tasks**: Use `multiprocessing` (bypasses GIL)
- **Mixed/blocking libraries**: Use `ThreadPoolExecutor`
- **NEVER** use asyncio for CPU-bound work (slowest option)

---

## 4. Performance Optimization

### Language-Level (Python 3.12+)
- **PREFER** comprehensions over explicit loops (2x faster with PEP 709 inlining)
- **MUST** use `__slots__` for classes with 1000+ instances (2-3x memory reduction, 15% faster access)
- **MUST** choose data structures by access pattern: `set`/`dict` for lookups O(1), `deque` for queues O(1)
- **MUST** use generators for large datasets consumed once (O(n) → O(1) memory)

### Optimization Priority Order
1. **Algorithmic complexity** (O(n²) → O(n log n)) - 10-1000x gains
2. **I/O optimization** (batching, connection pooling, caching) - 2-10x gains
3. **Concurrency model** (multiprocessing for CPU-bound) - 2-8x gains
4. **Python version upgrade** (3.12/3.13) - 5-15% gains
5. **Memory management** (`__slots__`, generators) - 10-40% when applicable
6. **Micro-optimizations** - <5% gains, do last after profiling

### Profiling Strategy
- **MUST** profile before optimizing (use Scalene for development, py-spy for production)
- **NEVER** micro-optimize without profiling data
- **MUST** optimize algorithms before considering Cython/Rust extensions

---

## 5. Testing with pytest

### Test Architecture
- **MUST** follow test pyramid: 50% unit, 30% integration, 20% e2e
- **MUST** use src layout for production code
- **MUST** structure tests by type: `tests/unit/`, `tests/integration/`, `tests/e2e/`
- **MUST** mirror application structure in test directories

### Fixture Design
- **MUST** choose scope based on isolation vs performance (module scope: 40% faster for expensive setup)
- **MUST** use factory fixtures for multiple instances: `@pytest.fixture def user_factory():`
- **MUST** use yield for setup/teardown to ensure cleanup on failure
- **MUST** use `pytest.mark.asyncio` for async tests

### Quality Metrics
- **MUST** achieve >80% line coverage AND >80% mutation score (mutmut)
- **NEVER** trust coverage alone - use mutation testing to validate test quality
- **MUST** test edge cases and error paths, not just happy paths
- **MUST** use specific assertions with expected values, not weak checks (`is not None`)
- **ALWAYS** use `autospec=True` with mocks to enforce interface compliance

### Test Configuration
```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--cov=src --cov-report=term-missing --cov-fail-under=80 --strict-markers"
markers = [
    "slow: marks tests as slow",
    "integration: marks integration tests",
]
```

---

## 6. Project Structure & Packaging

### Project Layout (src layout)
```
project/
├── src/
│   └── package_name/
│       ├── __init__.py
│       └── module.py
├── tests/
├── pyproject.toml
├── README.md
└── LICENSE
```

### pyproject.toml (PEP 621)
```toml
[build-system]
requires = ["hatchling >= 1.26"]
build-backend = "hatchling.build"

[project]
name = "package-name"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "requests>=2.31.0,<3.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "ruff>=0.1.0",
    "mypy>=1.8.0",
]
```

### Dependency Management (uv)
- **MUST** use `uv` for virtual environments and dependency management
- **MUST** commit lockfiles (`uv.lock`) to version control
- **NEVER** commit `.venv/` to version control
- **MUST** use `uv sync --frozen` in CI/CD for reproducible builds
- **MUST** specify version constraints: `>=X.Y.Z,<NEXT_MAJOR` for libraries

### Build & Distribution
- **MUST** build both wheel and sdist: `uv build`
- **NEVER** manually create distribution files
- **MUST** use hatchling for pure Python, setuptools only for C/C++ extensions

---

## 7. Security

### Input Validation
- **MUST** use parameterized queries for SQL - never concatenate user input
- **MUST** implement allowlisting, not denylisting
- **MUST** validate server-side (client-side can be bypassed)
- **MUST** sanitize output with framework templating (Jinja2 autoescaping)

### Authentication & Secrets
- **MUST** use Argon2 for password hashing (not SHA-256/MD5/bcrypt)
- **MUST** use 256-bit random secrets for JWT with short expiration (15-30 min)
- **NEVER** hardcode secrets in source code
- **MUST** use secret managers in production (AWS Secrets Manager, Azure Key Vault)
- **MUST** implement MFA for privileged accounts

### Dependency Security
- **MUST** implement automated dependency scanning in CI/CD (Safety, pip-audit)
- **MUST** scan for malicious packages, not just vulnerabilities
- **MUST** pin exact dependency versions with `==` in production
- **MUST** update dependencies monthly and monitor security advisories

### Deserialization
- **NEVER** use `pickle.load()` with untrusted data (arbitrary code execution)
- **MUST** use JSON for untrusted data serialization
- **MUST** use `yaml.safe_load()`, never `yaml.unsafe_load()`
- **MUST** scan ML models for malicious pickle files (multiple tools, not just picklescan)

### Cryptography
- **MUST** use `cryptography` library, never implement custom crypto
- **MUST** use AES-256-GCM for symmetric encryption
- **MUST** use `secrets` module for random generation, not `random`
- **NEVER** reuse IVs/nonces for encryption

### API Security
- **MUST** configure CORS restrictively - whitelist origins, never `*` with credentials
- **MUST** implement CSRF protection for state-changing operations
- **MUST** implement adaptive rate limiting (token bucket algorithm)
- **MUST** set security headers: `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`

### Logging
- **NEVER** log passwords, tokens, private keys, PII, or sensitive data
- **MUST** use structured logging (JSON format) for production
- **MUST** implement log rotation (`RotatingFileHandler`)
- **MUST** centralize logs with real-time monitoring and alerting

---

## 8. Code Quality

### Linting with Ruff
```toml
[tool.ruff.lint]
select = [
    "E",     # pycodestyle errors
    "F",     # Pyflakes
    "B",     # flake8-bugbear
    "I",     # isort
    "UP",    # pyupgrade
    "S",     # Bandit security
    "C90",   # McCabe complexity
]

[tool.ruff.lint.mccabe]
max-complexity = 10  # Evidence-based threshold
```

### Formatting
- **MUST** use Ruff formatter (99.9% Black-compatible): `ruff format`
- **MUST** set line length to 88 characters
- **MUST** enforce consistent formatting in CI: `ruff format --check`

### Docstrings (PEP 257 + Google Style)
```python
def function(arg1: str, arg2: int) -> bool:
    """Summary line in imperative mood ending with period.

    Extended description providing context.

    Args:
        arg1: Description of arg1.
        arg2: Description of arg2.

    Returns:
        Description of return value.

    Raises:
        ValueError: When invalid input provided.
    """
```

### Complexity Management
- **MUST** enforce cyclomatic complexity <10 per function
- **MUST** run dead code detection: `ruff check --select F401,F841` and `vulture`
- **MUST** refactor functions with complexity >10 immediately

---

## 9. Container & Deployment

### Docker Best Practices
```dockerfile
FROM python:3.12-slim AS builder
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

FROM python:3.12-slim
RUN useradd -m -u 1000 appuser
COPY --from=builder /app/.venv /app/.venv
COPY src/ ./src/
USER appuser
ENV PATH="/app/.venv/bin:$PATH"
CMD ["python", "-m", "my_package"]
```

### Container Security
- **MUST** use distroless or slim base images (not Alpine with Python)
- **MUST** run containers as non-root user
- **MUST** pin base image versions (never `latest` tag)
- **MUST** scan images for vulnerabilities (Trivy, Snyk)
- **MUST** use multi-stage builds (dependencies separate from code)

### Python Version
- **MUST** use Python 3.12.11+ or 3.13+ (multiple CVE fixes)
- **MUST** monitor python.org security advisories and OSV database

---

## 10. CI/CD Quality Gates

### Pre-commit Hooks
```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.7.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.11.0
    hooks:
      - id: mypy
```

### CI Pipeline Requirements
- **MUST** run `ruff check` (zero errors to merge)
- **MUST** run `ruff format --check` (enforce formatting)
- **MUST** run `mypy` with strict mode (100% public API typed)
- **MUST** run `pytest --cov --cov-fail-under=80` (coverage threshold)
- **MUST** run security scanning: `ruff check --select S` and `safety check`
- **MUST** run `vulture` for dead code detection
- **MUST** fail builds on high/critical security vulnerabilities

### Quality Thresholds
1. Ruff checks: 0 errors
2. Type coverage: 100% of public APIs
3. Test coverage: ≥80% line coverage
4. Mutation score: >80% for critical modules
5. Cyclomatic complexity: ≤10 per function
6. Security vulnerabilities: 0 high/critical

---

## 11. XML & File Upload Security

### XML Processing
```python
from lxml import etree
parser = etree.XMLParser(resolve_entities=False, no_network=True)
tree = etree.parse(source, parser)
```
- **MUST** disable external entities to prevent XXE attacks
- **PREFER** `defusedxml` library for comprehensive protection

### File Uploads
- **MUST** whitelist allowed file extensions
- **MUST** verify magic bytes (file signature), not just extension
- **MUST** limit file size to prevent DoS
- **MUST** store uploads outside web root
- **MUST** run antivirus/sandbox scanning on uploads
- **MUST** sanitize SVG files (can contain XSS)

---

## 12. Critical Anti-Patterns

### Never Do This
❌ Use bare `except:` (catches `KeyboardInterrupt`, `SystemExit`)
❌ Use `pickle.load()` with untrusted data
❌ Hardcode secrets in source code
❌ Use string concatenation for SQL queries
❌ Block async event loop with sync operations
❌ Use `random` module for security tokens
❌ Run containers as root
❌ Use `latest` tag for base images
❌ Log passwords, tokens, or PII
❌ Trust client-side validation alone
❌ Implement custom cryptography
❌ Use MD5/SHA-1 for security purposes
❌ Use `eval()` or `exec()` with user input
❌ Ignore SAST/security scan failures
❌ Deploy without vulnerability scanning

---

## Production Deployment Checklist

### Pre-Deployment
- [ ] Python 3.12.11+ or 3.13+ with latest security patches
- [ ] All dependencies scanned and vulnerabilities resolved
- [ ] `pyproject.toml` with `[build-system]` and `[project]` tables
- [ ] Lockfile (`uv.lock`) committed to version control
- [ ] Type checking passes: `mypy --strict src/`
- [ ] Linting passes: `ruff check src/`
- [ ] Formatting enforced: `ruff format --check src/`
- [ ] Tests pass with ≥80% coverage
- [ ] Mutation score >80% for critical modules
- [ ] Security scanning passes (no high/critical vulnerabilities)
- [ ] Secrets in environment variables or secret manager
- [ ] Docker image scanned and runs as non-root
- [ ] CORS configured with explicit origin whitelist
- [ ] CSRF protection enabled
- [ ] Rate limiting configured
- [ ] Security headers set
- [ ] Structured logging with secret redaction
- [ ] Centralized logging with alerting configured

### Monitoring
- [ ] Application metrics (response times, error rates)
- [ ] Security monitoring (failed auth, anomalous behavior)
- [ ] Dependency vulnerability monitoring (automated alerts)
- [ ] Log analysis and alerting configured
- [ ] Performance profiling enabled (py-spy in production)

---

## References

### Official Standards
- PEP 8: Style Guide
- PEP 257: Docstring Conventions
- PEP 518: Build System
- PEP 621: Project Metadata
- PEP 695: Type Parameter Syntax
- PEP 654: Exception Groups
- PEP 709: Comprehension Inlining

### Official Documentation
- python.org: Language reference and security advisories
- packaging.python.org: Python Packaging Authority guides
- docs.astral.sh/uv: uv dependency manager
- docs.astral.sh/ruff: Ruff linter and formatter
- mypy.readthedocs.io: mypy type checker
- pytest.org: pytest testing framework

### Security Resources
- OWASP Top 10:2021
- OWASP Cheat Sheet Series
- python.org/downloads: Security releases
- osv.dev: Open Source Vulnerability database
- github.com/advisories: GitHub Advisory Database

### Best Practice Guides
- Real Python: Modern Python tutorials
- Google Python Style Guide
- Python Packaging User Guide (PyPA)

---

**Last Updated:** Mid-2025
**Minimum Python Version:** 3.12
**Target Audience:** Production systems, enterprise applications
**Philosophy:** High-impact rules that prevent bugs, security issues, and maintainability problems. Cargo cult practices explicitly rejected.