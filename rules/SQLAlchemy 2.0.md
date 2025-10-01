# SQLAlchemy 2.0 Production Rules

## 1. Type System & Model Definition

### Type Annotations (CRITICAL: SQLAlchemy 2.0 Breaking Changes)
- **MUST** use `Mapped[]` annotations for all columns—every column must wrap its Python type in `Mapped[]` (e.g., `id: Mapped[int]`). SQLAlchemy 2.0 derives SQL types from the Python type inside `Mapped[]`.
- **MUST** use `mapped_column()` instead of `Column()`—modern declarative style requires `mapped_column()` for full type checking support. `Column()` is legacy and doesn't integrate with the type system.
- **MUST NOT** use the mypy plugin—the SQLAlchemy mypy plugin is deprecated and will be removed in 2.1. SQLAlchemy 2.0's native typing makes it unnecessary and causes conflicts (only works up to mypy 1.10.1).
- **MUST NOT** install sqlalchemy-stubs packages—all stub packages (sqlalchemy-stubs, sqlalchemy2-stubs) must be uninstalled. They conflict with SQLAlchemy 2.0's built-in typing and break type checking.

### Nullability & Optional Handling
- **MUST** use `Mapped[Optional[str]]` or `Mapped[str | None]` for nullable columns, and `Mapped[str]` for non-nullable columns. The presence of `Optional[]` controls database-level nullability automatically.
- **SHOULD** be aware that explicit `nullable` parameter overrides type hints—when you specify `nullable=False` on `mapped_column()`, it takes precedence over `Optional[]` in the annotation.

### Relationship Type Annotations
- **MUST** specify collection type in relationship annotations—use `Mapped[List["Child"]]` for one-to-many or `Mapped["Parent"]` for many-to-one. The ORM derives collection behavior (`uselist`) from the generic type (List, Set, etc.).
- **MUST** use `WriteOnlyMapped[]` for large collections—for relationships with thousands of items, use `WriteOnlyMapped["Model"]` instead of `Mapped[List["Model"]]`. Prevents accidental full collection loads and provides SQL-generating methods like `select()`, `insert()`, `update()`.
- **MUST** use `back_populates` instead of legacy `backref` for bidirectional relationships—provides explicit configuration on both sides, works better with PEP 484 typing, and makes attributes visible in source code for IDE support.
- **SHOULD** use forward references for circular dependencies—use string literals like `Mapped["OtherModel"]` and guard imports with `if TYPE_CHECKING:` to avoid circular imports.

### Table Configuration
- **MUST** use `__tablename__` attribute to explicitly specify table names—don't rely on automatic naming. Use `__table_args__` for schema, indexes, and constraints.
- **MUST** define naming conventions in metadata—explicit naming conventions for constraints, indexes, foreign keys ensure predictable, consistent database structure across environments.

---

## 2. Session & Transaction Management

### Session Lifecycle (CRITICAL: Most Common Production Failures)
- **MUST** use context managers for all session lifecycles—sessions should always be created and destroyed within a context manager (`with Session() as session:` or `with Session.begin() as session:`) to ensure proper cleanup and prevent resource leaks.
- **MUST** map session scope to business transaction boundaries, not function boundaries—a session represents a single database transaction. Create it at the start of a logical operation (web request, background task), close when operation completes.
- **MUST** create one session per request in web applications—never share sessions across requests or reuse them; each web request gets a fresh session that's closed when request ends.
- **MUST** ensure one Session per thread, one AsyncSession per asyncio task—sessions are mutable, stateful objects. Sharing across concurrent contexts causes resource leaks and data corruption.
- **NEVER** share Engine instances across process boundaries—when forking processes, either create a new Engine in the child process or call `Engine.dispose()` on any inherited engine before use.

### Transaction Management
- **MUST** explicitly handle commits and rollbacks in production code—rely on context managers or explicit try/except blocks. Pattern: yield session, commit on success, rollback on exception, always close in finally block.
- **SHOULD** use `Session.begin()` for explicit transaction blocks—this context manager commits automatically on success or rolls back on exceptions, eliminating manual commit/rollback boilerplate.
- **MUST** understand that commit/rollback affects outermost transaction only—in SQLAlchemy 2.0, these methods always operate on the outermost transaction; nested transactions require `begin_nested()` with separate handling.
- **NEVER** close session without handling uncommitted transactions—when closing a session with an active transaction, it automatically rolls back; always commit explicitly before close or use context managers.

### Session Configuration
- **SHOULD** set `expire_on_commit=False` in web applications—prevents unnecessary database queries to refresh objects after commit, particularly important for APIs returning committed data immediately.
- **NEVER** rely on thread-local scoped_session in new code—the scoped_session pattern ties sessions to threads arbitrarily; modern applications should use context managers and dependency injection instead.
- **MUST** keep session lifecycle separate from business logic—sessions should be created at function's top level, not inside data access functions; ensures predictable transactional scope.
- **SHOULD** pass sessions explicitly to awaitable functions—rather than using globals or thread-locals, explicitly pass the session object to functions that need database access.

---

## 3. Query Construction & Optimization

### Query API (SQLAlchemy 2.0 Breaking Changes)
- **MUST** use `select()` instead of legacy `Query()` API—Query API is deprecated in 2.0. Always construct queries with `select()` and execute using `session.execute()` or `session.scalars()`. The new approach unifies Core and ORM querying.
- **MUST** use `session.scalars()` when retrieving ORM entities—when selecting full ORM objects like `select(User)`, use `session.scalars()` to return entities directly rather than row tuples. Reserve `session.execute()` for complex multi-column results.
- **MUST** use `Session.execute()` for multi-column queries—when selecting specific columns (not whole objects), use `session.execute()` which returns `Result[Tuple[...]]`. Using `scalars()` would silently discard all columns except the first.
- **MUST** use `session.get(Model, id)` instead of deprecated `query.get(id)` for primary key lookups.

### Result Handling
- **MUST** call `.unique()` on results when using `joinedload()` with collections—when `joinedload()` is used for one-to-many or many-to-many relationships, it multiplies rows. Always apply `.unique()` to deduplicate: `session.scalars(stmt).unique()`.

### Subqueries & CTEs
- **MUST** use `scalar_subquery()` for subqueries in SELECT/WHERE clauses—when embedding a subquery that returns a single value, call `.scalar_subquery()` on the select statement. Use `.subquery()` for subqueries in FROM clauses and `.cte()` for common table expressions.
- **SHOULD** use CTEs via `.cte()` for complex queries with repeated subqueries—Common Table Expressions improve readability and can optimize query execution plans. Use `.cte()` instead of `.subquery()` when the same subquery is referenced multiple times.

### Query Optimization
- **SHOULD** select only needed columns for read-heavy operations—use `select(User.id, User.name)` instead of `select(User)` when you don't need the full ORM object. Reduces data transfer and object instantiation overhead.
- **MUST** use database-side COUNT instead of `len()`—fetching all rows for counting sends massive data over network; `func.count()` executes server-side.
- **NEVER** disable query caching without measuring impact—SQLAlchemy's compiled query cache significantly improves performance. Disabling it severely degrades ORM lazy loader and refresh query performance.

---

## 4. Relationships & Eager Loading

### N+1 Query Prevention (CRITICAL: Performance Impact)
- **SHOULD** use `selectinload()` for one-to-many and many-to-many relationships—most efficient strategy for collections. Emits a separate SELECT with an IN clause, avoiding row duplication and performing well with larger datasets.
- **SHOULD** use `joinedload()` for many-to-one relationships—for single-object references (many-to-one), `joinedload()` is the most general-purpose strategy, loading the related object in the same query via JOIN.
- **NEVER** rely on default lazy loading in production code—lazy loading causes N+1 query problems. Always specify explicit eager loading strategies (`selectinload`, `joinedload`) or use `raiseload()` to catch unexpected lazy loads during development.
- **MUST** use `raiseload()` during development to detect missing eager loads—set `lazy='raise'` or use `raiseload()` option to raise exceptions when relationships are accessed without eager loading, helping identify N+1 issues early.

### Relationship Configuration
- **SHOULD** use `cascade="all, delete-orphan"` for parent-owned one-to-many relationships—where children cannot exist without their parent. Use plain `cascade="all"` when children can be reassigned between parents.
- **MUST** understand cascade is uni-directional in bidirectional relationships—setting cascade on Parent.children does not automatically cascade from Child.parent; configure cascade independently on each side.
- **NEVER** use `cascade="all"` with async/await patterns—it aggressively expires related objects, causing issues with AsyncIO sessions. Use explicit cascade combinations instead.

### Hybrid Properties & Computed Attributes
- **SHOULD** use `column_property()` for SQL expressions that load eagerly with the parent row—correlated subqueries, aggregations. Use `@hybrid_property` when you need different logic for Python instance access vs SQL class-level queries.
- **MUST** define separate `@property.expression` decorators for hybrid properties when the SQL expression differs significantly from the Python implementation.

---

## 5. Performance & Scalability

### Connection Pooling
- **MUST** use QueuePool with default settings for web applications—the default pool_size=5 and max_overflow=10 provide reasonable defaults; only tune after profiling shows connection bottlenecks.
- **MUST** set `pool_pre_ping=True` for applications with long idle periods—prevents "MySQL server has gone away" errors and ensures connections are valid before use.
- **MUST** configure `pool_recycle` for databases that close idle connections—especially MySQL. Set to value less than database's connection timeout.

### Bulk Operations
- **MUST** use `bulk_insert_mappings()` for inserting >100 records—benchmarks show 0.32s vs 2.39s for ORM (7.5x faster for 100k records). Use Core's `session.execute(insert(...), list_of_dicts)` for even faster performance (0.21s).
- **NEVER** request primary keys back from bulk operations unless required—dramatically degrades performance; if needed, use Core's RETURNING with insertmanyvalues support in SQLAlchemy 2.0.
- **MUST** use bulk operations for inserting/updating multiple records—SQLAlchemy 2.0 provides optimized bulk insert with `session.execute(insert(...), list_of_dicts)` and RETURNING support. Never insert records one-by-one in loops.

### Indexing
- **MUST** add indexes for columns in WHERE, JOIN, and ORDER BY clauses—poorly indexed databases degrade performance by up to 80%; indexes change ilike queries from O(n) sequential scans to O(log n) bitmap heap scans.
- **SHOULD** use EXPLAIN to analyze slow queries—identifies missing indexes and inefficient query plans; if database takes long to return results, restructure query or add indexes.

### Pagination
- **NEVER** use LIMIT/OFFSET for large offsets—performance degrades severely (26.7s for offset=200k); use keyset/seek pagination instead, which scales to 100M+ rows with proper indexes.

### Caching
- **SHOULD** implement result caching with dogpile.cache for read-heavy workloads—cache frequently accessed query results; SQLAlchemy provides ORMCache extension with FromCache query option for seamless integration.
- **MUST** enable SQL compilation caching—set `cache_ok=True` for custom types and `inherit_cache=True` for custom constructs; allows skipping expensive string compilation for structurally equivalent queries.

---

## 6. Migrations & Schema Management (Alembic)

### Migration Generation & Review
- **MUST** always review and modify auto-generated migrations—Alembic's `--autogenerate` is not perfect and cannot detect all changes (especially table/column renames, complex type changes). Manual review is critical before applying to production.
- **MUST** keep migrations atomic and focused—one logical change per migration revision. Makes troubleshooting easier, rollbacks safer, and migration history clearer.
- **NEVER** edit previously committed migration files—always create a new migration to fix or adjust something. Editing existing migrations breaks version consistency across environments.

### Data vs Schema Migrations
- **SHOULD** define static table schemas within data migrations—for data manipulation migrations, define table structures directly in the migration file using SQLAlchemy Core constructs. Prevents breakage if model definitions change later.
- **MUST** use `op.bulk_insert()` or `op.execute()` for simple data changes—for complex data transformations requiring relationships, use full ORM models and sessions, but document the dependency on model state.

### Production Deployment & Safety
- **MUST** test migrations in staging before production—use `alembic upgrade --sql head` to preview SQL changes. Test both upgrade and downgrade paths in non-production environments.
- **MUST** back up production databases before downgrade operations—downgrade migrations often drop objects (tables, columns) causing permanent data loss. Downgrades should be rare in production and always require backups first.
- **SHOULD** integrate migrations into application startup or CI/CD—run `alembic upgrade head` automatically during deployment to ensure database schema stays synchronized with application code.
- **NEVER** run downgrade commands in production without explicit necessity—design forward-compatible migrations instead. If rollback is required, test thoroughly in staging first and ensure data preservation strategies are in place.
- **MUST** implement backward-compatible migrations for zero-downtime deployments—break destructive changes into multiple migrations: add new column → migrate data → deploy code using new column → remove old column in next release.

### Testing & Multi-Tenancy
- **SHOULD** use pytest-alembic for automated migration testing—test that migrations upgrade/downgrade cleanly (`up_down_consistency`), maintain single head revision (`single_head_revision`), and that model definitions match DDL (`model_definitions_match_ddl`).
- **MUST** use separate `alembic_version` tables per tenant/database—when managing multi-tenant schema-level isolation (PostgreSQL schemas, MySQL databases), each tenant requires independent version tracking. Use `alembic init multidb` template for multiple database configurations.

---

## 7. Testing Strategies

### Test Database Setup & Teardown
- **MUST** use session-scoped fixtures for engine and metadata—create the database connection and tables once at the start of the test suite and destroy them at the end to avoid repeated expensive operations.
- **MUST** use function-scoped fixtures for sessions with transaction rollback—each test function should receive a fresh session that operates within a transaction, with all changes rolled back after the test completes for isolation.
- **MUST** use `join_transaction_mode="create_savepoint"` for test sessions—allows the Session to use SAVEPOINT-based transactions exclusively, letting tests call `session.commit()` freely while keeping the outer transaction uncommitted for final rollback.
- **NEVER** commit the outer transaction in test fixtures—keep the externally initiated transaction non-committed and active under all circumstances, ensuring complete rollback of all test changes.

### Testing Philosophy & Coverage
- **SHOULD** avoid mocking SQLAlchemy queries in favor of real database testing—the effort to correctly mock SQLAlchemy statements often doesn't provide value over testing against an actual database; functional tests with real databases are more reliable and maintainable.
- **MUST** write tests for every line of behavioral code—SQLAlchemy's own philosophy is that untested code is a bug; tests for features should be five to ten times the lines of code as the actual behavior.
- **SHOULD** test against CRUD operations and constraints for each model—integration tests must verify Create, Read, Update, and Delete operations, plus database constraints and custom validations for every model.

### Factory Patterns
- **SHOULD** use Factory Boy's `SQLAlchemyModelFactory` with scoped sessions—configure a project-wide scoped_session, bind each factory to it via `Meta.sqlalchemy_session`, and call `Session.remove()` in test teardown.
- **MUST** bind factory sessions to the same session fixture used by tests—ensure the SQLAlchemy session in your factory is the same transactional session used by the test, guaranteeing everything is rolled back together.

---

## 8. Async Patterns

### Async Session Management (CRITICAL: Different from Sync)
- **MUST NOT** share AsyncSession across concurrent tasks—a single AsyncSession instance is not safe for concurrent asyncio tasks; each task requires its own session instance to prevent data corruption.
- **SHOULD** set `expire_on_commit=False` for async sessions—prevents "IO attempted in unexpected place" errors by avoiding lazy attribute loading after commits in async contexts.
- **MUST** use eager loading strategies in async code—apply `selectinload()` or similar options to relationships; lazy loading doesn't work reliably in async patterns and causes runtime errors.
- **MUST NOT** access expired attributes in async contexts without refresh—all objects expire after commit by default; either set `expire_on_commit=False` or explicitly refresh objects before accessing their attributes.

### Async Testing
- **MUST** use `AsyncSession` with connection-based transactions for async tests—create an async connection with `engine.connect()`, begin a transaction with `connection.begin()`, then bind the AsyncSession to that connection with the create_savepoint join mode.
- **MUST** wrap async test fixtures with proper context managers—use `async with` for both the connection and transaction to ensure proper resource cleanup and transaction lifecycle management.
- **MUST** call `engine.dispose()` in async fixtures after rollback—when testing async code, dispose of the engine after transaction rollback to prevent RuntimeError about tasks attached to different event loops.

---

## 9. Repository & Architecture Patterns

### Repository Pattern
- **SHOULD** separate domain models from ORM models in complex applications—use the repository pattern with data mapper functions (`model_to_entity` and `entity_to_model`) to maintain clean boundaries between domain logic and persistence infrastructure.
- **MUST** keep repository methods free of commit/rollback logic—repositories should add, retrieve, and manipulate entities but never commit. Transaction control belongs to the service/application layer that coordinates multiple repository calls.
- **SHOULD NOT** implement custom Unit of Work when Session suffices—SQLAlchemy's Session already implements the Unit of Work pattern. Only add a custom UoW abstraction if you need to swap persistence implementations or enforce domain-specific transaction boundaries.

---

## Implementation Priority

**Priority 1 (Breaking Changes & Critical Production Failures):**
- Use Mapped[] and mapped_column() (2.0 requirement)
- Do NOT use mypy plugin (deprecated)
- Use context managers for sessions (resource leaks)
- Use select() instead of query() (query deprecated)
- Use selectinload() to prevent N+1 (performance)
- Set pool_pre_ping=True (prevents connection errors)
- Review autogenerated migrations (data loss prevention)

**Priority 2 (Architecture & Patterns):**
- Use back_populates not backref (type safety)
- Map session scope to business transactions
- Do NOT share sessions across threads/async tasks
- Set expire_on_commit=False for web apps
- Use repository pattern for complex apps

**Priority 3 (Performance & Optimization):**
- Use bulk operations for >100 records (7.5x faster)
- Add indexes for WHERE/JOIN/ORDER BY
- Never use LIMIT/OFFSET for large offsets
- Use database-side COUNT not len()

**Priority 4 (Testing & Safety):**
- Use transaction rollback in tests
- Test migrations in staging
- Back up before downgrade

---

**Version:** SQLAlchemy 2.0 (Mid-2025)
**Sources:** Official SQLAlchemy 2.0 documentation, Alembic documentation, Python typing best practices, production troubleshooting reports, performance benchmarks.

These rules address production-critical concerns: SQLAlchemy 2.0 breaking changes (type system, query API), session lifecycle management (most common production bugs), N+1 query prevention (performance impact), connection pooling (resource management), Alembic safety (data loss prevention), and async patterns (different from sync). Following these rules ensures code quality meets enterprise standards for SQLAlchemy 2.0 applications.
