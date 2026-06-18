---
title: Ichiba Marketplace Listing & Search API - Architecture Specification
version: 1.0
date_created: 2026-06-13
last_updated: 2026-06-13
owner: Platform Engineering
tags: architecture, api, postgresql, go, full-text-search, pagination, marketplace, docker
---

# Introduction

This specification defines the architecture, data contracts, requirements, and validation criteria for Ichiba, a marketplace listing and search API. The system provides catalog management, full-text search with relevance ranking, and cursor-based paginated listing retrieval over a PostgreSQL data store, exposed as a JSON HTTP API.

## 1. Purpose & Scope

**Purpose:** Define all structural, behavioral, and operational requirements for the Ichiba API so that any conforming implementation produces consistent, predictable results and can be extended or replaced by AI-assisted development without ambiguity.

**Scope:** Covers the following system boundaries:
- PostgreSQL schema design, indexing strategy, and migration lifecycle
- Go domain model and repository layer implementing CRUD and search operations
- HTTP routing and handler contracts for all exposed endpoints
- Containerization and local orchestration via Docker and Docker Compose
- Unit testing strategy for handlers and repository logic

**Out of scope:** Authentication/authorization, rate limiting, observability/tracing, and production deployment infrastructure.

**Intended audience:** Backend engineers, AI code generation agents, and infrastructure engineers working on or extending this system.

**Assumptions:**
- The runtime environment is Linux (amd64 or arm64).
- PostgreSQL version is 15 or higher (required for `GENERATED ALWAYS AS ... STORED` tsvector columns).
- Go version is 1.21 or higher.
- The system operates in a trusted internal network context; TLS termination is handled upstream.

---

## 2. Definitions

| Term | Definition |
|---|---|
| **FTS** | Full-Text Search: PostgreSQL's built-in mechanism for lexeme-based document matching using `TSVECTOR` and `TSQUERY` types. |
| **GIN** | Generalized Inverted Index: A PostgreSQL index type optimised for composite values such as `TSVECTOR`, enabling fast full-text and array lookups. |
| **TSVECTOR** | A PostgreSQL data type representing a document as a sorted list of lexemes, used as the target for FTS queries. |
| **TSQUERY** | A PostgreSQL data type representing a parsed text search query, used to match against a `TSVECTOR`. |
| **`ts_rank`** | A PostgreSQL function that computes a relevance score (float4) for a document/query pair based on term density and position. |
| **`websearch_to_tsquery`** | A PostgreSQL function that converts a user-supplied search string into a `TSQUERY` using web-search-style syntax (supports `AND`, `OR`, `-`, phrase). |
| **Cursor-based pagination** | A pagination strategy where the next page is located using an opaque pointer (cursor) referencing the last seen record, rather than a numeric offset. |
| **Composite cursor** | A pagination cursor composed of two fields (`created_at`, `id`) to guarantee uniqueness and determinism across pages even when `created_at` values collide. |
| **pgxpool** | A connection pool provided by `jackc/pgx/v5/pgxpool`, used for concurrent, safe database access in Go. |
| **Domain model** | A Go struct (`domain.Listing`) that is the canonical in-memory representation of a listing record, independent of database or HTTP concerns. |
| **Repository** | A Go struct (`PostgresListingRepository`) that encapsulates all SQL queries and maps between database rows and domain models. |
| **Handler** | A Go struct (`ListingHandler`) that processes HTTP requests, validates inputs, calls the repository, and serialises responses. |
| **Migration** | A versioned, ordered SQL file pair (`.up.sql` / `.down.sql`) that applies or reverts a schema change. |
| **`sqlmock`** | The `DATA-DOG/go-sqlmock` library used to intercept `database/sql` calls in unit tests without a live database. |
| **SLA** | Service Level Agreement: a defined threshold for performance or availability. |
| **UUID** | Universally Unique Identifier (RFC 4122). Used as the primary key type for listings. |

---

## 3. Requirements, Constraints & Guidelines

### Database

- **REQ-DB-001**: The `listings` table must define the columns: `id` (UUID, primary key, default `gen_random_uuid()`), `title` (TEXT, NOT NULL), `description` (TEXT, nullable), `category` (TEXT, NOT NULL), `price` (NUMERIC(10,2), NOT NULL), `status` (TEXT, NOT NULL, default `'active'`), `created_at` (TIMESTAMPTZ, NOT NULL, default `now()`), and `search_vec` (TSVECTOR, generated).
- **REQ-DB-002**: The `search_vec` column must be a `GENERATED ALWAYS AS ... STORED` column computed by `to_tsvector('english', title || ' ' || COALESCE(description, ''))`. It must never be written to by application code.
- **REQ-DB-003**: A GIN index named `idx_listings_search` must be created on the `search_vec` column.
- **REQ-DB-004**: A B-tree composite index named `idx_listings_cursor` must be created on `(created_at DESC, id)` to support cursor-based pagination queries.
- **REQ-DB-005**: A partial B-tree composite index named `idx_listings_filters` must be created on `(category, price)` with the condition `WHERE status = 'active'`.
- **REQ-DB-006**: Each schema change must be delivered as a migration pair: `{version}_create_listings_table.up.sql` and `{version}_create_listings_table.down.sql`. The down migration must cleanly reverse all changes made by its paired up migration.
- **CON-DB-001**: PostgreSQL version must be >= 15. The `GENERATED ALWAYS AS ... STORED` syntax for `TSVECTOR` is required.
- **CON-DB-002**: The `price` column type `NUMERIC(10,2)` must not be changed to a floating-point type (`FLOAT`, `REAL`, `DOUBLE PRECISION`) to avoid rounding errors in financial values.

### Domain Model

- **REQ-DM-001**: The `domain.Listing` struct must include the fields: `ID` (string), `Title` (string), `Description` (string), `Category` (string), `Price` (float64), `Status` (string), `CreatedAt` (time.Time), and `Rank` (*float32, omitempty).
- **REQ-DM-002**: The `domain.FilterParams` struct must include: `Query` (string), `Category` (string), `MinPrice` (*float64), `MaxPrice` (*float64), `CursorTime` (*time.Time), `CursorID` (string), and `Limit` (int).
- **CON-DM-001**: The domain package must not import the `repository`, `handler`, or any HTTP/database packages. It must remain a pure value-object layer.

### Repository

- **REQ-REPO-001**: The repository must implement a `Create` method that inserts a new listing and scans `id` and `created_at` back into the provided `*domain.Listing`.
- **REQ-REPO-002**: The repository must implement a `GetByID` method that returns `nil, nil` (not an error) when no row is found (distinguished from `pgx.ErrNoRows`).
- **REQ-REPO-003**: The repository must implement a `List` method that dynamically constructs a parameterised SQL query based on the provided `FilterParams`. All filter clauses are additive (AND). The cursor clause must use the row-value comparison `(created_at, id) < ($n, $n+1)` to correctly paginate over the composite index.
- **REQ-REPO-004**: The repository must implement a `ListRanked` method that uses `ts_rank` for relevance scoring, orders results by rank descending then `created_at` descending, and returns `Rank` populated on each result.
- **REQ-REPO-005**: All FTS queries must use `websearch_to_tsquery('english', $n)` (not `plainto_tsquery` or `to_tsquery`) to support natural web-search syntax from user input.
- **CON-REPO-001**: The repository must use `pgxpool.Pool` for database connectivity. Raw `database/sql` connections must not be used.
- **CON-REPO-002**: All database rows must be closed via `defer rows.Close()` immediately after a successful `Query` call.
- **GUD-REPO-001**: Dynamic SQL must be built using `strings.Builder` with `fmt.Sprintf` for parameter placeholder injection. String concatenation with user values must never be used (SQL injection risk).

### HTTP Handlers

- **REQ-HTTP-001**: The `POST /listings` endpoint must decode the request body as a `domain.Listing` JSON object, default `Status` to `"active"` if empty, call `repo.Create`, and return HTTP 201 with the created listing as JSON.
- **REQ-HTTP-002**: The `GET /listings/{id}` endpoint must extract the `id` path parameter via `chi.URLParam`, call `repo.GetByID`, return HTTP 404 with body `"Listing not found\n"` when the listing does not exist, and HTTP 200 with the listing as JSON otherwise.
- **REQ-HTTP-003**: The `GET /listings` endpoint must accept the query parameters `q`, `category`, `min_price`, `max_price`, `limit`, `cursor_time` (RFC3339 format), and `cursor_id`. Default `Limit` is 10. It must return a JSON object with a `data` array and a `pagination` object containing `next_cursor_time` and `next_cursor_id`. The cursor fields must be populated only when `len(results) == filters.Limit`; otherwise they must be empty strings.
- **REQ-HTTP-004**: The `GET /listings/ranked` endpoint must require the `q` query parameter and return HTTP 400 with body `"Query parameter 'q' is required\n"` if absent. On success it must return a JSON array of listings with `rank` populated.
- **REQ-HTTP-005**: All successful responses must set the `Content-Type: application/json` response header before writing the body.
- **CON-HTTP-001**: The router must be implemented using `go-chi/chi/v5`. No other HTTP routing library is permitted.
- **CON-HTTP-002**: The `GET /listings/ranked` route must be registered before the `GET /listings/{id}` route in the router to prevent `"ranked"` from being captured as an `id` path parameter.
- **GUD-HTTP-001**: Handler errors from the repository layer must surface as HTTP 500 responses. Input validation failures must return HTTP 400. Resource-not-found cases must return HTTP 404. Do not return raw error messages from internal packages to clients in production; the current implementation may be extended with error sanitisation.

### Containerization

- **REQ-INF-001**: The application must be packaged as a multi-stage Docker image. The builder stage must use `golang:1.21-alpine`. The runtime stage must use `alpine:3.18`. The final image must contain only the compiled binary and expose port 8080.
- **REQ-INF-002**: The Docker Compose configuration must define two services: `app` and `db`. The `app` service must declare a `depends_on` condition of `service_healthy` on `db`.
- **REQ-INF-003**: The `db` service must use `postgres:15-alpine`, mount a named volume `postgres_data` to `/var/lib/postgresql/data`, and define a healthcheck using `pg_isready -U postgres -d ichiba` with interval 5s, timeout 5s, and 5 retries.
- **CON-INF-001**: The compiled binary must be built with `CGO_ENABLED=0` to produce a statically linked binary compatible with the `alpine:3.18` runtime image.

### Testing

- **REQ-TEST-001**: All HTTP handlers must be testable using `net/http/httptest.ResponseRecorder` without a live HTTP server or live database.
- **REQ-TEST-002**: Repository and handler tests must use table-driven test patterns (a slice of test case structs iterated with `t.Run`).
- **REQ-TEST-003**: Chi URL parameters must be injected manually in handler unit tests by creating a `chi.RouteContext`, adding params via `rctx.URLParams.Add`, and passing it through `context.WithValue(req.Context(), chi.RouteCtxKey, rctx)`.
- **GUD-TEST-001**: For production-grade handler tests, handler dependencies (repositories) should be abstracted behind Go interfaces so that mock implementations can be injected, rather than depending directly on `*PostgresListingRepository`.

---

## 4. Interfaces & Data Contracts

### Database Schema

```sql
CREATE TABLE listings (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title       TEXT NOT NULL,
    description TEXT,
    category    TEXT NOT NULL,
    price       NUMERIC(10,2) NOT NULL,
    status      TEXT NOT NULL DEFAULT 'active',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    search_vec  TSVECTOR GENERATED ALWAYS AS (
                    to_tsvector('english', title || ' ' || COALESCE(description, ''))
                ) STORED
);
```

### Domain Model Contracts

```go
// Listing is the canonical in-memory representation of a marketplace listing.
type Listing struct {
    ID          string    `json:"id"`
    Title       string    `json:"title"`
    Description string    `json:"description"`
    Category    string    `json:"category"`
    Price       float64   `json:"price"`
    Status      string    `json:"status"`
    CreatedAt   time.Time `json:"created_at"`
    Rank        *float32  `json:"rank,omitempty"` // non-nil only for ranked search results
}

// FilterParams is the complete set of optional filters passed to the List repository method.
type FilterParams struct {
    Query      string
    Category   string
    MinPrice   *float64
    MaxPrice   *float64
    CursorTime *time.Time
    CursorID   string
    Limit      int
}
```

### HTTP API Endpoints

#### `POST /listings`

| Field | Value |
|---|---|
| Request Content-Type | `application/json` |
| Request Body | `Listing` object (see domain model; `id` and `created_at` are ignored on input) |
| Success Response | `201 Created`, `application/json`, full `Listing` object with server-assigned `id` and `created_at` |
| Error: Invalid body | `400 Bad Request`, plain text |
| Error: Server fault | `500 Internal Server Error`, plain text |

#### `GET /listings/{id}`

| Field | Value |
|---|---|
| Path Parameter | `id` - UUID string |
| Success Response | `200 OK`, `application/json`, full `Listing` object |
| Error: Not found | `404 Not Found`, `"Listing not found\n"` |
| Error: Server fault | `500 Internal Server Error`, plain text |

#### `GET /listings`

| Query Parameter | Type | Required | Description |
|---|---|---|---|
| `q` | string | No | Web-search-style full-text query |
| `category` | string | No | Exact category match |
| `min_price` | float | No | Minimum price (inclusive) |
| `max_price` | float | No | Maximum price (inclusive) |
| `limit` | int | No | Page size (default: 10) |
| `cursor_time` | string (RFC3339) | No | Pagination cursor: `created_at` of last item from previous page |
| `cursor_id` | string (UUID) | No | Pagination cursor: `id` of last item from previous page; must be provided with `cursor_time` |

**Success Response (200 OK):**
```json
{
  "data": [ /* array of Listing objects */ ],
  "pagination": {
    "next_cursor_time": "2024-01-15T10:00:00Z",
    "next_cursor_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```
When there is no next page, both `next_cursor_time` and `next_cursor_id` are empty strings `""`.

#### `GET /listings/ranked`

| Query Parameter | Type | Required | Description |
|---|---|---|---|
| `q` | string | Yes | Full-text search terms for relevance ranking |

**Success Response (200 OK):** JSON array of `Listing` objects with `rank` field populated (float32, higher = more relevant).

**Error: Missing `q`:** `400 Bad Request`, `"Query parameter 'q' is required\n"`

---

## 5. Acceptance Criteria

- **AC-001**: Given a valid `POST /listings` payload with `title`, `category`, and `price`, when the request is processed, then the response status is `201`, the body contains the submitted fields, and `id` is a non-empty UUID.
- **AC-002**: Given a `GET /listings/{id}` request with a UUID that does not exist in the database, when processed, then the response status is `404` and the body is `"Listing not found\n"`.
- **AC-003**: Given a `GET /listings` request with `cursor_time` and `cursor_id` set to the `created_at` and `id` of the last item on page N, when processed, then the response contains items strictly older than that cursor position with no duplicates.
- **AC-004**: Given a `GET /listings` response where the number of returned items equals the requested `limit`, then `pagination.next_cursor_time` and `pagination.next_cursor_id` must both be non-empty strings.
- **AC-005**: Given a `GET /listings` response where the number of returned items is less than the requested `limit`, then `pagination.next_cursor_time` and `pagination.next_cursor_id` must both be empty strings.
- **AC-006**: Given a `GET /listings/ranked` request with `q=vintage camera`, when processed, then the response is a JSON array where each item contains a non-nil `rank` field and items are ordered by `rank` descending.
- **AC-007**: Given a `GET /listings/ranked` request with no `q` parameter, when processed, then the response status is `400`.
- **AC-008**: Given the Docker Compose stack is started with `docker compose up`, when the `db` healthcheck passes, then the `app` container starts successfully and responds to `GET /listings` on port 8080 with a `200` status.
- **AC-009**: Given a table-driven handler unit test for `GET /listings/{id}`, when the test runs with a nil-returning mock repository, then `httptest.ResponseRecorder.Code` equals `404`.
- **AC-010**: Given concurrent `POST /listings` and `GET /listings` requests, when processed simultaneously, then no request returns duplicate rows or skips rows due to index inconsistency (validated by the composite cursor correctness).

---

## 6. Test Automation Strategy

- **Test Levels**:
  - **Unit**: Handler logic tested with `net/http/httptest` and mock repositories. Repository SQL logic tested with `DATA-DOG/go-sqlmock`.
  - **Integration**: Requires a live PostgreSQL 15 instance (e.g., via `docker compose up db`). Tests exercise the full repository stack against real SQL.
  - **End-to-End**: Full stack started via `docker compose up`. HTTP calls made against `localhost:8080` to validate all endpoint contracts.

- **Frameworks**:
  - `testing` (standard library) for test runner and assertions.
  - `github.com/stretchr/testify/assert` for readable assertions.
  - `github.com/DATA-DOG/go-sqlmock` for unit-level database interaction mocking.
  - `net/http/httptest` for HTTP handler isolation.

- **Test Data Management**:
  - Unit tests must be fully self-contained with no shared state between test cases.
  - Integration tests must truncate the `listings` table in a `t.Cleanup` function after each test run.

- **Test Pattern Requirement**: All test functions must use the table-driven pattern: a `tests` slice of anonymous structs, iterated with `t.Run(tt.name, func(t *testing.T) {...})`.

- **Handler Interface Requirement (GUD-TEST-001)**: For robust unit testing, the `ListingHandler` must depend on a `ListingRepository` interface (not the concrete `*PostgresListingRepository`), enabling mock injection.

- **Coverage Requirements**: Minimum 80% statement coverage on `internal/handler` and `internal/repository` packages.

- **CI Integration**: All unit tests must pass via `go test ./...` with no live infrastructure dependency. Integration tests should be gated behind a build tag (e.g., `//go:build integration`).

---

## 7. Rationale & Context

### Cursor-based pagination over OFFSET

SQL `OFFSET n` requires the database to scan and discard the first `n` rows on every page request, giving O(N) query cost as pages increase. Additionally, if rows are inserted or deleted between page requests, `OFFSET`-based pagination produces duplicated or skipped items. The composite cursor `(created_at DESC, id DESC)` with the row-value predicate `(created_at, id) < ($n, $n+1)` uses the `idx_listings_cursor` index to seek directly to the correct position in O(log N) time and is stable under concurrent writes.

### Generated TSVECTOR column over application-side indexing

Maintaining the `search_vec` column as a `GENERATED ALWAYS AS ... STORED` expression delegates lexeme computation to PostgreSQL at write time. This eliminates the risk of the search index diverging from the source text (e.g., due to failed application updates), removes the need for any cron-based reindexing job, and ensures the GIN index always reflects the current `title` and `description` values.

### `websearch_to_tsquery` over `to_tsquery`

`to_tsquery` requires callers to supply properly formatted tsquery syntax (e.g., `cat & dog`). User-supplied strings frequently contain characters that cause parse errors. `websearch_to_tsquery` accepts natural web-search syntax (`cat dog`, `"exact phrase"`, `-excluded`) and is safe against malformed input, making it appropriate for direct use with user query strings.

### Multi-stage Docker build

The two-stage build separates the Go toolchain (large) from the runtime binary (small). The final `alpine:3.18` image contains only the statically linked binary, minimising attack surface and image size. `CGO_ENABLED=0` is required to produce a binary with no C runtime dependency, which would otherwise be absent in the minimal Alpine environment.

---

## 8. Dependencies & External Integrations

### External Systems

- **EXT-001**: PostgreSQL 15+ - Primary data store for all listing records, full-text search index, and cursor pagination. Accessed via TCP connection pool.

### Third-Party Services

None required for core operation.

### Infrastructure Dependencies

- **INF-001**: Docker Engine (20.10+) and Docker Compose (v2) - Required for local development orchestration and containerised deployment via the defined `docker-compose.yml`.
- **INF-002**: Network connectivity between the `app` and `db` Docker Compose services on the internal bridge network (`db:5432`).

### Data Dependencies

- **DAT-001**: PostgreSQL connection string - Must be provided to the application via the `DATABASE_URL` environment variable in the format `postgres://user:password@host:port/dbname?sslmode=disable`.

### Technology Platform Dependencies

- **PLT-001**: Go 1.21+ - Required for `GOARCH`-compatible builds with the standard `net/http`, `encoding/json`, and `testing` packages at the versions expected by the module.
- **PLT-002**: PostgreSQL 15+ - Required for the `GENERATED ALWAYS AS (expr) STORED` column syntax on `TSVECTOR` type.

### Compliance Dependencies

None defined at this specification version.

---

## 9. Examples & Edge Cases

### Pagination: First page request

```
GET /listings?category=electronics&limit=2
```

Response:
```json
{
  "data": [
    { "id": "uuid-A", "created_at": "2024-03-10T12:00:00Z", ... },
    { "id": "uuid-B", "created_at": "2024-03-09T08:00:00Z", ... }
  ],
  "pagination": {
    "next_cursor_time": "2024-03-09T08:00:00Z",
    "next_cursor_id": "uuid-B"
  }
}
```

### Pagination: Next page request using cursor

```
GET /listings?category=electronics&limit=2&cursor_time=2024-03-09T08:00:00Z&cursor_id=uuid-B
```

SQL predicate generated: `AND (created_at, id) < ('2024-03-09T08:00:00Z', 'uuid-B')`

### Edge Case: Two listings with identical `created_at`

Given two rows:
```
id: uuid-X, created_at: 2024-03-09T08:00:00Z
id: uuid-Y, created_at: 2024-03-09T08:00:00Z
```
The composite cursor distinguishes them by `id`. If the cursor is `(2024-03-09T08:00:00Z, uuid-X)`, the row with `uuid-Y` will appear on the next page because `(created_at, id) < (ts, uuid-X)` evaluates per lexicographic UUID order. Both rows will appear across the two pages without duplication or omission.

### Edge Case: Final page (fewer results than `limit`)

When the last page returns 1 item but `limit=10`, `len(listings) < filters.Limit`, so `next_cursor_time` and `next_cursor_id` are both `""`.

```json
{
  "data": [ { "id": "uuid-Z", ... } ],
  "pagination": {
    "next_cursor_time": "",
    "next_cursor_id": ""
  }
}
```

### Full-text search query examples

| User input | `websearch_to_tsquery` result (conceptual) |
|---|---|
| `vintage camera` | `'vintag' & 'camera'` |
| `"vintage camera"` | `'vintag' <-> 'camera'` (phrase) |
| `camera -digital` | `'camera' & !'digit'` |

### Repository dynamic query: all filters applied

```sql
SELECT id, title, description, category, price, status, created_at
FROM listings
WHERE 1=1
  AND category = $1
  AND price >= $2
  AND price <= $3
  AND search_vec @@ websearch_to_tsquery('english', $4)
  AND (created_at, id) < ($5, $6)
ORDER BY created_at DESC, id DESC
LIMIT $7
```

---

## 10. Validation Criteria

- **VAL-001**: Running `go build ./...` from the module root produces no errors and outputs the binary at the path specified in the Dockerfile (`cmd/api/main.go`).
- **VAL-002**: Running `go test ./...` (unit tests only, no `integration` build tag) completes with exit code 0.
- **VAL-003**: `docker compose build` completes successfully and the final image size is less than 50MB.
- **VAL-004**: `docker compose up` starts both services; within 30 seconds `curl -s http://localhost:8080/listings` returns a `200` response with a JSON body containing a `data` key.
- **VAL-005**: `POST /listings` with a valid payload returns `201` and the response body contains a non-empty `id` field that is a valid UUID (matches `/^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/`).
- **VAL-006**: `GET /listings/{id}` with a UUID that was never inserted returns `404`.
- **VAL-007**: `GET /listings/ranked` without a `q` parameter returns `400`.
- **VAL-008**: Executing `EXPLAIN ANALYZE` on a `List` query with FTS and cursor parameters confirms use of `idx_listings_search` (GIN) and/or `idx_listings_cursor` (B-tree) indexes rather than sequential scans, for a dataset of at least 10,000 rows.
- **VAL-009**: The `search_vec` column cannot be set via an `INSERT` or `UPDATE` statement; any attempt must be rejected by PostgreSQL with an error (`cannot insert into column "search_vec"`).
- **VAL-010**: Two sequential `GET /listings` requests using the cursor returned from the first page produce no duplicate `id` values across the combined result sets.

---

## 11. Related Specifications / Further Reading

- [PostgreSQL Full-Text Search Documentation](https://www.postgresql.org/docs/current/textsearch.html)
- [PostgreSQL GENERATED Columns](https://www.postgresql.org/docs/current/ddl-generated-columns.html)
- [pgx/v5 Documentation](https://pkg.go.dev/github.com/jackc/pgx/v5)
- [go-chi/chi Routing Documentation](https://pkg.go.dev/github.com/go-chi/chi/v5)
- [DATA-DOG/go-sqlmock Documentation](https://pkg.go.dev/github.com/DATA-DOG/go-sqlmock)
- [RFC 3339: Date and Time on the Internet (Timestamps)](https://datatracker.ietf.org/doc/html/rfc3339)
