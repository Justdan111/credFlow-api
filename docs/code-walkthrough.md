# CredFlow Code Walkthrough

A senior-engineer commentary on every code file in the project: what it does, why it's structured that way, and what would break if it were done differently. The code itself stays lean; the prose lives here.

Updated through Phase 2 (database layer).

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

## `.env`

| Variable | Purpose |
|---|---|
| `PORT` | HTTP listen port. Default 8080 if unset. |
| `APP_ENV` | `development` / `staging` / `production`. Used by later phases for verbose-logging toggles. |
| `DATABASE_URL` | `postgres://user:pass@host:port/db?sslmode=...`. Production should be `sslmode=require` or stricter. |
| `DB_MAX_CONNS` | Pool cap. Default 10 in `main.go`. |
| `DB_MIN_CONNS` | Idle connections kept warm. Default 2. |

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
```
