# REST API Design Production Rules (2025)

## 1. Resource Naming & URI Design

### Resource Naming

**ALWAYS use nouns to represent resources, NEVER verbs** - Resources represent entities (nouns), HTTP methods represent actions (verbs). Good: `GET /orders`, `POST /users`. Bad: `GET /createOrder`, `POST /getUser`.

**ALWAYS use plural nouns for collection resources** - Collections contain multiple items and should use plural naming. Good: `/users`, `/orders`, `/products`. Bad: `/user`, `/order`, `/product`.

**ALWAYS use singular nouns for singleton resources only** - Applies only to resources that represent a single instance by nature. Example: `/user-management/users/{id}/profile` (profile is singular).

**ALWAYS use lowercase letters in URI paths** - RFC 3986: URIs are case-sensitive except scheme and host. Prevents case-related errors and improves consistency. Good: `/managed-devices`. Bad: `/managedDevices`, `/ManagedDevices`.

**ALWAYS use hyphens (-) to separate words, NEVER underscores** - Improves readability in URIs. Good: `/device-management/managed-devices`. Bad: `/device_management/managed_devices`.

**NEVER include file extensions in URIs** - Use `Content-Type` header to specify data format. Good: `/users/123`. Bad: `/users/123.json`.

**NEVER include CRUD function names in URIs** - HTTP methods define the operation. Good: `DELETE /users/123`. Bad: `POST /deleteUser/123`.

### URI Structure & Hierarchy

**ALWAYS use forward slashes (/) to indicate hierarchical relationships** - Structure reflects logical relationships between resources. Pattern: `/parent-resource/{parent-id}/child-resource/{child-id}`. Example: `/customers/5/orders/12`.

**NEVER use trailing forward slashes in URIs** - Adds no semantic value and causes confusion. Good: `/users/123`. Bad: `/users/123/`.

**ALWAYS limit URI hierarchy depth to maintain simplicity** - Avoid nesting beyond `collection/item/collection`. Complex relationships should use query parameters or links instead. Good: `/users/123/orders`. Questionable: `/users/123/orders/456/items/789/details`.

**ALWAYS keep URIs under 2000 characters total length** - RFC 3986 practical constraint, includes path and query components.

**NEVER mirror internal database structure in URIs** - URIs should represent business entities, not implementation details. Design from API consumer perspective, not database schema.

### Query Parameters

**ALWAYS use query parameters for filtering, sorting, and pagination** - Path for resource identification, query for non-hierarchical operations. Good: `/devices?region=USA&brand=XYZ&sort=date:asc`. Bad: `/devices/USA/XYZ/sorted-by-date`.

**ALWAYS make query parameters optional** - Provide sensible defaults when parameters are omitted. Example: Default `limit=25` for pagination.

**ALWAYS use camelCase or snake_case for query parameter names consistently** - Match field naming convention used in request/response bodies. Query parameter names MUST start with a letter.

**ALWAYS percent-encode query parameter values** - RFC 3986 requirement for special characters. Example: `?name=John%20Doe`.

**ALWAYS validate query parameters and set upper limits** - Prevent abuse and performance issues. Example: Maximum `limit=100` for pagination.

---

## 2. HTTP Methods & Semantics

### GET Method

**MUST be safe - NEVER modify server state** - GET requests must have no side effects.

**MUST be idempotent** - Repeated calls produce identical results.

**MUST be cacheable** - Enable caching for performance optimization.

**MUST NOT include request payload for resource retrieval** - Query parameters only.

**NEVER use GET for state-modifying operations** - Creates CSRF vulnerabilities.

**Response codes**: 200 (OK) for successful retrieval, 204 (No Content) for successful retrieval with no body, 404 (Not Found) when resource doesn't exist.

### POST Method

**MUST NOT be safe - modifies server state** - POST changes data on the server.

**MUST NOT be idempotent by default** - Multiple identical requests may create multiple resources.

**USE for creating new resources in a collection where server assigns URI** - Server determines the new resource identifier.

**MUST implement idempotency keys for critical operations** - Use `Idempotency-Key` header with UUID to prevent duplicate processing.

**STORE idempotency keys in cache (Redis) with TTL** - Few hours to 1 day retention.

**RETURN identical response for duplicate requests with same idempotency key** - Prevents duplicate resource creation.

**Response codes**: 201 (Created) when new resource created with Location header, 200 (OK) when processing completed without creating resource, 204 (No Content) when successful with no response body, 400 (Bad Request) for invalid client data, 409 (Conflict) when duplicate resource exists, 422 (Unprocessable Entity) for duplicate requests in progress.

**APPLY POST to collections, not individual resources** - POST creates within a collection context.

### PUT Method

**MUST NOT be safe - modifies server state** - PUT replaces resources.

**MUST be idempotent** - Multiple identical requests have same effect as single request.

**MUST completely replace entire resource representation** - Not for partial updates.

**MUST require complete resource representation in request body** - All fields must be provided.

**APPLY to individual resources, NEVER to collections** - PUT targets specific resources.

**Response codes**: 200 (OK) when resource updated and response body returned, 201 (Created) when new resource created, 204 (No Content) when successfully updated with no response body, 404 (Not Found) when resource doesn't exist and creation not allowed, 409 (Conflict) when request conflicts with current resource state.

**NEVER use PUT for partial updates - use PATCH instead** - PUT semantics require complete replacement.

### PATCH Method

**MUST NOT be safe - modifies server state** - PATCH modifies resources.

**NOT required to be idempotent** - Though CAN be implemented as idempotent.

**USE for partial updates - only fields to be changed** - Send only modified fields.

**MUST apply changes atomically - all or nothing** - No partial application of changes.

**APPLY to individual resources, NEVER to collections** - PATCH targets specific resources.

**USE JSON Merge Patch (RFC 7396) or JSON Patch (RFC 6902)** - Specify Content-Type correctly (application/merge-patch+json or application/json-patch+json).

**CONSIDER making PATCH idempotent using conditional requests** - Use If-Match/ETag headers.

**Response codes**: 200 (OK) when resource updated and response body returned, 204 (No Content) when successfully updated with no response body, 400 (Bad Request) for malformed patch document, 409 (Conflict) when valid patch cannot be applied to current state, 415 (Unsupported Media Type) when patch format not supported, 422 (Unprocessable Entity) when patch semantically invalid.

### DELETE Method

**MUST NOT be safe - modifies server state** - DELETE removes resources.

**MUST be idempotent** - Multiple identical deletes have same effect as single delete.

**First DELETE returns 204 or 200** - Successful deletion response.

**Subsequent DELETE of same resource returns 204 or 404 - both acceptable** - Maintains idempotency guarantee.

**Response codes**: 204 (No Content) when successfully deleted, 200 (OK) when deleted and returning representation in response body, 404 (Not Found) acceptable for already-deleted resource (maintains idempotency), 409 (Conflict) when deletion conflicts with current state.

**NEVER use DELETE with query parameters to delete multiple resources** - Use POST with filter criteria for bulk deletes instead.

**CONSIDER soft deletes for audit trails** - Mark as deleted rather than physical removal.

---

## 3. HTTP Status Codes & Error Handling

### Success Responses (2xx)

**MUST use 200 OK** when GET returns a resource successfully, PUT/PATCH updates an existing resource and returns the updated representation, or DELETE completes successfully and returns a response body with status information.

**MUST use 201 Created** when POST successfully creates a new resource or PUT creates a new resource (upsert scenario). MUST include Location header pointing to the newly created resource's URI.

**MUST use 202 Accepted** when request is valid but processing is asynchronous and not yet complete. MUST include Location header pointing to a status endpoint for polling. SHOULD include Retry-After header to reduce unnecessary polling requests.

**MUST use 204 No Content** when operation succeeds but intentionally returns no response body. Common for DELETE operations, PUT/PATCH with no content to return. MUST NOT include a message body with 204 responses.

### Client Error Responses (4xx)

**NEVER return 200 OK with an error in the response body** - Breaks HTTP semantics, confuses monitoring systems, and complicates error handling.

**MUST use 400 Bad Request** when request syntax is malformed (invalid JSON, missing required headers), request violates HTTP protocol rules, or generic client error when no more specific 4xx code applies.

**MUST use 401 Unauthorized** when authentication credentials are missing, invalid, or expired. MUST include WWW-Authenticate header indicating expected authentication scheme.

**MUST use 403 Forbidden** when client is authenticated but lacks permission to access the resource. Authorization failed despite valid authentication. NEVER use 403 for authentication failures - use 401.

**MUST use 404 Not Found** when requested resource does not exist. MAY use 404 instead of 403 to prevent information leakage about resource existence.

**MUST use 409 Conflict** when request conflicts with current resource state (concurrent modification detected), duplicate resource creation attempted, constraint violations or business rule conflicts occur, or resource version mismatch in optimistic locking scenarios.

**MUST use 422 Unprocessable Entity** when request syntax is valid but semantic validation fails, business logic validation errors occur, or data violates application constraints. NEVER use 422 for syntax errors - use 400.

**MUST use 429 Too Many Requests** when client exceeds rate limits. MUST include Retry-After header indicating when client can retry. SHOULD include rate limit information in response body.

### Server Error Responses (5xx)

**MUST distinguish between 4xx and 5xx** - 4xx indicates client can fix the problem, 5xx indicates server-side issues requiring administrative intervention.

**MUST use 500 Internal Server Error** when unhandled exception occurs in server code, database query fails unexpectedly, or generic server error with no more specific 5xx code.

**MUST use 502 Bad Gateway** when acting as gateway/proxy and receiving invalid response from upstream server or upstream server returns malformed response.

**MUST use 503 Service Unavailable** when server is temporarily unable to handle requests (overloaded, maintenance) or backend service is down or unreachable. SHOULD include Retry-After header indicating when service will be available.

**NEVER expose stack traces, internal paths, or sensitive system details** in 5xx error responses.

### Error Response Structure (RFC 9457)

**MUST use RFC 9457 Problem Details format** for error responses with Content-Type: `application/problem+json`.

**Standard fields** (all optional but recommended):
- `type`: URI identifying the problem type (defaults to "about:blank")
- `title`: Short, human-readable summary that SHOULD NOT change between occurrences
- `status`: HTTP status code (100-599) matching the response status
- `detail`: Human-readable explanation specific to this occurrence
- `instance`: URI identifying this specific problem occurrence

**MUST provide consistent error structure** across all endpoints - NEVER vary error format between different API operations.

**MUST include field-level validation errors** when returning 422, specifying field name that failed validation, error code for the specific validation failure, and human-readable message explaining the constraint.

**MUST use generic error messages for authentication failures** (401/403) to prevent revealing whether usernames/emails exist, account status information, or internal authentication mechanisms.

**NEVER include sensitive information** in error responses: stack traces, database queries, internal file paths, configuration details, or user data from other accounts.

---

## 4. API Versioning & Evolution

### URI Versioning

**MUST include major version in URI path** - Format: `/v1/`, `/v2/`. Example: `https://api.example.com/v1/resource`.

**MUST NOT include minor or patch versions in URI path** - Only major versions appear in URI.

**ALWAYS use integer major versions** - Format: v1, v2, v3. Not decimals like v1.0, v1.1.

**MUST place version as first segment after base URI** - Standard pattern for clarity.

### Semantic Versioning

**MUST increment MAJOR version for backward-incompatible changes** - Breaking changes require new major version.

**MUST increment MINOR version for backward-compatible new features** - Non-breaking additions.

**MUST increment PATCH version for backward-compatible bug fixes** - Internal fixes without API changes.

**NEVER expose minor/patch versions in REST API URIs** - Only track internally.

**MUST treat the API contract (request/response schema) as the versioned surface** - Focus on external contract, not internal implementation.

### Breaking Changes

**Breaking changes include**: Removing or renaming endpoints/resources/fields, changing field types, adding required parameters, stricter validation constraints, changing HTTP status codes for existing operations, modifying error response structures, changing authentication or authorization requirements.

**MUST release breaking changes only in new major versions** - Never introduce breaking changes to existing versions.

### Non-Breaking Changes

**Non-breaking changes include**: Adding new endpoints, adding optional parameters, adding new fields to responses, making required fields optional, loosening validation constraints, adding new error codes.

**MUST deploy non-breaking changes to all supported versions** - Keep versions consistent where possible.

**ALWAYS make new fields optional by default** - Prevents breaking existing clients.

### Backward Compatibility

**MUST maintain backward compatibility within the same major version** - Stability guarantee for major version lifecycle.

**ALWAYS add new fields rather than modify existing ones** - Extend, don't change.

**NEVER remove fields within the same major version** - Field removal is breaking change.

**MUST ignore unknown fields sent by clients** - Be permissive in what you accept.

**MUST document all fields as required or optional from initial release** - Clear contract from start.

### Deprecation Strategy

**MUST announce deprecation minimum 6-12 months before removal** - Adequate migration time.

**MUST use `Sunset` HTTP header to indicate deprecation date** - RFC 8594 standard.

**MUST communicate through multiple channels** - Documentation, email, blog posts, in-app notifications.

**MUST provide migration guides with code examples** - Clear path forward for consumers.

**MUST monitor API usage to identify affected consumers before sunset** - Proactive support.

**MUST support deprecated versions for minimum 24 months after replacement version release** - Long support window.

**ALWAYS continue security patches for deprecated versions until sunset** - Security is non-negotiable.

### Version Support Policy

**MUST support each major version for minimum 24 months** - Predictable lifecycle.

**MUST document version support lifecycle clearly** - Transparency for consumers.

**MUST define clear end-of-life dates when releasing new versions** - No surprises.

**ALWAYS give 3-6 months notice before end-of-life** - Final warning period.

### API Request Handling

**MUST require clients to specify API version explicitly** - No implicit versioning.

**NEVER default to "latest version" when version is omitted** - Return 400 Bad Request instead.

**MUST validate version parameter and return 404 for unsupported versions** - Clear error feedback.

---

## 5. Pagination, Filtering & Sorting

### Pagination - General Requirements

**MUST implement pagination from day one** for all collection endpoints - Adding pagination later is a breaking change.

**MUST use cursor-based pagination** for large datasets (>100k records), frequently changing data, or real-time feeds. Cursor pagination provides O(1) performance regardless of page depth, while offset pagination degrades to O(n) for deep pages.

**MUST provide pagination metadata** in responses via Link headers (RFC 8288) OR response body envelope. Link headers are preferred for cleaner separation of concerns.

**NEVER return unbounded collections** - Always enforce a maximum limit even if client doesn't specify one.

### Pagination - Cursor-Based (Recommended)

**MUST use opaque, URL-safe page tokens** that clients CANNOT parse or construct manually.

**MUST use `page_token`** (Google) or `starting_after`/`ending_before` (Stripe) or `cursor` as query parameter names.

**MUST return `next_page_token` in response** - Empty string indicates end of collection.

**MUST expire cursors after 3 days maximum** - Prevent stale state accumulation.

**MUST return HTTP 400** with clear error message for invalid/expired cursors - Include guidance on restarting pagination.

**MUST maintain stable, deterministic sort order** - Always include unique field (like `id`) as tiebreaker when sorting by non-unique fields to prevent pagination inconsistencies.

**MUST validate all pagination parameters match original request** when using page tokens - Different filters/sorts invalidate the cursor.

### Pagination - Offset-Based (Limited Use)

**ONLY use offset pagination** when random page access is required (e.g., UI with page numbers) AND dataset size is <100k records.

**MUST use `limit` and `offset`** as query parameter names.

**MUST set default limit to 50** and maximum limit to 1000. Alternative: 30 default, 100 max (GitHub); 50-100 default, 100-1000 max (consensus).

**MUST return HTTP 400** for negative offset values.

**MUST silently coerce excessive limits** to maximum allowed value OR return HTTP 400 with clear maximum limit in error message.

**MUST set maximum offset limit** (e.g., 100,000) to prevent performance degradation on large datasets.

**MUST include total count with caution** - `COUNT(*)` queries can be expensive. Consider estimated counts, caching, or separate async endpoint for large tables.

### Pagination - Link Headers (RFC 8288)

**MUST use Link header format** when implementing header-based pagination:
```
Link: <https://api.example.com/items?page=3>; rel="next",
      <https://api.example.com/items?page=1>; rel="prev",
      <https://api.example.com/items?page=1>; rel="first",
      <https://api.example.com/items?page=10>; rel="last"
```

**MUST include `rel="next"`** relation type for next page (required).

**SHOULD include `rel="prev"`, `rel="first"`, `rel="last"`** relation types when applicable.

**MUST omit `rel="next"`** when at end of collection (only way to signal completion).

### Filtering - Query Parameters

**MUST use direct field parameters** for simple equality filters: `?status=active&category=books`.

**SHOULD use `filter[field]` bracket notation** for complex APIs to distinguish filtering from pagination/sorting (JSON:API pattern).

**MUST support operator suffixes** for range queries: `?price_gte=10&price_lte=100` OR `?filter[price][gte]=10`.

**MUST validate all filter parameters** and return HTTP 400 for unrecognized fields with clear error message listing supported filters.

**MUST apply filters BEFORE pagination** - Paginated results are a "view" into filtered dataset.

**MUST document supported operators**: `eq` (equals), `ne` (not equals), `gt` (greater than), `gte` (greater or equal), `lt` (less than), `lte` (less or equal), `in` (in array), `nin` (not in array).

**NEVER expose internal database column names** - Use API-specific field names and map internally.

### Sorting - Query Parameters

**MUST use `sort` as query parameter name** - Industry standard.

**MUST use minus prefix for descending**: `?sort=-created_at,name` (JSON:API standard: plus for ascending, minus for descending).

**MUST support comma-separated multiple fields**: `?sort=-priority,created_at` (sorts by priority desc, then created_at asc).

**ALTERNATIVE: Use colon separator**: `?sort=priority:desc,created_at:asc` (OpenStack pattern).

**MUST apply stable secondary sort** on unique field (typically `id`) when primary sort field is non-unique - Prevents pagination inconsistencies.

**MUST validate sort field names** and return HTTP 400 for invalid fields with list of supported sort fields.

**MUST apply sorting BEFORE pagination** - Consistent with filtering.

**SHOULD default to created_at descending** or another sensible default when no sort specified.

### Performance & Error Handling

**MUST use database indexes** on all filterable and sortable fields to prevent table scans.

**MUST return HTTP 400** for malformed query parameters (invalid syntax, type errors).

**MUST return HTTP 422** for semantically invalid parameters (e.g., `limit=99999` when max is 1000) - Though HTTP 400 is also acceptable.

**MUST include descriptive error messages** with parameter name, invalid value, and allowed range/format.

**MUST implement rate limiting** to prevent pagination abuse - Consider per-client pagination request limits.

**NEVER perform COUNT(*) queries** on large tables without optimization - Use approximate counts, cached counts, or omit total from cursor-based pagination.

**MUST test pagination at scale** - Offset 100,000+ records to identify performance issues before production.

---

## 6. Security & Authentication

### HTTPS & Transport Security

**MUST enforce HTTPS-only for all API endpoints** - Never expose REST APIs over plain HTTP to protect credentials, tokens, and sensitive data in transit.

**MUST implement TLS 1.2 or higher** - TLS 1.0 and 1.1 are deprecated; TLS 1.3 is recommended for enhanced security.

### Authentication & Authorization

**MUST use OAuth 2.0 Authorization Code Flow with PKCE** - This is the current standard for authentication; never use Implicit Flow (deprecated) or Resource Owner Password Credentials Grant.

**MUST validate PKCE code challenges using S256 method** - Public clients must use PKCE; confidential clients should use PKCE as well.

**MUST implement access control checks at every API endpoint** - Validate user authorization for each function that accesses data.

**MUST implement object-level authorization** - Verify user permission for each object identifier before returning or modifying data.

**MUST implement property-level authorization** - Restrict access to sensitive object properties based on user permissions.

**MUST implement function-level authorization** - Separate and enforce administrative vs. regular user function access.

### Token Management

**MUST use short-lived access tokens** - Set JWT expiration to 5-15 minutes for high-security applications, maximum 1 hour for general applications.

**MUST implement refresh token rotation** - Issue new refresh tokens on each use; refresh tokens should be long-lived (7-30 days) and opaque.

**MUST implement token revocation mechanisms** - Use revocation lists or server-side validation for immediate token invalidation when needed.

**MUST validate all JWT claims** - Verify issuer (iss), audience (aud), expiration time (exp), and not-before time (nbf).

**MUST use separate signing secrets for different JWT purposes** - Each environment and purpose should have unique secrets.

**MUST audience-restrict tokens to specific resource servers** - Limit token privileges to minimum required scope.

### Input Validation & Injection Prevention

**MUST validate all input on the server side** - Never trust client-side validation; client-side checks can be bypassed.

**MUST use allowlists over denylists** - Define exactly what IS authorized; everything else is denied by default.

**MUST validate input length, range, format, and type** - Check all parameters against expected criteria.

**MUST use parameterized queries or prepared statements** - This prevents SQL injection by separating SQL logic from user input.

**MUST sanitize user inputs** - Remove or escape potentially harmful characters before processing.

**MUST validate content-type headers** - Ensure request/response body matches the declared content type to prevent code injection.

**MUST set appropriate request size limits** - Prevent resource exhaustion from oversized payloads.

### Rate Limiting & Resource Protection

**MUST implement rate limiting on all API endpoints** - Return HTTP 429 (Too Many Requests) when limits are exceeded.

**MUST use per-user/per-key rate limiting** - Track usage individually using authentication tokens or API keys.

**MUST implement resource-based limits** - Apply stricter limits to high-demand or expensive endpoints.

**MUST include rate limit information in response headers** - Use X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset.

**MUST implement protection against automated threats** - Use behavioral analysis and bot detection for critical business flows.

**MUST prevent unrestricted resource consumption** - Implement limits on CPU, memory, network bandwidth, and external service calls.

### CORS Configuration

**MUST configure CORS restrictively** - Never use Access-Control-Allow-Origin: * for APIs with sensitive data.

**MUST specify trusted origins explicitly** - Only allow known, trusted domains in CORS configuration.

**MUST follow principle of least privilege for CORS** - Only permit HTTP methods and headers strictly required.

**MUST NOT dynamically reflect origins without validation** - This creates exploitable vulnerabilities.

**ALWAYS validate CORS requests on the server** - CORS is a browser security feature; implement additional server-side protections.

### Error Handling & Information Disclosure

**MUST return generic error messages to clients** - Avoid exposing implementation details, stack traces, or technical information.

**MUST log detailed errors securely server-side** - Maintain audit logs for security events while sanitizing logged data.

**MUST use appropriate HTTP status codes** - Return correct status codes (400, 401, 403, 404, 429, 500, etc.).

**MUST NOT include sensitive data in URLs** - Query parameters are logged and cached; use request bodies for sensitive data.

### Security Headers

**MUST implement security headers** on all API responses:
- Cache-Control: no-store (for sensitive data)
- Content-Security-Policy
- Strict-Transport-Security (HSTS)
- X-Content-Type-Options: nosniff

### API Management

**MUST maintain comprehensive API inventory** - Track all endpoints, versions, and their exposure status.

**MUST deprecate and disable old API versions** - Remove outdated endpoints that may have security vulnerabilities.

**MUST protect management/administrative endpoints** - Use strong authentication and separate access controls.

**MUST require API keys for public endpoints** - Even public APIs need identification and rate limiting.

### Third-Party Integration

**MUST apply same security standards to third-party API data** - Validate and sanitize data from external APIs as if from direct user input.

**MUST validate URIs before fetching remote resources** - Implement strict allowlists to prevent SSRF attacks.

**MUST restrict external resource access** - Limit which external services APIs can communicate with.

### Configuration & Deployment

**MUST use secure defaults** - Disable unnecessary features, use secure cipher suites, and enforce authentication by default.

**MUST regularly audit API configurations** - Review security settings, exposed endpoints, and access controls.

**MUST implement automated security testing** - Integrate security validation into CI/CD pipelines.

**MUST implement CSRF protection** - Use state parameters and verify origin for state-changing operations.

**NEVER pass tokens in URLs** - URLs are logged and visible in browser history; use headers or request bodies.

**NEVER expose credentials in code or logs** - Use environment variables, secret managers, and sanitize logs.

**NEVER trust default configurations** - Complex systems often have insecure defaults that must be hardened.

---

## 7. Response Formats & Content Negotiation

### HTTP Response Structure

**MUST set `Content-Type: application/json` for JSON responses** - All JSON responses require explicit Content-Type declaration per RFC 9110.

**MUST use HTTP status codes as the primary success/error indicator** - Never duplicate status information in response body (e.g., `{"success": true}`). HTTP provides the envelope.

**MUST return `201 Created` with `Location` header for resource creation** - POST operations creating resources must include both 201 status and Location header pointing to the new resource URI.

**MUST include `Date` header in all 2xx, 3xx, and 4xx responses** - Origin servers with clocks must generate Date headers per RFC 9110.

### Content Negotiation

**MUST support content negotiation via `Accept` header** - Clients specify preferred representation formats through Accept header, not URL extensions.

**MUST respect quality factors (q parameter) in Accept headers** - Process q values (0-1 range) to determine client preference order when multiple formats are acceptable.

**NEVER hardcode format assumptions** - Servers must parse Accept header and return appropriate Content-Type, defaulting to application/json only when no Accept header is provided.

### Response Body Structure

**NEVER wrap responses in unnecessary envelopes** - Return resources directly at top level. HTTP headers carry metadata. Only envelope when JSONP support or header-incapable clients require it.

**MUST maintain consistent schema between requests and responses** - Whatever shape you accept in requests must match the shape returned in responses for the same resource.

**MUST use flat, shallow structures** - Minimize nesting depth. Avoid deeply nested objects that make navigation cumbersome.

**NEVER mix resource data with metadata in the same object** - Keep metadata (pagination, links) separate from resource attributes when enveloping is necessary.

### HATEOAS Implementation

**MUST include hypermedia links if claiming to be RESTful** - Hypermedia as the engine of application state is a REST constraint, not an option. You either implement HATEOAS or it's not REST.

**MUST use standard hypermedia formats when implementing HATEOAS** - Choose `application/hal+json` for simple APIs or `application/vnd.api+json` (JSON:API) for complex APIs with standardized CRUD operations.

**NEVER implement partial HATEOAS** - Either commit to hypermedia-driven interactions throughout the API or explicitly acknowledge you're building an HTTP API, not a RESTful API.

**MUST provide rel types with all links** - Links without relationship types (self, next, prev, first, last) are meaningless for client navigation.

**ALWAYS use HAL+JSON for minimal implementations** - When keeping it simple, HAL provides hypermedia benefits without excessive complexity.

**ALWAYS use JSON:API for complex domains** - When APIs require compound documents, efficient caching, standardized updates, and broad tooling support.

### Data Representation

**MUST use ISO 8601 format for dates/times** - All temporal values in UTC with explicit timezone: `2025-10-01T14:30:00Z`. Never use Unix timestamps or ambiguous formats.

**MUST use consistent field naming conventions** - Choose camelCase or snake_case and apply uniformly across the entire API surface. Never mix conventions.

**NEVER use strings for boolean values** - Use JSON's native `true`/`false`. Converting "true"/"false" strings adds unnecessary complexity.

**NEVER use strings for numeric values** - Use JSON number types unless precision requirements (financial calculations) mandate string representation for arbitrary precision.

### Performance & Caching

**MUST include cache control headers** - Set `Cache-Control`, `ETag`, and `Last-Modified` headers appropriately. Enable conditional requests with `If-None-Match`/`If-Modified-Since`.

**MUST support sparse fieldsets for large resources** - Allow clients to request specific fields via query parameters to reduce payload size and bandwidth.

**NEVER include sensitive data in cacheable responses** - Set `Cache-Control: private, no-store` for authenticated endpoints returning user-specific data.

---

## 8. Documentation & OpenAPI

### OpenAPI Specification

**MUST use OpenAPI 3.1 or later** as the standard specification format.

**MUST include at least one of**: `paths`, `components`, or `webhooks` fields in the OpenAPI document.

**MUST provide required fields**: `openapi` (version string), `info.title`, and `info.version`.

**RECOMMENDED to name the root OpenAPI document** `openapi.json` or `openapi.yaml`.

**MUST use YAML 1.2 or JSON format** for OpenAPI documents.

**MUST support CommonMark 0.27 markdown syntax** in description fields.

**MUST add OpenAPI descriptions to version control from project inception** - Track API contract changes.

**ALWAYS use design-first approach** - Create OpenAPI specification before implementation.

### Schema Design

**NEVER define schemas inline** - ALWAYS define in `components/schemas` section with unique names.

**MUST provide globally unique names** to avoid conflicts in code generation.

**MUST include non-empty, non-null examples** for all parameters, media types, and schemas.

**MUST ensure examples are valid** and conform to their schemas.

**MUST define examples that demonstrate intended payload** for automated test generation.

**MUST order parameters with required parameters before optional parameters** - Logical ordering for consumers.

### Documentation Completeness

**MUST provide descriptions** for info, operations, parameters, responses, and schemas.

**MUST include contact information** in the Info Object.

**MUST document at least one security scheme** in `securitySchemes`.

**MUST specify unique `operationId`** for each operation using code-friendly identifiers (letters, numbers, underscores, dashes).

**MUST assign at least one operation-level tag** for logical grouping.

**MUST define 2XX response with content** for all GET operations.

**MUST specify server information** in the servers list (not empty or null).

**MUST keep titles, names, and summaries under 50 characters** - Concise for readability.

### Interactive Documentation

**MUST provide interactive API documentation** using Swagger UI or equivalent.

**MUST allow users to test API calls directly** from documentation.

**MUST include "Try It Out" functionality** for all endpoints.

**MUST provide complete request/response examples** for every endpoint.

**MUST show example responses for all status codes** - Including error scenarios.

**MUST document authentication requirements** with working examples.

**MUST provide SDKs and client libraries** in multiple languages.

**MUST include "Getting Started" guide** for new users.

### Contract Testing & Validation

**MUST implement automated contract testing** from OpenAPI specification.

**MUST validate requests and responses** against OpenAPI schema.

**MUST use JSON Schema validators** (e.g., Ajv) to ensure compliance.

**MUST include contract validation in CI/CD pipeline** - Catch breaking changes early.

**MUST detect breaking changes before deployment** - Automated validation gates.

**MUST ensure API implementation stays synchronized with specification** - Single source of truth.

### Maintenance

**MUST maintain changelog** documenting all API changes.

**MUST document deprecation timeline** for old features.

**MUST provide migration guides** for version upgrades.

**MUST keep documentation in sync with implementation** - Documentation as code.

**MUST review and update documentation with every release** - Regular maintenance.

**MUST make OpenAPI specification available via API endpoint** for runtime discovery.

---

## 9. Idempotency

### Idempotency Keys

**MUST implement idempotency for all POST operations** that create or modify resources.

**MUST accept idempotency keys via `Idempotency-Key` header** - Standard header name.

**MUST use UUIDv4 or random strings** with at least 128 bits of entropy.

**MUST store first request result** (status + body) for any idempotency key.

**MUST return same result** for subsequent requests with same key.

**MUST expire idempotency keys after 24 hours minimum** - Prevent indefinite storage.

**MUST use atomic operations** (ACID transactions) when recording idempotency keys.

**MUST implement idempotency for payment, financial, and critical state-changing operations** - Essential for reliability.

---

## 10. Caching

### Cache Headers

**MUST make GET requests cacheable by default** - Enable browser and proxy caching.

**MUST include `Cache-Control` header** with directives:
- `max-age=<seconds>`: Cache duration
- `public` or `private`: Scope of caching
- `no-cache`: Require revalidation
- `immutable`: For static resources with cache-busting

**MUST provide ETag for resource versioning** - Enable conditional requests.

**MUST support conditional requests** with `If-None-Match` header.

**MUST return 304 Not Modified** when ETag matches.

**MUST include `Last-Modified` header** as fallback validator.

**NEVER cache sensitive or user-specific data** - Use `Cache-Control: private, no-store`.

**MUST encrypt cached sensitive data** - If caching is absolutely necessary.

### Cache Design

**MUST separate frequently changing data from stable data** - Different cache strategies.

**MUST avoid mixing public and private data** in same resource.

**MUST use separate endpoints** for generic vs. user-specific information.

---

## 11. Performance & Scalability

### Response Optimization

**MUST support partial responses** (field selection) for large resources - Reduce payload size.

**MUST implement compression** (gzip, brotli) for responses - Save bandwidth.

**MUST support asynchronous operations** for long-running tasks - Prevent timeouts.

**MUST return 202 Accepted with status endpoint** for async operations - Provide progress tracking.

**MUST implement connection timeouts and request timeouts** - Prevent hanging connections.

**MUST use CDN for static API documentation** - Fast global access.

---

These rules synthesize authoritative guidance from RFC specifications, OWASP, OAuth 2.0 Security Best Current Practice (RFC 9700), Microsoft Azure, Google Cloud, GitHub, Stripe, OpenAPI Initiative, and industry consensus as of 2025. They represent production-proven patterns for building secure, scalable, well-documented REST APIs.