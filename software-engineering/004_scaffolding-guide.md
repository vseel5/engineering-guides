# Software Architecture Scaffolding Guide

Language/framework-agnostic. Based on industry patterns (Clean Architecture, Domain-Driven Design, 12-Factor App). Copy to any project.

---

## Core Principle

**Each layer has one job. Dependencies point inward only.**

```
HTTP / UI
   ↓
Controllers / Handlers
   ↓
Services / Use Cases
   ↓
Domain / Business Logic
   ↓
Data Access / Repositories
   ↓
Database / External APIs
```

No layer skips another. No layer knows about layers above it.

---

## Backend Layers

### 1. Routes / Handlers
- Mount middleware
- Define URL + HTTP method
- Zero logic

```
POST /api/v1/users → [authenticate] → [validate] → controller.createUser
```

### 2. Controllers
- Parse req (body, params, query)
- Call one service method
- Send res
- No DB, no business rules

```ts
async createUser(req, res) {
  const dto = req.body;           // parse
  const user = await userService.create(dto);  // delegate
  res.status(201).json(user);     // respond
}
```

### 3. Services / Use Cases

**Use Case** = single user action, one public method `execute()`. Easier to test, prevents bloat.
**Service** = groups related use cases for simple CRUD.

Use Cases for complex logic, Services for simple CRUD. Both are valid — pick based on complexity.

```
# Use Case pattern (complex logic)
useCases/
  registerUser.ts       # class RegisterUserUseCase { execute(dto) }
  changePassword.ts     # class ChangePasswordUseCase { execute(dto) }

# Service pattern (simple CRUD)
services/
  userService.ts        # register(), updateProfile(), changePassword()
```

Rules for both:
- Business logic lives here
- Orchestrates repositories + shared logic
- No HTTP concepts (no req/res)

### 4. Repositories / Data Access
- All DB queries here
- Returns plain objects containing only business fields — no ORM methods attached (no `.save()`, no `.update()`), no database-specific types (`ObjectId` → `string`, `Date` stays `Date`)
- Example: `{ id: 'uuid', email: 'user@example.com', createdAt: '2025-01-01T00:00:00Z' }` — NOT a Prisma model or Mongoose document
- Swappable (Postgres → MongoDB = only this layer changes)

### 5. Domain / Shared Logic
- Pure functions only: output depends only on input
- No DB, no HTTP, no side effects
- Independently unit-testable
- Examples: `classifyAttendance()`, `computeRating()`, `checkLockout()`

---

## Layer Contract Rules

| Layer | Can call | Cannot call |
|-------|----------|-------------|
| Routes | Controller, middleware | Service, DB |
| Controller | Service | DB, domain logic directly |
| Service | Repository, domain logic | req/res, HTTP |
| Repository | DB client only | Service, HTTP |
| Domain logic | Nothing (pure) | DB, HTTP, state |

Violation = drift. Drift = bugs that are hard to trace.

---

## File Naming Convention

```
services/{feature}/
  {feature}.routes.ts
  {feature}.controller.ts
  {feature}.service.ts
  {feature}.repository.ts   (if DB layer separated)
  {feature}.schema.ts       (validation schemas)
  {feature}.types.ts        (types/interfaces)

shared/
  logic/         (pure business functions)
  utils/         (non-business helpers: logger, telemetry)
  errors/        (typed error hierarchy)
  middleware/    (auth, validation, rate-limit, idempotency)
  context/       (request-scoped storage, e.g. AsyncLocalStorage)
```

---

## Error Handling

**Typed hierarchy → single global handler.**

```ts
// errors/index.ts
class AppError extends Error {
  constructor(public statusCode: number, message: string) { super(message); }
}
class NotFoundError extends AppError {
  constructor(msg = 'Not found') { super(404, msg); }
}
class ValidationError extends AppError {
  constructor(msg: string) { super(400, msg); }
}
class UnauthorizedError extends AppError {
  constructor(msg = 'Unauthorized') { super(401, msg); }
}
class ForbiddenError extends AppError {
  constructor(msg = 'Forbidden') { super(403, msg); }
}
class ConflictError extends AppError {
  constructor(msg: string) { super(409, msg); }
}
```

```ts
// middleware/errorHandler.ts (last middleware, always)
app.use((err, req, res, next) => {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({ error: err.message });
  }
  logger.error(err);
  res.status(500).json({ error: 'Internal server error' });
});
```

Never `res.status(404)` in a controller. Throw `NotFoundError` instead.

---

## Validation

**Validate at system boundaries only.** Trust internal code.

```
Boundaries: HTTP req body, query params, env vars, external API responses, file imports
Not boundaries: internal fn calls, service-to-service within same process
```

Use schema validation (Zod, Joi, Yup) as middleware — not inside controllers or services.

```
POST /users → validateBody(createUserSchema) → controller
```

---

## Configuration / Environment

Follow **12-Factor App** config rules:

1. All config comes from env vars — never hardcode
2. Validate all env vars at startup — fail fast if missing
3. Use defaults for optional tuneable values
4. Never commit `.env` — commit `.env.example`
5. Separate configs per environment (dev/test/prod)

```ts
// config/env.ts — validate once at startup
const env = z.object({
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  PORT: z.coerce.number().default(3000),
  LOG_LEVEL: z.enum(['debug','http','info','warn','error']).default('http'),
}).parse(process.env);
```

**Constants rule:** only true domain constants (never changes regardless of deployment). Everything tuneable = env var.

---

## Security Checklist

### Auth
- [ ] JWT: validate shape + expiry, never trust payload without verify
- [ ] Short access token (15m), longer refresh token (7d)
- [ ] Refresh token stored hashed in DB
- [ ] Lockout after N failed attempts (configurable via env)
- [ ] Role checked per route, not just "is logged in"

### API
- [ ] Rate limiting on auth endpoints
- [ ] Idempotency keys for non-GET mutation endpoints
- [ ] `helmet` for HTTP security headers
- [ ] CORS restricted to known origins
- [ ] Never return stack traces or internal error codes to client
- [ ] Never log passwords, tokens, or PII

### Data
- [ ] Soft delete — never hard delete user data
- [ ] Parameterised queries only — no string concatenation in SQL
- [ ] Sensitive data (passwords) hashed with bcrypt (cost ≥ 12)
- [ ] Anonymise before storing if identity not needed

### Input
- [ ] Validate and strip unknown fields on every req body
- [ ] Sanitise before rendering in HTML (XSS)
- [ ] Cap pagination: max limit + max page
- [ ] File upload: validate type + size server-side

---

## Database Patterns

### Schema
- Soft deletes: `deletedAt DateTime?` + filter `WHERE deletedAt IS NULL`
- Timestamps: `createdAt`, `updatedAt` on every table
- UUIDs or opaque IDs for public-facing identifiers
- Indexes on every foreign key + every column used in WHERE/ORDER BY
- Unique constraints at DB level, not only in code

### Query patterns
- N+1 rule: if looping and querying inside loop → use `include`/`JOIN`
- Never `SELECT *` in production queries — select only needed fields
- Transactions for multi-step mutations

### Migrations
- Never edit existing migrations — always create new ones
- Always test rollback
- Seed script separate from migration

---

## API Design

### REST conventions
```
GET    /resources          → list (paginated)
GET    /resources/:id      → single item
POST   /resources          → create
PATCH  /resources/:id      → partial update
DELETE /resources/:id      → soft delete
```

### Response shape
```json
// success list
{ "data": [...], "total": 100, "page": 1, "limit": 20 }

// success single
{ "data": { ... } }

// error
{ "error": "Human readable message" }
```

### Versioning
- Prefix all routes: `/api/v1/`
- Never break existing clients — add `v2` instead

### Idempotency Keys

Client sends a unique string (UUID) in `Idempotency-Key` header. Server uses it to replay the same response on retries without reprocessing.

**Flow:**
```
Request + Idempotency-Key
  ↓
Hash request body → check stored key matches body hash
  ↓ (key not seen before)
Process request → store { status, body, statusCode, bodyHash } in Redis with 24h TTL
  ↓ (key seen before, body hash matches)
Replay stored response — skip all processing
  ↓ (key seen before, body hash differs)
Return 400 — same key, different payload is a client bug
```

**Storage entry shape:**
```json
{
  "statusCode": 201,
  "body": { "data": { "id": "uuid" } },
  "bodyHash": "sha256-of-request-body"
}
```

**Rules:**
- Apply only to non-GET mutation endpoints (POST, PATCH, DELETE)
- Scope cache key per user: `userId:idempotencyKey` — prevents cross-user cache leakage
- Store only successful 2xx responses — failed requests should be retryable
- TTL: 24 hours is the industry standard
- Run a cleanup job for expired keys if not using Redis TTL auto-expiry
- Return `409 Conflict` if key is currently being processed (in-flight deduplication)

---

## Observability

### Logging levels
```
error  — system broken, needs immediate attention
warn   — unexpected but recoverable (wrong password, 4xx)
info   — state changes (login, record created)
http   — every request/response
debug  — detailed dev traces (never in prod)
```

### What to log
- Every auth event (success + failure + reason)
- Every state-changing operation (create/update/delete)
- Every scheduled job tick + result
- Startup + shutdown

### What NOT to log
- Passwords, tokens, secrets
- PII (email, phone) unless explicitly required + redacted
- Read operations (GET requests flood logs with noise)
- Internal fn calls (too granular)

### Structured logging
Always log as JSON in production. Always include:
- `timestamp`
- `level`
- `traceId` (per-request ID for correlation)
- `service` (which module)
- `event` (machine-readable event name)
- `userId`, `role` (when available)

---

## Testing Strategy

```
Unit tests      → shared/logic/, pure functions, fast, no DB
Integration     → service + real DB, tests full stack below HTTP
E2E             → browser/HTTP client + full running system
```

**Ratio (rough):** 70% unit / 20% integration / 10% E2E

### Rules
- Reseed DB before every integration/E2E run — known state always
- Never mock the DB in integration tests — defeats the purpose
- Unit test edge cases. Integration test happy path + critical errors. E2E test user journeys.
- Test file lives next to source: `auth.ts` → `auth.test.ts`

---

## Frontend Layers

```
pages/{role}/        # UI only — no direct API calls
hooks/use{Svc}.ts    # TanStack Query / SWR wrappers
api/{svc}.ts         # raw HTTP calls, one file per backend service
lib/queryKeys.ts     # ALL cache keys — never inline strings
store/               # global client state (auth, UI) — Zustand/Redux
```

### Frontend layer contracts

| Layer | Can call | Cannot |
|-------|----------|--------|
| Page | Hooks only | api/*.ts directly |
| Hook | api/*.ts + queryKeys | Business logic |
| api | HTTP client | State, hooks |
| store | Nothing external | — |

### Component rules
- One concern per component
- Data fetching in hooks, not components
- No `useEffect` for data fetching — use query library
- Shared utilities extracted to `lib/utils.ts` — never re-implement inline

---

## Middleware Order (Express / similar)

```
cors
helmet
body parser (json)
request logger + trace ID
rate limiter
─── per-route ───
authenticate
authorize (role check)
validate (Zod schema)
─── handler ───
controller
─── global ───
errorHandler   ← always last
```

---

## Graceful Shutdown

Never kill the process mid-request. Drain in-flight work first, then close resources in dependency order.

**Order:**
```
1. Stop accepting new requests   (server.close() / mark pod unready in k8s)
2. Wait for in-flight requests   (configurable timeout, e.g. 30s)
3. Stop background jobs          (schedulers, queue workers)
4. Close DB connections          (Prisma.$disconnect(), mongoose.disconnect())
5. Close Redis / message queues
6. Force exit after hard timeout (e.g. 60s) — never hang forever
```

**Node.js example:**
```ts
const shutdown = async (signal: string) => {
  logger.info({ event: 'shutdown:start', signal });

  // 1. Stop accepting new HTTP connections
  server.close(() => logger.info({ event: 'shutdown:http_closed' }));

  // 2. Wait for in-flight requests (30s timeout)
  await new Promise(resolve => setTimeout(resolve, 30_000));

  // 3. Stop schedulers / background workers
  scheduler.stop();

  // 4. Close DB
  await prisma.$disconnect();
  logger.info({ event: 'shutdown:db_closed' });

  // 5. Close Redis
  await redis.quit();
  logger.info({ event: 'shutdown:redis_closed' });

  process.exit(0);
};

// Hard timeout — never hang
setTimeout(() => {
  logger.error({ event: 'shutdown:timeout_forced' });
  process.exit(1);
}, 60_000).unref();

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT',  () => shutdown('SIGINT'));
```

**Why SIGTERM and SIGINT:**
- `SIGTERM` — sent by container orchestrators (k8s, Docker) on pod stop
- `SIGINT` — sent by Ctrl+C in terminal (dev)

**Never** call `process.exit()` directly in application code — only in the shutdown handler.

---

## New Feature Checklist

### Backend
- [ ] Created `{svc}.routes.ts`, `{svc}.controller.ts`, `{svc}.service.ts`
- [ ] Mounted routes in `app.ts`
- [ ] Validation schema defined, applied as middleware
- [ ] Error types used (not raw `res.status`)
- [ ] Reusable logic → `shared/logic/`
- [ ] New env var → validated in `config/env.ts` with default
- [ ] New DB columns → migration created
- [ ] Indexes added for new query patterns
- [ ] Unit tests for shared logic
- [ ] Integration test for happy path

### Frontend
- [ ] Cache key added to `queryKeys.ts`
- [ ] API fn added to `api/{svc}.ts`
- [ ] Query/mutation hook created in `hooks/`
- [ ] Page uses hook, not raw API call
- [ ] Error state handled in UI
- [ ] Loading state handled in UI

---

## Anti-Patterns (never do)

| Anti-pattern | Why bad | Fix |
|---|---|---|
| Logic in controller | Untestable, bloated | Move to service |
| DB query in controller | Bypasses service layer | Move to repository/service |
| Inline SQL strings | SQL injection risk | ORM or parameterised queries |
| `SELECT *` in queries | Loads sensitive fields, slow | Select only needed fields |
| Hardcoded config | Can't deploy to different env | Env var |
| Global error swallowed (`catch {}`) | Silent failures | Log + rethrow or handle |
| Auth check inside service | Duplicated, easy to miss | Middleware on route |
| `any` type in TypeScript | Defeats type safety | Explicit type or `unknown` |
| Inline cache keys (`['users', id]`) | Typos, no refactor safety | Central `queryKeys.ts` |
| Re-implementing shared util per file | Drift between impls | Extract to `lib/utils.ts` |
| Hard delete user records | Data loss, compliance issues | Soft delete |
| Secrets in git | Credential leak | `.env` + `.gitignore` |
