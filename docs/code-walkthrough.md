# CredFlow Code Walkthrough

A senior-engineer commentary on every code file in the project: what it does, why it's structured that way, and what would break if it were done differently. The code itself stays lean; the prose lives here.

Updated through Phase 6 (payments) and the test suite.

---

## `cmd/server/main.go`

The entry point. Holds *only* wiring — env loading, migration application, pool construction, router setup, server lifecycle. No business logic ever lives here.

### Structure

```
godotenv.Load              // load .env into process env
mustEnv / envInt32 / envString  // read config
RunMigrations              // apply pending migrations BEFORE the pool opens
database.Connect           // build pgxpool, eager Ping, defer Close
App{DB: pool}              // bundle deps
chi router + middleware    // RequestID, RealIP, Logger, Recoverer, Timeout
http.Server with timeouts  // explicit, not http.ListenAndServe
go ListenAndServe          // serve in goroutine
signal.Notify + select     // block on SIGINT/SIGTERM or startup error
srv.Shutdown(10s ctx)      // graceful drain
```

### Key decisions

- **Migrate BEFORE pool connect.** If the pool opens first, a handler could fire a query against a schema that doesn't exist yet. Migrations are idempotent (golang-migrate tracks state in `schema_migrations`), so re-running on every startup is safe and cheap.
- **App struct, not globals.** `App` holds `DB *pgxpool.Pool`. Handlers are methods on `*App` (e.g. `app.handleHealthDB`) so they close over the pool without referencing a global. Phase 3 will add `JWTSigner`, `Mailer`, etc. as new fields — same pattern.
- **Two health endpoints, on purpose.** `/health` is the *liveness* probe (cheap, no DB). `/health/db` is the *readiness* probe (pings the DB, returns 503 if unreachable). Mixing them up causes outage cascades — a DB blip should *not* restart every pod.
- **Explicit `http.Server` with timeouts.** `http.ListenAndServe` has no read/write timeouts and is slowloris-vulnerable. Always build the struct yourself in production code.
- **Graceful shutdown.** Serve in a goroutine, block main on a signal channel, call `srv.Shutdown(ctx)` with a 10s bound. Deferred `pool.Close()` runs after Shutdown returns, so in-flight queries finish first.
- **`r.Context()` in the DB ping handler.** Inheriting the request context means a client disconnect cancels the query, freeing the pooled connection. Using `context.Background()` keeps the query running for the full 2 seconds even after the user gives up.

### Gotchas

- `signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)` — both signals, not just SIGINT. Docker/k8s/systemd send SIGTERM; Ctrl+C sends SIGINT. Catching only one means your container takes a SIGKILL after the grace period in production.
- `errors.Is(err, http.ErrServerClosed)` is the *expected* error from `ListenAndServe` after `Shutdown` is called — it's how the API signals "I closed cleanly," not a real failure. Easy to miss and end up `log.Fatal`-ing on a normal shutdown.

### What would break if done differently

- **Globals instead of App struct** → unit tests can't swap pools, parallel tests collide on the global, every new dep means another global to thread.
- **Migrate after pool connect** → race: a request can hit a handler before the schema is applied.
- **`http.ListenAndServe`** → no read timeouts → slowloris vulnerability → server hangs under attack.
- **`context.Background()` in handlers** → orphaned queries after client disconnect → connection pool exhaustion under load.

---

## `pkg/response/response.go`

The JSON envelope helpers. Every endpoint in CredFlow returns the same shape, so this package is the single place that formats it.

### The envelope

```json
{ "data": ..., "meta": null | { "page", "pageSize", "total" }, "error": null | { "message", "code" } }
```

### Helpers

| Helper | When to use | Nil-data fallback |
|---|---|---|
| `Success(w, status, data)` | Single-resource and detail endpoints (`GET /api/customers/:id`, `POST /api/...`) | `map[string]any{}` → renders as `{}` |
| `SuccessWithMeta(w, status, data, meta)` | List endpoints (`GET /api/customers`) | `[]any{}` → renders as `[]` |
| `Fail(w, status, message)` | Error responses (4xx, 5xx) | `{}` for data |

### Key decisions

- **`SuccessWithMeta` exists *separately* from `Success`.** List endpoints get a `meta` block; everything else gets `meta: null`. Two helpers is clearer than one helper with optional pagination args.
- **Nil-data fallback differs.** `Success` defaults to `{}` (an empty object); `SuccessWithMeta` defaults to `[]` (an empty array). Clients iterating `data.forEach(...)` on a list endpoint break if you send `null` or `{}`.
- **`writeJSON` order: header → status → body.** Setting `Content-Type` before `WriteHeader` is mandatory. Once `WriteHeader` is called, headers are flushed to the wire — adding `w.Header().Set` afterwards is a silent no-op in Go's HTTP server.

### What would break if done differently

- Calling `w.WriteHeader` before `w.Header().Set("Content-Type", ...)` → header lost, client guesses content type, bug surfaces inconsistently across browsers.
- Letting `Success` accept `nil` and not normalizing → response becomes `"data": null` instead of `{}`, frontend type guards trip.

---

## `pkg/database/database.go`

The Postgres connection pool wrapper. One job: build a pool, eagerly verify it works, hand it back.

### Why a wrapper at all

Calling `pgxpool.New` directly would work, but every call site would need to repeat the same setup (parse config, set sizing, wire timeouts, ping). Centralizing it here gives one place to evolve those defaults.

### Key decisions

- **Config injected, not read from env.** `Connect` takes a `Config` struct. Reading `os.Getenv` belongs to `main.go`, not to a reusable package. Lets tests pass alternate URLs without mutating process env.
- **Eager `Ping`.** `pgxpool.New` is lazy — it does not dial Postgres until the first `Acquire`. Without a ping at startup, a typo in `DATABASE_URL` would only surface on the first user request. Ping → fail fast at startup.
- **Bounded `ConnectTimeout`.** `context.WithTimeout(ctx, cfg.ConnectTimeout)` ensures a hung TCP connect doesn't freeze the binary forever.
- **`pool.Close()` on Ping failure.** If `NewWithConfig` succeeds but Ping fails, the partially-built pool has live goroutines and socket buffers. Close it before returning the error.
- **`MaxConnIdleTime` and `MaxConnLifetime`.** Network middleboxes (load balancers, NAT) silently kill idle TCP connections. Recycling proactively avoids "connection reset by peer" errors after long quiet periods.

### Pool sizing rule of thumb

Total connections across all app instances should stay well under Postgres' `max_connections` (default 100). One app instance with `MaxConns=10` × 5 replicas = 50 connections, leaving headroom. **The bottleneck is the database, not the app.**

### What would break if done differently

- **Skip Ping** → misconfig errors hide until first user traffic, often in production after deploy.
- **Forget `pool.Close()` on Ping failure** → leaked goroutines and file descriptors on every failed startup.
- **Read env vars inside the package** → can't unit-test against a separate DB without setenv hacks.

---

## `pkg/database/migrate.go`

Runs golang-migrate against the same Postgres URL as the pool. Designed to be called from `main.go` on every startup.

### Key decisions

- **Blank imports for drivers.** `import _ "..."` runs the package's `init()` function (which registers the driver with golang-migrate's registry) without exposing any symbols. This is the same pattern as `database/sql` drivers (`_ "github.com/lib/pq"`) and `image/png`. Without these blank imports, `migrate.New` would fail with `"unknown driver pgx5"` at runtime — not compile time.
- **URL scheme rewrite.** Our app's standard URL is `postgres://...`, but golang-migrate's pgx/v5 driver registers itself under the scheme `pgx5://`. `toPgx5URL` does the swap so the rest of the app doesn't need to know.
- **`errors.Is(err, migrate.ErrNoChange)` is not an error.** It's the library's "nothing pending" sentinel. The first call after a fresh migration is success; the next call returns ErrNoChange. Treating it as success is correct.
- **Idempotent + concurrent-safe.** golang-migrate uses Postgres advisory locks internally, so multiple replicas calling `RunMigrations` at boot is safe — only one applies, the rest see ErrNoChange.

### Naming convention for migration files

`NNNN_description.up.sql` and `NNNN_description.down.sql`. The numeric prefix sorts *lexically*, so `0001` not `1`. People who use `1.up.sql` ... `10.up.sql` discover this footgun the day they hit migration #10 and `10` sorts before `2`.

### What would break if done differently

- **Skip the blank imports** → runtime `unknown driver` error.
- **Skip the `ErrNoChange` check** → every startup after the first one `log.Fatal`s with "no change".
- **Forget the advisory-lock guarantee** → migration runner is unsafe under multiple replicas; you'd need a single migration job in CI/CD.

---

## `migrations/0001_init_businesses_users.up.sql` & `.down.sql`

The first schema migration: extensions, a reusable trigger function, businesses table, users table.

### Schema decisions

| Choice | Why |
|---|---|
| `id UUID DEFAULT gen_random_uuid()` | URL-safe, no enumeration leak, no central ID generator. Built into Postgres 13+ via `pgcrypto`. |
| `email CITEXT UNIQUE` | Case-insensitive uniqueness enforced at the database. `TEXT` + "lowercase in app code" has a footgun: forget once and you get duplicate "Bob@x.com" / "bob@x.com" rows. |
| `password_hash TEXT NOT NULL` | bcrypt outputs a 60-char string. TEXT, not BYTEA — easier to inspect in psql, no encoding gymnastics. |
| `role TEXT CHECK (role IN (...))` | Modern Postgres idiom. Easier to ALTER than the native ENUM type. Adding a role = one ALTER, no lock dance. |
| `timestamptz`, not `timestamp` | Timezone-aware. `timestamp` (without tz) is one of Postgres' classic footguns — it stores wall-clock time, not absolute time. |
| `updated_at` trigger, not application code | Postgres has no MySQL-style `ON UPDATE CURRENT_TIMESTAMP`. The reusable `set_updated_at()` function is the canonical workaround. App code can never forget. |
| `business_id` index on users | The hottest access pattern is `WHERE business_id = $1`. Without this index, every tenant query is a sequential scan once the table grows. |
| `ON DELETE CASCADE` on the FK | Deleting a business should take its users with it. The default (`NO ACTION`/`RESTRICT`) would block the delete until users are cleaned up by hand. |

### Down-migration discipline

`DROP TABLE` implicitly drops attached triggers. We don't `DROP EXTENSION` — extensions are shared infrastructure and other tables may rely on them. The `set_updated_at()` function is dropped only because migration #1 owns it exclusively; future migrations adding tables that use it must NOT drop it in their down step.

### What would break if done differently

- **`TEXT` instead of `CITEXT` for email** → case-insensitive uniqueness lives only in app code → eventually a code path forgets → duplicate accounts.
- **No `business_id` index** → tenant queries start slow at ~10k rows, unusable at ~1M.
- **No `ON DELETE CASCADE`** → business deletion returns an FK violation; cleanup requires manual SQL.
- **`timestamp` (no tz) instead of `timestamptz`** → silent timezone drift between dev/prod, especially around DST transitions.
- **Native ENUM type instead of CHECK** → adding a new role becomes a lock-taking `ALTER TYPE`, painful in production.

---

## `internal/auth/models.go`

Domain types and request/response shapes for auth.

### Key decisions

- **`json:"-"` on `PasswordHash`.** The field stays on the struct (the service needs it for bcrypt comparison) but the JSON encoder is forbidden from emitting it. Even if a future handler accidentally returns `user` directly, the hash cannot leak.
- **`*string` for nullable optional fields.** `Business.Industry` and `Business.Size` are nullable in Postgres. Using `*string` (not `string`) distinguishes "absent" from "empty string." `omitempty` then drops nil values from JSON.
- **`Token` is `omitempty`.** Auth responses for register/login carry a token; `/me` does not need one and would expose an empty-string field without `omitempty`.
- **camelCase JSON tags throughout.** Matches the envelope spec (`meta.pageSize`) and frontend conventions.

---

## `internal/auth/password.go`

A two-function wrapper around bcrypt.

### Key decisions

- **Wrap the library even though it's two lines.** Centralizes the cost factor and the algorithm choice. If we switch to argon2id later, one file changes.
- **Cost factor 12.** OWASP 2025 recommendation. ~40ms per hash on modern hardware — slow enough to deter brute force, fast enough to not DoS the login endpoint.
- **Bcrypt embeds the salt in the hash.** No separate salt column, no manual salt handling. `CompareHashAndPassword` extracts the salt automatically.

### What would break if done differently

- `sha256(password)` instead of bcrypt → rainbow-table crackable in seconds.
- Cost 4-6 → trivially brute-forceable on a modern GPU.
- Cost 14+ → 1+ second login latency, your auth endpoint becomes a self-DoS surface.

---

## `internal/auth/jwt.go`

JWT minting and verification, HS256.

### Key decisions

- **HS256 (symmetric).** One secret, used for both sign and verify. Simplest and matches the project's single-server-trust model. RS256 (asymmetric) is for cases where third parties need to verify without holding the signing key.
- **Custom claims `bid` and `role` alongside `RegisteredClaims`.** `bid` = business ID. Embedding the tenant ID in the token means every authenticated request scopes to the right tenant without a DB lookup.
- **Algorithm check is mandatory.** The verifier rejects any token whose signing method isn't HMAC. This blocks the famous `alg=none` attack — without the check, attackers can forge tokens by setting `alg=none` and dropping the signature.
- **Sentinel `ErrInvalidToken`.** Middleware doesn't care whether the token was expired, malformed, or signature-mismatched — all are 401. One sentinel keeps that mapping clean.

### What would break if done differently

- Skip the algorithm check → `alg=none` attack lets anyone forge tokens.
- No `ExpiresAt` → tokens live forever; stolen token = permanent compromise.
- 16-character `JWT_SECRET` → offline brute-force (`hashcat`) cracks HS256 with weak keys in hours.

---

## `internal/auth/repository.go`

The SQL layer. Translates Go calls into queries. No business logic.

### Key decisions

- **`DBTX` interface for transactional flexibility.** Both `*pgxpool.Pool` and `pgx.Tx` satisfy it (matching `QueryRow` and `Exec` signatures). Repo methods accept any `DBTX`, so the same code works inside and outside a transaction. Without this, every method would need a transactional twin.
- **`SELECT` named columns, not `*`.** Reordering columns in a future migration won't silently corrupt `Scan` targets.
- **`RETURNING ...` on INSERT.** Lets us get DB-generated values (UUID, timestamps) in one round-trip instead of `INSERT` then `SELECT`.
- **`NULLIF($n, '')` for optional strings.** The frontend often sends `""` for empty optional fields. `NULLIF(value, '')` translates that to NULL inside the SQL, no Go-side preprocessing needed.
- **`pgconn.PgError` for SQLSTATE matching.** `errors.As(err, &pgErr)` lets us match unique-violation (`23505`), foreign-key violation (`23503`), etc. by structured code, not fragile string matching against Postgres' error messages.
- **Sentinel errors at the package boundary.** `ErrUserNotFound`, `ErrEmailTaken`. The service layer cares about *meaning*; the repo translates pgx errors into domain errors.

### What would break if done differently

- `SELECT *` → silent field corruption after a column reorder migration.
- String matching against Postgres errors → breaks when PG changes the wording in a minor version.
- No `DBTX` interface → every method doubled to support transactions, code triples.

---

## `internal/auth/service.go`

Orchestration: validation, transaction management, token minting, user-enumeration safety.

### Key decisions

- **`pgx.BeginFunc` over manual transactions.** The manual pattern (`tx, _ := db.Begin(); defer tx.Rollback(); ... tx.Commit()`) is footgun-heavy. `BeginFunc(ctx, db, fn)` commits on `fn` returning nil, rolls back otherwise. One line of nesting handles all the cleanup.
- **`Register` is atomic.** Business creation + user creation share one transaction. If the user INSERT fails (e.g. email conflict), the business is rolled back. No orphan tenants.
- **User-enumeration safety in `Login`.** Whether the email doesn't exist or the password is wrong, we return the *same* error (`ErrInvalidCredentials`). Sites that say "email not registered" let attackers build a list of valid emails by trial.
- **Password length cap of 72.** Bcrypt silently truncates inputs longer than 72 bytes. Without the cap, two passwords sharing the first 72 characters both validate against the same hash. Enforce explicitly.
- **Validation lives in the service, not the handler.** The service is the API boundary. Future CLI tools or admin scripts that create users get the same checks without duplicating code.
- **`fmt.Errorf("%w: detail", ErrValidation)` for typed wrapping.** The handler matches with `errors.Is(err, ErrValidation)` for the HTTP-status mapping *and* uses `err.Error()` for the user-facing message — both bits of info from one error.

### What would break if done differently

- No transaction in Register → user insert fails after business succeeds → orphan business survives.
- "Email not found" vs "wrong password" distinct errors → trivial email enumeration.
- No password length cap → bcrypt truncates silently → confusing security bug.

---

## `internal/auth/handler.go`

HTTP-boundary glue. Parse JSON, call service, write response.

### Key decisions

- **Centralized error → status mapping in `writeServiceError`.** One switch over sentinel errors. New error type? One case to add.
- **Generic message on 500.** `response.Fail(w, 500, "internal server error")` — never echo the inner error to the wire. The structured server log keeps the real message; the public response stays opaque.
- **`UserIDFromContext` from the auth package, not the handler.** Handler doesn't know whether auth comes from a JWT, a session cookie, or something else. The middleware writes the context value, the handler reads it. Replace the middleware tomorrow, the handler doesn't change.

---

## `internal/auth/context.go`

Context key plumbing — the seam between middleware and handler.

### Key decisions

- **Private typed key, not a string.** Go's context docs explicitly warn that string keys are collision-prone across packages. `type ctxKey int` with private values means no other package can collide with ours, even by accident.
- **Three values: user ID, business ID, role.** All three are non-secret IDs that downstream handlers will need. Putting them in context once at the middleware layer beats pulling JWT claims apart in every handler.
- **Symmetric write/read helpers (`WithUserContext` / `*FromContext`).** Keeps the key handling inside this file. Handlers don't see the `ctxKey` type at all.

### What would break if done differently

- String key like `"userID"` → another package using `"userID"` silently overwrites your value at runtime, no compile error.
- Storing the whole `User` struct in context → every authenticated request blocks on a DB lookup before reaching the handler.

---

## `internal/middleware/auth.go`

Bearer-token middleware. Extracts the token, verifies via `JWTService`, injects user context.

### Key decisions

- **`func(http.Handler) http.Handler`** signature. The canonical Go middleware shape. Chi accepts it directly via `Use`.
- **Factory function `RequireAuth(jwt *auth.JWTService)`.** Takes the JWT service as a dependency and returns the middleware. This is how middleware acquires runtime config without becoming a singleton.
- **`r.WithContext(ctx)` to thread the new context.** Returns a shallow copy of the request — never mutate the existing request's context directly, that's not safe across goroutines or future handlers.
- **Reject `Authorization` headers that don't start with `Bearer `.** 401 with a clear message. We don't try to accept tokens via query params or cookies — Bearer in the header is the unambiguous canonical form for API auth.

### What would break if done differently

- Apply this middleware to public routes (`/login`) → chicken-and-egg: you need a token to get a token.
- Mutate `r.Context()` instead of `r.WithContext(...)` → race conditions across handler goroutines.
- Read JWT directly in handlers instead of using context → coupling between every handler and the JWT format. Switching auth methods means rewriting every handler.

---

## `cmd/server/main.go` (Phase 3 additions)

Wiring now constructs the auth stack and mounts protected/public routes via `chi.Router.Group`.

### Key additions

- **`auth.NewJWTService(jwtSecret, jwtTTL)` constructed before service.** Single instance shared by middleware and service — they sign and verify with the same secret.
- **`r.Route("/api/auth", ...)` for the URL prefix.** Inside, `r.Post` for public endpoints and `r.Group` for the protected `/me`. `Group` applies its middleware (`appmiddleware.RequireAuth`) only to routes registered inside it. Public siblings are unaffected.
- **Import aliasing.** `chimiddleware` for Chi's built-ins, `appmiddleware` for ours. Two packages with the same final name — aliasing resolves the conflict without ambiguity.
- **`envDuration` for `JWT_TTL`.** `time.ParseDuration("24h")` accepts human-friendly strings like `"24h"`, `"1h30m"`, `"500ms"`. Better than parsing an integer and multiplying.

---

## `migrations/0002_create_customers.up.sql` & `.down.sql`

Adds the `customers` table — multi-tenant, soft-deletable, indexed for the common read paths.

### Schema decisions

| Choice | Why |
|---|---|
| `business_id UUID REFERENCES businesses(id) ON DELETE CASCADE` | Every customer belongs to a tenant. Deleting a business takes its customers with it. The most-queried column. |
| `email CITEXT` (nullable) | Customers might not have email. CITEXT keeps case-insensitive matching consistent with the users table. |
| `risk_level TEXT CHECK (... IN ('low','medium','high'))` | Same TEXT + CHECK idiom as `users.role` — ALTER-friendly, no ENUM type rigidity. |
| `credit_limit NUMERIC(14,2)` | Money column. **Never `float`** — floating-point cents = silent rounding bugs in financial software. |
| `deleted_at TIMESTAMPTZ NULL` | Three states in one column: NULL = active, non-NULL = soft-deleted at that timestamp. |
| `customers_business_id_active_idx` *partial* on `(business_id, created_at DESC) WHERE deleted_at IS NULL` | The hot-path index. Excludes soft-deleted rows → smaller, faster. Includes `created_at DESC` for the default sort. |
| `customers_business_id_email_uniq` *partial unique* on `(business_id, email) WHERE email IS NOT NULL AND deleted_at IS NULL` | Per-tenant email uniqueness. Partial so: NULL emails are allowed; deleted rows free up their email for reuse. |

### Why partial indexes matter

A plain `UNIQUE(business_id, email)` would:
1. Reject inserts if any deleted row had the same email — leaking soft-delete state to the API.
2. Reject `NULL` emails as duplicates of each other (Postgres treats NULLs as distinct in UNIQUE, but multiple NULLs are still confusing in practice).

The partial unique with `WHERE email IS NOT NULL AND deleted_at IS NULL` says: "uniqueness only applies among active rows that actually have an email." That's exactly the semantic we want.

---

## `internal/customers/models.go`

Domain type + request shapes + list query.

### Key decisions

- **Pointer fields on `Customer`** for nullable columns (`Email`, `Phone`, `CompanyName`, `Address`, `Notes`). `*string` distinguishes "value is NULL in DB" from "value is empty string."
- **`UpdateRequest` uses pointers on every field.** This is *the* PATCH semantic: a `nil` pointer means "field was not in the JSON, leave unchanged." A non-nil pointer to `""` means "client explicitly cleared this field." Plain `string` collapses the two.
- **`ListQuery` is a value struct, not a URL parser.** Handler does the URL parsing; service receives a clean Go struct. Keeps query-parsing details out of the service.
- **`DeletedAt` deliberately absent from JSON.** API contract is "deleted = doesn't exist." Soft delete is a storage detail.

---

## `internal/customers/repository.go`

The SQL layer. Tenant-scoped on every method.

### Key decisions

- **`businessID` is always the first non-context argument.** No method allows lookup by ID alone — the signature itself enforces tenant scoping. A code review can spot `GetByID(id)` instantly as wrong.
- **Every query filters `WHERE business_id = $1 AND deleted_at IS NULL`** (except SoftDelete, which omits `deleted_at IS NULL` from its own SET clause but keeps the WHERE).
- **`customerColumns` constant + `scanCustomer` helper.** DRY across five repo methods. One column-list change in one place.
- **Dynamic UPDATE with `add(col, val)` closure.** Builds parallel `sets` slice and `args` slice atomically so the `$N` placeholder index always matches the args length. The naive separate-slices approach is footgun-rich (off-by-one nightmare).
- **`pgconn.PgError` for `23505` matching.** Postgres SQLSTATE for unique violation. `errors.As` extracts the typed error; we map it to `ErrEmailTaken`. Don't match on error message strings — they change between PG versions.
- **`List` returns `(items, total, error)`.** Two queries — one for the page, one for `COUNT(*)`. Acceptable for typical SaaS scale; cursor pagination is a future upgrade when count cost becomes painful.
- **Secondary sort by `id ASC`** in `ORDER BY`. Stabilizes pagination when sort values tie. Without it, two rows with the same `created_at` could appear on different pages or be skipped entirely.
- **`SoftDelete` returns `ErrNotFound`** when no row matched (already deleted or never existed in this tenant). Uses `tag.RowsAffected() == 0` as the signal — no extra query needed.

### What would break if done differently

- Forget `WHERE business_id = $1` → cross-tenant data leak. The number one multi-tenant bug.
- Parameterize the sort column (`ORDER BY $1`) → PG interprets `$1` as the column *number*, not name; baffling silent reorder.
- No `id` tiebreaker in ORDER BY → pagination skips/duplicates on equal sort values.
- Match unique violations by `err.Error()` string match → breaks on the next PG minor version.

---

## `internal/customers/service.go`

Validation, defaults, sort whitelisting, pagination clamping.

### Key decisions

- **Sort *whitelist* via `sortFields map[string]string`.** Maps public API names (`createdAt`, `riskLevel`) to SQL column names (`created_at`, `risk_level`). User input that isn't a key → reject. **This is the only safe pattern for dynamic ORDER BY** — column names cannot be parameterized in SQL.
- **`-` prefix means DESC** (`?sort=-name`). Common REST API convention. Easy to construct client-side, easy to parse server-side.
- **`map[string]struct{}` for set membership.** Zero-byte values; idiomatic "is X in this set." Reads better than `map[string]bool` with an unused value.
- **Pagination clamping centralized here.** `page < 1 → 1`, `pageSize > 100 → 100`. Repo trusts what it gets — bypass the service (CLI, tests) and it still functions; service is where the defaults live.
- **`validateUpdate` operates through the pointer** (`if r.Name != nil` then `*r.Name`). Only validates fields the client actually sent. Empty unsupplied fields don't fail validation.

### What would break if done differently

- Pass user's `sort` straight to SQL → SQL injection. `?sort=name;DROP TABLE customers--` is just one example.
- No `pageSize` cap → client can request 1,000,000 → OOM on the count + scan.
- Use `map[string]bool` for the risk-level set → wastes a byte per entry, less idiomatic.

---

## `internal/customers/handler.go`

HTTP boundary. URL params, query parsing, error → status mapping.

### Key decisions

- **`auth.BusinessIDFromContext(r.Context())` at the top of every handler.** Single point of "no auth = 401." The auth middleware already would have rejected unauthenticated requests, but this is defense-in-depth — if someone forgets `RequireAuth` on a route, the handler still won't leak data.
- **`chi.URLParam(r, "customerId")` for path variables.** Chi reads from the URL pattern (`/{customerId}`); no manual splitting.
- **`atoiOr` for safe query-param parsing.** `?page=abc` falls back to default. Never 500 on a non-critical malformed param.
- **`204 No Content` for DELETE.** REST convention. No body, no envelope.
- **`response.SuccessWithMeta` for List.** The pagination helper from Phase 1 finally has a real caller.

### What would break if done differently

- No `BusinessIDFromContext` check → if `RequireAuth` is ever accidentally removed from the route, every handler silently runs with empty `businessID`, returning empty lists everywhere.
- Return `Success` instead of `204` for DELETE → still works, but bloats the response and breaks REST convention.
- Crash on bad query params → trivial DoS vector.

---

## `cmd/server/main.go` (Phase 4 additions)

Three new constructor lines, one new route block. The pattern is meant to be boring and repeatable — that's the point.

### Key additions

- **`r.Route("/api/customers", ...)` with `r.Use(RequireAuth(jwtSvc))` inside.** All five customer routes need auth. No inner `Group` like we used for `/api/auth/me` because there are no public sibling routes.
- **Three constructor lines** (repo → service → handler) per module. Future phases (debts, payments) will copy-paste-modify these three lines. When the wiring approaches ~30 lines we'll extract a `bootstrap()` function; for now it stays in main.

---

## `migrations/0003_create_debts.up.sql` & `.down.sql`

The `debts` table — money owed by a customer, tenant-scoped, soft-deletable, with a status lifecycle.

### Schema decisions

| Choice | Why |
|---|---|
| `amount NUMERIC(14,2) CHECK (amount > 0)` | Money is never `float`. A zero/negative debt is nonsense — the DB rejects it. |
| `status TEXT CHECK (... IN ('pending','partial','paid'))` | Stored lifecycle state. `partial` is reserved for Phase 6 (payments); Phase 5 transitions only pending→paid. |
| `issued_date DATE`, `due_date DATE` | **Calendar days, not instants.** A due date is "June 30", not "June 30 00:00:00+01". `DATE` avoids timezone bugs in overdue calculations. |
| `paid_at TIMESTAMPTZ` | **An instant, not a day.** The exact moment of payment. Deliberate contrast with the `DATE` columns. |
| `business_id` FK `ON DELETE CASCADE` | Tenant teardown should be clean. |
| `customer_id` FK `ON DELETE RESTRICT` | Protects financial history. Customers are soft-deleted so this never fires in normal flow — but if a customer is ever hard-deleted, RESTRICT blocks silent loss of their debts. |
| `CHECK (due_date >= issued_date)` | Table-level (spans two columns). A debt due before it was issued is physically unstorable. |
| 3 partial indexes | Tenant list `(business_id, created_at DESC)`, per-customer `(customer_id)`, due-date `(business_id, due_date)` — all `WHERE deleted_at IS NULL`. |

### `overdue` is derived, not stored

A debt is overdue when `due_date < CURRENT_DATE AND status <> 'paid'`. That depends on *today's date* — a stored `overdue` column would need a nightly cron job to stay correct. Instead it's computed in every SELECT. Always correct, zero background infrastructure.

### Currency: deliberately omitted

No multi-currency requirement exists in the spec, and `businesses` has no currency column. Adding `currency` now would be speculative (YAGNI). It's a clean future migration if multi-currency is ever needed.

---

## `internal/debts/models.go`

Domain type + request shapes + list query.

### Key decisions

- **Derived fields on the struct: `Overdue bool`, `AmountRemaining float64`.** Computed in SQL (`overdue`) or trivially (`amount_remaining` = amount or 0). Not columns. The client can't tell the difference — that's the point.
- **Dates as `string` in request types** (`CreateRequest.DueDate`, `UpdateRequest.DueDate`). Go's `encoding/json` only unmarshals `time.Time` from RFC3339; a bare `"2026-06-30"` fails. Taking strings and parsing in the service yields a friendly *"dueDate must be YYYY-MM-DD"* error.
- **`*time.Time` for `PaidAt`.** Nil until paid.

---

## `internal/debts/repository.go`

The SQL layer. Tenant-scoped, same discipline as customers, plus two patterns unique to debts.

### Pattern 1 — `INSERT ... SELECT ... WHERE EXISTS` (atomic customer check)

`Create` inserts the debt *only if* an active customer exists in the tenant:

```sql
INSERT INTO debts (...)
SELECT $1, $2, ...
WHERE EXISTS (SELECT 1 FROM customers WHERE id = $2 AND business_id = $1 AND deleted_at IS NULL)
RETURNING ...
```

The naive alternative — "SELECT to check the customer, then INSERT" — is a TOCTOU race: the customer could be deleted between the two statements. Folding the check into the INSERT makes it one atomic operation under one snapshot. If the customer isn't valid, 0 rows insert, `RETURNING` is empty, `pgx.ErrNoRows` fires → mapped to `ErrCustomerNotFound`.

### Pattern 2 — `MarkPaid` two-step disambiguation

`UPDATE ... SET status='paid' WHERE ... AND status <> 'paid'` affects 0 rows in *two* different cases: the debt doesn't exist, or it's already paid. A single statement can't tell them apart. So on 0 rows, `MarkPaid` does one follow-up `Get` to disambiguate: `ErrNotFound` vs `ErrAlreadyPaid`. The extra query only runs on the failure path — the happy path stays a single statement.

### Other notes

- **`debtSelect` constant** holds the column list *plus* the two derived expressions (`overdue`, `amount_remaining`). Every read uses it, so the derived values are computed identically everywhere — no drift between Get, List, Create, MarkPaid.
- **`CustomerExists`** — a cheap `SELECT EXISTS(...)` used by the nested endpoint to 404 a bad customer id instead of returning an empty list.

---

## `internal/debts/service.go`

Validation, date parsing, sort whitelist, status filtering.

### Key decisions

- **Go's reference-date layout: `"2006-01-02"`.** Go date layouts ARE a specific reference time (`01/02 03:04:05PM '06 -0700`), not strftime codes. `time.Parse("2006-01-02", "2026-06-30")` parses a `YYYY-MM-DD` date.
- **`due.Before(issued)` guard.** Service-layer friendly error; the DB `CHECK` is the backstop.
- **`ListByCustomer` reuses `List`.** It verifies the customer exists, sets `q.CustomerID`, and delegates — no duplicated pagination/filter/sort logic.
- **Sort whitelist** (`createdAt`, `dueDate`, `amount`, `status`, `updatedAt`) — same anti-injection pattern as customers.

---

## `internal/debts/handler.go`

HTTP boundary. Same shape as the customers handler, plus:

- **`MarkPaid`** — handles the `POST /api/debts/{debtId}/mark-paid` action endpoint. Returns the updated debt (200), or 409 if already paid.
- **`ListByCustomer`** — handles the nested `GET /api/customers/{customerId}/debts`. Reads `customerId` from the path, debt filters from the query string.
- **`parseListQuery` helper** — shared by `List` and `ListByCustomer` to build a `ListQuery` from URL params.
- Error mapping adds `ErrAlreadyPaid → 409` and `ErrCustomerNotFound → 404`.

---

## `cmd/server/main.go` (Phase 5 additions)

- **Constructor trio** for debts (repo → service → handler).
- **New `/api/debts` route group** behind `RequireAuth`, with the `mark-paid` action as `POST /{debtId}/mark-paid`.
- **Nested route** `GET /api/customers/{customerId}/debts` added *inside* the existing `/api/customers` group — it's a customer-shaped URL served by the debts handler. Route placement and handler ownership are independent.

---

## `migrations/0004_create_payments.up.sql` & `.down.sql`

The `payments` table. Linked to a customer (required) and a debt (optional).
Idempotency keys live here.

### Schema decisions

| Choice | Why |
|---|---|
| `customer_id` FK `ON DELETE RESTRICT` | Same as debts — protect financial history. |
| `debt_id` FK *nullable* `ON DELETE RESTRICT` | Payments may be unattributed cash, or tied to a specific debt. Nullable FK is a valid Postgres construct: NULL allowed, non-NULL must reference. |
| `amount NUMERIC(14,2) CHECK (amount > 0)` | Money. No floats, no zero/negative. |
| `method TEXT CHECK (... IN (...))` | Small enum: cash, card, bank_transfer, check, mobile_money, other. |
| `paid_at TIMESTAMPTZ DEFAULT NOW()` | When the money actually moved (may be backdated by recording later). Distinct from `created_at`. |
| `idempotency_key TEXT` (nullable) | Optional client-supplied retry guard. |
| Partial unique on `(business_id, idempotency_key)` | Per-tenant uniqueness, ignoring NULLs and voided rows. |
| 3 partial active-row indexes | Tenant list, per-customer, per-debt. |

### The idempotency key is enforced by a database index

There is no application-layer lock, no Redis, no separate idempotency-keys
table. The unique index *is* the mechanism. The repository does INSERT;
if SQLSTATE 23505 fires on this specific constraint, the request is a
replay — fetch the existing payment and return it. Persistent across
restarts, no race window, no extra infrastructure.

---

## `internal/payments/models.go`

Three structs, same shape as the other modules. Notable: `IdempotencyKey`
is read from the request body for now (`POST` body). A header-based variant
(`Idempotency-Key: <uuid>`) is a simple later addition without a model
change.

---

## `internal/payments/repository.go`

### Three patterns layered together

1. **`INSERT ... SELECT ... WHERE EXISTS`** with branching args — atomic
   existence check for both the customer (always) and the debt (if linked).
   No TOCTOU race.
2. **Idempotency by constraint name** — `pgconn.PgError.ConstraintName` is
   matched against the literal constraint name from migration 0004. SQLSTATE
   alone is not enough: many unique constraints can fail; only one is the
   idempotency guard. The constraint-name check is a teaching example of how
   to be precise about which integrity violation just happened.
3. **`RecomputeDebtStatus(ctx, db, businessID, debtID)`** — a CTE-based
   UPDATE that reads the current debt amount + the SUM of active payments
   and transitions status. The `WHEN current_status = 'paid' THEN 'paid'`
   guard preserves administrative closes: voiding a payment against a
   mark-paid debt does NOT reopen the debt. The administrative decision
   outranks the arithmetic.

### `DBTX` interface, again

Both `*pgxpool.Pool` and `pgx.Tx` satisfy `interface{ QueryRow; Exec; Query }`.
Every payment-repo method takes a `DBTX` so the service can run Create plus
RecomputeDebtStatus under one transaction. Same pattern as the auth repo.

---

## `internal/payments/service.go`

### One transaction per money-moving operation

Both `Create` and `Delete` wrap their work in `pgx.BeginFunc`:
- **Create:** insert payment → recompute linked debt status → commit, or
  rollback if anything errors. On idempotency replay the recompute is
  skipped (the original request already did it).
- **Delete (void):** soft-delete payment → fetch debt_id from the soft-delete
  result → recompute debt status → commit.

This is the *single* most important pattern in Phase 6. A naive
non-transactional implementation would let the server crash between INSERT
payment and UPDATE debt, leaving the totals inconsistent. The transaction
makes that impossible.

### Why the service owns the transaction (not the repo)

Repositories speak SQL. Services compose operations. The transaction is a
*composition* decision: "these reads and writes belong together as one unit
of work." That's the service's responsibility. Repository methods stay
transaction-agnostic (they take `DBTX`) so they can be composed differently
later.

---

## `internal/debts/repository.go` — Phase 6 update

`debtSelect` now derives `amount_paid` (SUM of active payments) and
`amount_remaining` (= amount when mark-paid, else amount minus the SUM)
via a correlated subquery. The subquery references the outer row by the
bare table name `debts.id` — which works in SELECT, INSERT...RETURNING,
and UPDATE...RETURNING contexts uniformly. The `payments_debt_id_active_idx`
makes the subquery cheap.

The administrative-close case (`status = 'paid'`) stays as 0 remaining
regardless of payment sum — so `mark-paid` and "paid via payments" both
produce `amountRemaining = 0`, but the *path* is different.

---

## `cmd/server/main.go` (Phase 6 additions)

- New constructor trio: `paymentRepo → paymentSvc → paymentHandler`.
- `paymentHandler` takes `debtRepo` as a second dependency for `CreateForDebt`
  (which reads the debt to extract `customerId` from the URL).
- Two new nested mounts: `GET /api/customers/{id}/payments` (in the customers
  group) and `POST /api/debts/{id}/payments` (in the debts group). Standalone
  CRUD in `/api/payments`.

---

## Test suite

### Layout

```
pkg/response/response_test.go            unit  — envelope helpers
internal/auth/password_test.go           unit  — bcrypt round-trip, salting
internal/auth/jwt_test.go                unit  — JWT incl. alg=none rejection
internal/auth/service_test.go            unit  — register validation
internal/customers/service_test.go       unit  — sort whitelist + validation
internal/debts/service_test.go           unit  — date parsing + validation
internal/middleware/auth_test.go         http  — RequireAuth behaviour
internal/testutil/testdb.go              helper — test DB connect + truncate
test/integration/helpers_test.go         integration helpers (build tag)
test/integration/auth_test.go            integration — register/login/me + enumeration
test/integration/tenant_isolation_test.go integration — the headline isolation test
test/integration/payments_test.go         integration — status transitions, idempotency, void
internal/payments/service_test.go        unit  — validation + sort whitelist + date parsing
Makefile                                 developer commands
.github/workflows/test.yml               CI workflow
```

### How tiers separate

- **Unit + HTTP-handler tests** live next to the code they test (Go convention).
  They run with plain `go test ./...` — no Docker required.
- **Integration tests** live under `test/integration/` and start with the build
  constraint `//go:build integration`. They are *skipped* by `go test ./...` and
  run only with `-tags=integration`. They need a Postgres database reachable at
  `TEST_DATABASE_URL` (default: `credflow_test` in the local container).

### Patterns worth keeping

- **Table-driven tests with `t.Run(name, ...)`**: one test function, many input
  rows, each row a named subtest. Failures point at the row name.
- **`httptest.NewRecorder`** for in-process handler/middleware tests;
  **`httptest.NewServer`** for full-stack integration tests over real HTTP.
- **`json.RawMessage`** in the response envelope helper so each test decodes the
  data block into whatever shape it expects.
- **`t.Skipf` when DB unreachable** — keeps the unit suite green when Docker is
  off; integration tests skip with a clear message instead of failing mysteriously.
- **Security regression tests** (`TestJWT_rejectsAlgNone`,
  `TestTenantIsolation`) — once a class of bug is locked in by a test, it can't
  silently come back.

### Running

```bash
make test               # unit only, fast — runs on every save in dev
make test-integration   # integration only, needs Postgres
make test-all           # everything
```

CI (`.github/workflows/test.yml`) runs the full suite on every push and PR,
against a Postgres service container.

---

## `.env`

| Variable | Purpose |
|---|---|
| `PORT` | HTTP listen port. Default 8080 if unset. |
| `APP_ENV` | `development` / `staging` / `production`. Used by later phases for verbose-logging toggles. |
| `DATABASE_URL` | `postgres://user:pass@host:port/db?sslmode=...`. Production should be `sslmode=require` or stricter. |
| `DB_MAX_CONNS` | Pool cap. Default 10 in `main.go`. |
| `DB_MIN_CONNS` | Idle connections kept warm. Default 2. |
| `JWT_SECRET` | HS256 signing key. Generate via `openssl rand -base64 64`. Minimum 32 bytes — shorter keys are offline-crackable. |
| `JWT_TTL` | Access token lifetime. Accepts Go duration strings (`24h`, `15m`, `1h30m`). Default 24h. |

`.env` is loaded by `godotenv.Load()` in `main.go`. A missing `.env` is non-fatal — production injects env vars from the platform (k8s, fly.io, systemd) and no file exists.

**Never commit a real `.env` to git.** Add it to `.gitignore` and commit a `.env.example` instead.

---

## Operations cheatsheet

```bash
# Postgres lifecycle (Docker)
docker start credflow-pg               # bring it up
docker stop credflow-pg                # bring it down
docker logs credflow-pg                # tail logs
docker volume rm credflow-pg-data      # NUKE all data (after stop+rm)

# Connect via psql
PGPASSWORD=credflow_dev psql -h localhost -U credflow -d credflow

# Run server
go run ./cmd/server

# Quick checks
curl http://localhost:8080/health
curl http://localhost:8080/health/db

# Auth flow
curl -X POST http://localhost:8080/api/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"businessName":"Acme","email":"a@a.com","password":"pass1234","name":"A"}'

TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"a@a.com","password":"pass1234"}' \
  | jq -r .data.token)

curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/auth/me

# Customers (all protected)
curl -X POST http://localhost:8080/api/customers \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"name":"John Roe","email":"john@roe.test","riskLevel":"medium","creditLimit":5000}'

# List with filters and pagination
curl -H "Authorization: Bearer $TOKEN" \
  'http://localhost:8080/api/customers?search=john&riskLevel=medium&sort=-createdAt&page=1&pageSize=20'

# Detail
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/customers/{id}

# Patch (only fields you want changed)
curl -X PATCH http://localhost:8080/api/customers/{id} \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"riskLevel":"high","creditLimit":1000}'

# Soft delete
curl -X DELETE -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/customers/{id}

# Debts (all protected)
curl -X POST http://localhost:8080/api/debts \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"customerId":"{cid}","amount":1500.00,"description":"Invoice #42","dueDate":"2026-06-30"}'

# List with filters
curl -H "Authorization: Bearer $TOKEN" \
  'http://localhost:8080/api/debts?status=pending&overdue=true&sort=dueDate&page=1'

# Mark a debt paid
curl -X POST -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/debts/{id}/mark-paid

# One customer's debts (nested)
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/customers/{cid}/debts
```
