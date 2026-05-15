# CredFlow Code Walkthrough

A senior-engineer commentary on every code file in the project: what it does, why it's structured that way, and what would break if it were done differently. The code itself stays lean; the prose lives here.

Updated through Phase 3 (authentication).

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
```
