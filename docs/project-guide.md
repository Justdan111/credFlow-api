# CredFlow Backend — Build Guide & Production Roadmap

A complete study document: what we built across Phases 1–5, *how* we built it, the
*thinking* behind each decision, the *Go techniques* used, and an honest list of what
still stands between this codebase and production.

Read it top to bottom. It assumes you've seen the code but not that you remember why
any particular line is there.

> Companion docs:
> - [code-walkthrough.md](code-walkthrough.md) — per-file commentary
> - [endpoints.md](endpoints.md) — the full endpoint catalog (the product target)

---

## Table of contents

1. [What CredFlow is](#1-what-credflow-is)
2. [The big picture — architecture](#2-the-big-picture--architecture)
3. [The layered architecture explained](#3-the-layered-architecture-explained)
4. [The journey of a request](#4-the-journey-of-a-request)
5. [Phase-by-phase: what, how, and why](#5-phase-by-phase-what-how-and-why)
6. [How we used Go — techniques reference](#6-how-we-used-go--techniques-reference)
7. [Cross-cutting design principles](#7-cross-cutting-design-principles)
8. [What's done — checklist](#8-whats-done--checklist)
9. [What's remaining for production](#9-whats-remaining-for-production)
10. [Suggested order for the road ahead](#10-suggested-order-for-the-road-ahead)

---

## 1. What CredFlow is

CredFlow is a REST API for businesses to track money owed to them — customer debts,
payments, and cash flow. It is **multi-tenant**: many businesses use the same API and
database, and each one sees only its own data.

**Tech stack:**

| Concern | Choice |
|---|---|
| Language | Go 1.25 |
| HTTP router | `github.com/go-chi/chi/v5` |
| Database | PostgreSQL 16 (in Docker locally) |
| DB driver | `github.com/jackc/pgx/v5` (native pool, `pgxpool`) |
| Migrations | `github.com/golang-migrate/migrate/v4` |
| Auth | JWT (`github.com/golang-jwt/jwt/v5`) + bcrypt (`golang.org/x/crypto`) |
| Config | `.env` via `github.com/joho/godotenv` |

**Every endpoint returns the same JSON envelope:**

```json
{ "data": ..., "meta": null | {"page":1,"pageSize":20,"total":0}, "error": null | {"message":"..."} }
```

---

## 2. The big picture — architecture

```
                        ┌─────────────────────────────────────────┐
   HTTP request ───────▶│  cmd/server/main.go                     │
                        │  • loads config   • runs migrations     │
                        │  • builds the DB pool                   │
                        │  • wires every module   • starts server │
                        └────────────────────┬────────────────────┘
                                             │
                          ┌──────────────────▼──────────────────┐
                          │  chi router + middleware stack       │
                          │  RequestID▸RealIP▸Logger▸Recoverer▸  │
                          │  Timeout▸RequireAuth (on protected)  │
                          └──────────────────┬──────────────────┘
                                             │
        ┌────────────────────────────────────┼────────────────────────────────┐
        │                                    │                                │
  ┌─────▼──────┐                      ┌──────▼──────┐                  ┌──────▼──────┐
  │  Handler   │  parse request,      │  Handler    │                  │  Handler    │
  │            │  map errors→status   │             │                  │             │
  └─────┬──────┘                      └──────┬──────┘                  └──────┬──────┘
        │                                    │                                │
  ┌─────▼──────┐                      ┌──────▼──────┐                  ┌──────▼──────┐
  │  Service   │  validation,         │  Service    │                  │  Service    │
  │            │  business rules      │             │                  │             │
  └─────┬──────┘                      └──────┬──────┘                  └──────┬──────┘
        │                                    │                                │
  ┌─────▼──────┐                      ┌──────▼──────┐                  ┌──────▼──────┐
  │ Repository │  SQL only            │ Repository  │                  │ Repository  │
  └─────┬──────┘                      └──────┬──────┘                  └──────┬──────┘
        └────────────────────────────────────┼────────────────────────────────┘
                                             │
                                    ┌────────▼────────┐
                                    │  PostgreSQL     │
                                    └─────────────────┘
       auth module                customers module                debts module
```

**Folder structure and what each part means:**

```
cmd/server/main.go        The binary. Wiring only — no business logic.
internal/                 Private application code. Cannot be imported by other projects.
  auth/                   Registration, login, JWT, the auth middleware's context helpers.
  customers/              Customer CRUD.
  debts/                  Debt CRUD + lifecycle.
  middleware/             The RequireAuth HTTP middleware.
pkg/                      Reusable code that *could* be imported by other projects.
  database/               Connection pool + migration runner.
  response/               The JSON envelope helpers.
migrations/               Versioned SQL schema changes (0001, 0002, 0003...).
docs/                     This file and its companions.
```

Why `internal/` vs `pkg/`? Go enforces `internal/` — packages under it can only be
imported by code rooted at the same parent. It's a *compiler-enforced* "this is private."
`pkg/` is a convention for "this is safe for anyone to reuse." `database` and `response`
have no CredFlow-specific business logic, so they live in `pkg/`.

---

## 3. The layered architecture explained

Every feature module (auth, customers, debts) has the same four files. Each layer has
**one job** and depends only on the layer below it.

| Layer | File | Job | Does NOT do |
|---|---|---|---|
| **Handler** | `handler.go` | Parse the HTTP request, call the service, map the result/error to an HTTP status + envelope. | Validation, business rules, SQL. |
| **Service** | `service.go` | Validation, business rules, orchestration (e.g. "register = create business + user in a transaction"). | HTTP details, raw SQL. |
| **Repository** | `repository.go` | Translate Go method calls into SQL. Nothing else. | Validation, HTTP, business decisions. |
| **Models** | `models.go` | The data shapes: domain types, request types, response types. | Behaviour. |

**Why bother with layers?**

1. **Each layer is testable in isolation.** You can test the service's validation
   without a database or an HTTP server. You can test the handler's request parsing
   with a fake service.
2. **Change is contained.** Switching from JWT to session cookies touches only the
   middleware and handlers — the service and repository never know.
3. **It reads predictably.** Once you've read the auth module, the customers and debts
   modules hold no surprises. Same shape, every time.

The rule that keeps it honest: **dependencies point one direction only.** Handler imports
service. Service imports repository. Repository imports the database. Never the reverse.

---

## 4. The journey of a request

Trace `POST /api/customers` with a valid token and body:

1. **Kernel → Go's `http.Server`.** The server we built in `main.go` (with explicit
   read/write timeouts) accepts the TCP connection.
2. **Middleware stack**, outer to inner:
   - `RequestID` tags the request with a unique ID.
   - `RealIP` resolves the true client IP behind any proxy.
   - `Logger` starts timing.
   - `Recoverer` arms a `defer` that will turn any panic into a 500.
   - `Timeout` attaches a 60-second deadline to the request context.
   - `RequireAuth` reads the `Authorization: Bearer ...` header, verifies the JWT,
     and injects `userID` + `businessID` + `role` into the request context.
3. **Chi routing.** Chi matches the path to `customerHandler.Create`.
4. **Handler.** `Create` pulls `businessID` from the context, JSON-decodes the body
   into a `CreateRequest`, and calls `service.Create(ctx, businessID, req)`.
5. **Service.** `Create` validates (name required, email format, risk level, credit
   limit ≥ 0), then calls `repository.Create`.
6. **Repository.** Runs the parameterized `INSERT ... RETURNING`, scans the new row
   into a `Customer` struct, maps a unique-violation error to `ErrEmailTaken`.
7. **Back up the stack.** The `Customer` returns handler → which calls
   `response.Success(w, 201, customer)` → which writes the JSON envelope.
8. **Middleware unwinds.** `Logger` prints the line with status + duration.
9. The response reaches the client.

If anything fails, the error travels back *up* the same path, and the handler's
`writeServiceError` switch maps it to the right status code.

---

## 5. Phase-by-phase: what, how, and why

### Phase 1 — Project setup

**What:** A running HTTP server with a `/health` endpoint and the JSON envelope helpers.

**How:**
- Built an **explicit `http.Server`** instead of `http.ListenAndServe`. The convenience
  function has no timeouts, which leaves the server open to *slowloris* attacks (a
  client that dribbles bytes forever, holding a connection hostage).
- Added a **chi middleware stack** in a deliberate order: `RequestID` before `Logger`
  (so request IDs appear in logs), `Recoverer` before the handlers (so panics become
  500s instead of crashing the process).
- Implemented **graceful shutdown**: the server runs in a goroutine; `main` blocks on
  an OS-signal channel; on `SIGINT`/`SIGTERM` it calls `srv.Shutdown(ctx)` with a
  10-second budget so in-flight requests finish.
- Wrote `pkg/response` — `Success`, `SuccessWithMeta`, `Fail` — so no handler ever
  hand-rolls JSON.

**Thinking:** Phase 1 sets the *defaults* every later phase inherits. A senior gets the
unsexy production-safety pieces (timeouts, graceful shutdown, panic recovery) in on day
one, because retrofitting them later means touching every entry point.

### Phase 2 — Database layer

**What:** A PostgreSQL connection pool and a migration system. First two tables:
`businesses` and `users`.

**How:**
- `pkg/database/database.go` builds a `pgxpool.Pool`, configures sizing, and **pings
  eagerly** at startup. `pgxpool` is lazy — it doesn't actually connect until first use
  — so without the ping, a bad `DATABASE_URL` would only blow up on the first user
  request. Ping = fail fast.
- `pkg/database/migrate.go` runs `golang-migrate`. Migrations apply automatically on
  startup, *before* the pool opens, so no handler can query a table that doesn't exist
  yet.
- Migration `0001` created `businesses`, `users`, and a reusable `set_updated_at()`
  trigger function.
- Introduced the **`App` struct** to hold shared dependencies (`DB`, later `JWT`, etc.)
  — no global variables.

**Thinking:** The big choices were *configuration injection* (the database package
takes a `Config` struct; it never reads env vars itself, so it stays testable) and
*migrate-before-connect ordering* (schema must exist before traffic).

### Phase 3 — Authentication

**What:** `register`, `login`, `me`, and the `RequireAuth` middleware.

**How:**
- **bcrypt cost 12** for password hashing. bcrypt salts automatically and is
  deliberately slow — slow enough to make brute force expensive.
- **JWT HS256** access tokens carrying `userID`, `businessID`, `role`, and `exp`.
  The token verifier explicitly **rejects any non-HMAC signing method** — defending
  against the classic `alg=none` JWT forgery attack.
- **`register` runs in a transaction** (`pgx.BeginFunc`): it creates the business, then
  the user. If user creation fails, the business insert rolls back — no orphan rows.
- **User-enumeration safety:** login returns the identical error
  ("invalid email or password") whether the email is unknown or the password is wrong,
  so an attacker can't probe which emails are registered.
- The middleware injects identity into the request **context** using a private typed
  key, so no other package can collide with it.

**Thinking:** Auth is where security correctness matters most. Every decision —
the alg check, the equal-time login error, the transaction — closes a specific,
named attack or bug class.

### Phase 4 — Customers

**What:** Full customer CRUD — list (with search, filter, sort, pagination), create,
get, update, soft-delete.

**How:**
- Migration `0002` added `customers` with a `deleted_at` column for **soft delete**
  and **partial indexes** (indexes that cover only `WHERE deleted_at IS NULL` rows).
- **Tenant scoping**: every repository method takes `businessID` as its first argument,
  and every query filters `WHERE business_id = $1`. There is no "get by ID alone."
- **Sort whitelist**: user-supplied sort fields are looked up in a fixed
  `map[string]string` (API name → SQL column). Anything not in the map is rejected.
  This is the *only* safe way to do dynamic `ORDER BY` — column names can't be
  parameterized.
- **PATCH semantics**: `UpdateRequest` uses pointer fields (`*string`), so a missing
  field (`nil`) is distinguishable from a field explicitly set to empty.

**Thinking:** Phase 4 establishes the multi-tenant data pattern that debts (and every
future resource) copies verbatim. The verification suite explicitly tried cross-tenant
access and confirmed 404s.

### Phase 5 — Debts

**What:** Debt CRUD, a `mark-paid` action, and the nested `GET /api/customers/{id}/debts`.

**How:**
- Migration `0003` added `debts` with a `status` lifecycle (`pending`/`partial`/`paid`),
  a foreign key to `customers`, and `CHECK` constraints (`amount > 0`,
  `due_date >= issued_date`).
- **`overdue` is derived, not stored.** It's computed in every SQL `SELECT`
  (`due_date < CURRENT_DATE AND status <> 'paid'`). A stored column would need a nightly
  cron job to stay accurate.
- **`INSERT ... SELECT ... WHERE EXISTS`**: creating a debt checks the customer exists
  in the same tenant *inside the insert statement*. One atomic query — no
  check-then-insert race.
- **Calendar dates vs instants**: `issued_date`/`due_date` are `DATE` (a calendar day);
  `paid_at`/`created_at` are `TIMESTAMPTZ` (an exact instant). Mixing them up causes
  timezone bugs.

**Thinking:** Debts is the first time we model *money* and a *state machine*. The
guiding principle was "derive what depends on time, store what doesn't."

---

## 6. How we used Go — techniques reference

This section is a Go learning reference, each item grounded in code you've written.

### 6.1 Package organization

Go programs are trees of packages. `cmd/` holds binaries, `internal/` holds private
code (compiler-enforced privacy), `pkg/` holds reusable code. A package's *exported*
identifiers start with an uppercase letter (`Success`); lowercase ones
(`writeJSON`) are private to the package.

### 6.2 Structs and JSON tags

Structs are the data shapes. Field tags control JSON serialization:

```go
type User struct {
    ID           string `json:"id"`
    PasswordHash string `json:"-"`            // never serialized
    Industry     *string `json:"industry,omitempty"` // dropped from JSON when nil
}
```

`json:"-"` is a compile-time guarantee a field can't leak. `omitempty` drops zero-value
fields. We use **pointer fields** (`*string`) for nullable columns and PATCH requests.

### 6.3 Interfaces — small and implicit

Go interfaces are satisfied *implicitly* — a type just needs the right methods. Our
`DBTX` interface is satisfied by both `*pgxpool.Pool` and `pgx.Tx`:

```go
type DBTX interface {
    QueryRow(ctx context.Context, sql string, args ...any) pgx.Row
    Exec(ctx context.Context, sql string, args ...any) (pgconn.CommandTag, error)
}
```

This lets one set of repository methods run inside *or* outside a transaction —
no duplicated code.

### 6.4 Error handling

Go has no exceptions. Errors are values, returned explicitly and checked.

- **Sentinel errors** — package-level error values that mean something specific:
  ```go
  var ErrNotFound = errors.New("customer not found")
  ```
- **Wrapping** with `%w` preserves the chain:
  ```go
  return fmt.Errorf("create business: %w", err)
  ```
- **Matching** with `errors.Is` (for sentinels) and `errors.As` (for typed errors):
  ```go
  if errors.Is(err, pgx.ErrNoRows) { return Customer{}, ErrNotFound }

  var pgErr *pgconn.PgError
  if errors.As(err, &pgErr) && pgErr.Code == "23505" { ... } // unique violation
  ```

The handler's `writeServiceError` switch is where sentinel errors become HTTP statuses.

### 6.5 Pointers for tri-state (PATCH)

A plain `string` field can't tell "the client omitted this" from "the client sent
an empty string." A `*string` can: `nil` = omitted, `&""` = explicitly empty,
`&"x"` = a value. This is what makes PATCH (partial update) correct.

### 6.6 Concurrency — goroutines and channels

We used exactly one concurrency pattern, in graceful shutdown:

```go
serverErr := make(chan error, 1)
go func() {                       // server runs in its own goroutine
    if err := srv.ListenAndServe(); ... { serverErr <- err }
}()

stop := make(chan os.Signal, 1)
signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)

select {                          // block until EITHER channel fires
case err := <-serverErr: ...
case sig := <-stop: ...
}
```

`go func()` starts a goroutine (a lightweight thread). Channels carry values between
goroutines safely. `select` waits on multiple channels at once.

### 6.7 context.Context

`context.Context` carries request-scoped data and cancellation. We use it three ways:
1. **Cancellation/timeouts**: `context.WithTimeout` bounds a database ping.
2. **Request-scoped values**: the auth middleware stores `userID` in the context;
   handlers read it back.
3. **Propagation**: handlers pass `r.Context()` down through service → repository, so
   a client disconnect cancels the database query too.

### 6.8 Dependency injection via constructors

No globals. Each module has a constructor that takes its dependencies:

```go
repo := customers.NewRepository(pool)
svc  := customers.NewService(repo)
h    := customers.NewHandler(svc)
```

`main` wires the graph once at startup. Everything else receives what it needs as a
parameter. This is what makes the code testable — in a test you pass a fake.

### 6.9 Methods and the receiver

A method is a function with a receiver. `func (h *Handler) Create(...)` — `Create`
belongs to `*Handler`. Pointer receivers (`*Handler`) avoid copying the struct and
allow mutation. Our handlers are methods so they can reach their injected service.

### 6.10 Other idioms used

| Idiom | Where | What it does |
|---|---|---|
| Blank import `import _ "..."` | `migrate.go` | Runs a package's `init()` for its side effects (driver registration) without using its symbols. |
| `defer` | everywhere | Schedules cleanup (`rows.Close()`, `cancel()`) to run on function exit. |
| `map[string]struct{}` | sort/risk whitelists | A set. `struct{}` is zero bytes. |
| Closures | dynamic `UPDATE` builder | The `add(col, val)` closure captures and grows the `sets`/`args` slices. |
| `time.Parse("2006-01-02", s)` | debt date parsing | Go date layouts are a *reference time*, not strftime codes. |
| The `must` prefix | `mustEnv` | Convention for "fatal on error" helpers. |

---

## 7. Cross-cutting design principles

These ideas show up in every module — they're the spine of the codebase.

### Multi-tenancy and tenant scoping

Every business shares one database. Isolation is enforced in **every query**:
`WHERE business_id = $1`. The `businessID` comes from the verified JWT (never from the
request body or URL), and it's always the first argument to a repository method. A
lookup "by ID alone" does not exist in the code. This is the single most important
security property of the system.

### Soft delete

`customers` and `debts` have a `deleted_at TIMESTAMPTZ` column. "Deleting" sets the
timestamp; the row stays. Every read filters `deleted_at IS NULL`. This preserves
financial history and gives a free audit trail. Partial indexes keep the active-row
queries fast.

### Validation layering

Validation happens **twice**, on purpose:
- The **service** validates with friendly messages ("amount must be greater than 0").
- The **database** validates with `CHECK` constraints as the last line of defense.

The service is the *user-facing* guard; the DB constraint catches anything that
bypasses the service (bugs, future code paths, manual SQL).

### Derived vs stored state

If a value depends on *now* (`overdue`, `amountRemaining`), derive it at read time.
Storing it would require a background job to keep it fresh and risks drift. If a value
is fixed once set (`amount`, `issued_date`), store it.

### Security practices already in place

- Passwords: bcrypt cost 12, never stored or returned in plaintext.
- JWT: HS256 with a 64-byte secret; the verifier rejects `alg=none`.
- SQL injection: every value is a bound parameter; the only interpolated thing
  (sort column) comes from a fixed whitelist.
- User enumeration: login gives one error for both failure modes.
- Secrets: `.env` is git-ignored; `.env.example` documents the keys.
- Panics: `Recoverer` middleware converts them to 500s.
- Server: explicit read/write timeouts against slowloris.

---

## 8. What's done — checklist

**Infrastructure**
- [x] HTTP server with middleware, timeouts, graceful shutdown
- [x] PostgreSQL connection pool (pgx)
- [x] Versioned migrations (golang-migrate), 3 applied
- [x] Consistent JSON envelope
- [x] Config via `.env`
- [x] Liveness (`/health`) and readiness (`/health/db`) probes

**Features (endpoints live: 22)**
- [x] `POST /api/auth/register`, `POST /api/auth/login`, `GET /api/auth/me`
- [x] JWT auth middleware
- [x] Customers — list, create, get, update, soft-delete
- [x] Debts — list, create, get, update, soft-delete, mark-paid; `amount_paid`/`amount_remaining` derived from payments
- [x] Payments — list, create (with idempotency keys), get, void; atomic debt-status transitions
- [x] Nested: `GET /api/customers/{id}/debts`, `GET /api/customers/{id}/payments`, `POST /api/debts/{id}/payments`

**Quality**
- [x] Layered architecture, consistent across modules
- [x] Multi-tenant isolation, verified by test runs
- [x] Manual end-to-end verification each phase (curl suites)
- [x] **Automated test suite**: 95+ assertions across unit, HTTP-handler, and integration tiers
- [x] **CI pipeline** (`.github/workflows/test.yml`) running build, vet, unit, and integration tests
- [x] **Makefile** with `test`, `test-integration`, `test-all`, `db-up`, `db-reset` targets

---

## 9. What's remaining for production

This is the honest gap. The catalog in [endpoints.md](endpoints.md) lists ~80 endpoints;
we have 16. But missing *endpoints* is the smaller problem — the missing *engineering*
below matters more.

### 9.1 Testing — foundation laid ✓ (was the biggest gap)

A baseline test suite is now in place (~95 passing assertions). It covers:
- **Unit tests** for `pkg/response`, `internal/auth` (password, JWT including the
  `alg=none` regression, validation), `internal/customers` and `internal/debts`
  (sort whitelist, pagination clamping, validation).
- **HTTP-handler tests** for `internal/middleware` via `httptest`.
- **Integration tests** under `test/integration/` (build tag `integration`) that
  exercise the full HTTP→service→repo→Postgres stack — including the headline
  `TestTenantIsolation` that asserts no cross-tenant access path returns success.
- CI via `.github/workflows/test.yml` running build, vet, unit, and integration
  tests against a Postgres service container on every push.

What still belongs in this category as the codebase grows:
- Tests for each new endpoint as it's added.
- Repository-level tests that exercise SQL edge cases directly.
- A linter beyond `go vet` (`golangci-lint`).
- Coverage reporting in CI with a minimum threshold.

### 9.2 Remaining features

- ~~Payments~~ ✓ Done in Phase 6 — atomic status transitions, idempotency keys, void.
- ~~Idempotency keys~~ ✓ Done — `(business_id, idempotency_key)` partial unique index;
  replay returns 200 with the same row, no double-write.
- **Refresh tokens + logout + token revocation** — currently a token is valid until it
  expires; there's no way to revoke one.
- **Dashboard & analytics** aggregation endpoints.
- **Notifications, reminders, communications.**
- **Forgot/reset password** (needs email), **2FA** (needs a TOTP library).
- **Business profile & onboarding** endpoints.
- **Customer notes & activity timelines.**
- **Search, exports, file uploads, audit log, saved reports.**

### 9.3 Security hardening

- **RBAC enforcement** — users have a `role` (`owner`/`admin`/`member`) but no endpoint
  checks it yet. Destructive actions should require elevated roles.
- **Rate limiting** — especially on `login` and `register` (brute-force defense).
- **CORS** — needed before a browser frontend can call the API.
- **Security headers** — `X-Content-Type-Options`, `X-Frame-Options`, HSTS, etc.
- **Request body size limits** — `http.MaxBytesReader` to stop oversized-payload DoS.
- **TLS** — production must be HTTPS-only (usually terminated at a load balancer).
- **Audit logging** — a tamper-evident record of destructive and financial actions.
- **Secrets management** — `.env` is fine for dev; production needs a vault / secret
  manager, and `JWT_SECRET` rotation.

### 9.4 Observability

- **Structured logging** — replace the standard `log` package with `log/slog` (JSON
  logs, levels, request-ID correlation).
- **Metrics** — Prometheus-style counters/histograms (request rate, latency, error rate).
- **Tracing** — OpenTelemetry spans across handler → service → repository → DB.
- **Error monitoring** — a Sentry-style service for aggregated, alertable errors.

### 9.5 Infrastructure & deployment

- **Dockerfile** — a multi-stage build producing a small static binary image.
- **CI/CD pipeline** — build, test, lint, then deploy.
- **Migrations as a deploy step** — running them on app startup is fine for one
  instance; with multiple replicas, run them as a separate, ordered job.
- **Environment config** — proper staging/production separation.
- **Health checks wired** to the orchestrator (Kubernetes liveness/readiness).
- **Graceful rollout** — the graceful shutdown we built pays off here.

### 9.6 Data & performance

- **Database statement timeouts** — cap how long any single query may run.
- **Cursor-based pagination** — offset pagination degrades on deep pages and large
  tables; large lists should move to keyset/cursor pagination.
- **N+1 query awareness** — as nested data grows, watch for per-row query loops.
- **Backups & point-in-time recovery** — non-negotiable for financial data.
- **Connection-pool tuning** — size `MaxConns` against real load and Postgres limits.
- **Full-text / trigram search** — current search is `ILIKE '%q%'`; fine to ~100k rows,
  then it needs a proper index strategy.
- **Data retention** — a policy for genuinely purging soft-deleted rows.

### 9.7 API maturity

- **OpenAPI / Swagger spec** — machine-readable API docs for consumers.
- **API versioning strategy** — a plan for breaking changes (`/api/v2/...`).
- **Consistent error codes** — the envelope has an optional `error.code`; adopt a
  documented catalog of codes, not just human messages.
- **Pagination/sorting consistency** — apply the customers/debts list conventions
  uniformly to every future list endpoint.

---

## 10. Suggested order for the road ahead

A pragmatic sequence — each step makes the next safer:

1. ~~Add a test suite~~ ✓ Done — unit + HTTP + integration tiers, CI workflow.
2. ~~Set up CI~~ ✓ Done — `.github/workflows/test.yml` runs the suite on every push.
3. ~~Phase 6 — Payments~~ ✓ Done — atomic transitions, idempotency, void rollback.
4. **RBAC enforcement** — start checking the `role` claim on destructive endpoints.
5. **Refresh tokens + revocation** — close the "token valid until expiry" gap.
6. **Structured logging (`slog`)** — you'll want good logs before you have real traffic.
7. **Dockerfile + CI/CD** — make deployment reproducible.
8. **Rate limiting + CORS + security headers** — the pre-launch security pass.
9. **The remaining feature phases** — dashboard, analytics, notifications, etc.
10. **Observability (metrics, tracing) and DB hardening** — as real load appears.

---

*Built across Phases 1–5. This document reflects the state of the codebase at the end
of Phase 5 (debts). Keep it updated as the project grows.*
