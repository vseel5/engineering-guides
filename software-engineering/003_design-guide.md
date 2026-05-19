# Software Design Guide

Sits between the **Ideation & Planning Guide** (requirements exist) and the **Scaffolding Guide** (folder structure follows design decisions). Do not start this guide until all 7 planning gates are passed.

Language/framework-agnostic. Running example: **Event booking system** (browse events, reserve seats, pay, receive confirmation).

---

## Why Security comes before API Design

Auth strategy (JWT vs session, token lifetimes) and authorization (roles × resources) directly shape the API surface — which endpoints exist, what headers are required, what each endpoint can return. You cannot finalize API contracts without the security model locked. Design the security model first, then design the API inside those constraints.

---

## 1. Sequence Diagrams

**Answers:** What happens step-by-step for every important flow — happy path, errors, and async?

Draw these before writing a single endpoint. They reveal missing steps, wrong ownership, and race conditions that are invisible in a bullet-point spec.

### Format (text-based)

```
Actor → System : Action
System → ExternalService : Call
ExternalService → System : Response
System → Actor : Result
```

### Happy Path — Event booking

```
Attendee → API         : POST /bookings { eventId, paymentMethod }
API      → Auth        : Verify JWT
Auth     → API         : { userId, role: 'ATTENDEE' }
API      → BookingSvc  : reserveSeat(eventId, userId)
BookingSvc → DB        : BEGIN TRANSACTION
BookingSvc → DB        : SELECT event WHERE id=eventId FOR UPDATE
DB         → BookingSvc: { capacity: 10, booked: 9 }
BookingSvc → DB        : INSERT booking (status: PENDING)
BookingSvc → DB        : UPDATE event SET booked = 10
BookingSvc → DB        : COMMIT
BookingSvc → API       : { bookingId, status: PENDING }
API      → Stripe      : POST /payment_intents { amount, currency }
Stripe   → API         : { clientSecret }
API      → Attendee    : 201 { bookingId, clientSecret }

-- Attendee completes payment in browser --

Stripe   → API         : POST /webhooks (payment_intent.succeeded)
API      → WebhookSvc  : handlePaymentSuccess(paymentIntentId)
WebhookSvc → DB        : UPDATE booking SET status = CONFIRMED
WebhookSvc → EmailSvc  : sendConfirmation(bookingId)
EmailSvc → Attendee    : Confirmation email with QR ticket
```

### Error Paths

Document every failure branch — not just the happy path.

**Validation failure (sync):**
```
Attendee → API        : POST /bookings { eventId: null }
API      → Attendee   : 400 { error: "eventId is required" }
-- No DB calls made; fail fast at the boundary --
```

**Auth failure:**
```
Attendee → API        : POST /bookings (expired JWT)
API      → Auth       : Verify JWT
Auth     → API        : TokenExpiredError
API      → Attendee   : 401 { error: "Token expired" }
```

**Capacity race condition:**
```
AttendeeA → BookingSvc : reserveSeat(eventId) -- last seat
AttendeeB → BookingSvc : reserveSeat(eventId) -- concurrent
BookingSvc → DB        : SELECT ... FOR UPDATE  -- AttendeeA acquires lock
BookingSvc → DB        : UPDATE booked = capacity -- success for A
DB         → BookingSvc: Lock released
BookingSvc → DB        : SELECT ... FOR UPDATE  -- AttendeeB acquires lock
DB         → BookingSvc: { capacity = booked }  -- full
BookingSvc → API       : EventFullError
API        → AttendeeB : 409 { error: "Event is full", waitlistAvailable: true }
```

**Payment failure (async):**
```
Stripe → API         : POST /webhooks (payment_intent.payment_failed)
API    → WebhookSvc  : handlePaymentFailure(paymentIntentId)
WebhookSvc → DB      : UPDATE booking SET status = FAILED
WebhookSvc → DB      : UPDATE event SET booked = booked - 1  -- release seat
WebhookSvc → EmailSvc: sendPaymentFailureNotice(bookingId)
WebhookSvc → API     : 200 OK  -- always ACK webhooks
```

### Async Flows

Any flow where the response is not immediate needs its own diagram.

**Webhook receipt pattern:**
```
ExternalService → API : POST /webhooks/stripe
API → WebhookMiddleware : Verify signature (HMAC)
WebhookMiddleware → API : Verified / 400 invalid signature
API → Queue            : Enqueue job { event, payload, receivedAt }
API → ExternalService  : 200 OK  -- ACK immediately, always
Queue → Worker         : Dequeue + process
Worker → DB            : Persist state change
Worker → EmailSvc      : Notify user
```

**Background job pattern:**
```
Scheduler → JobRunner  : Trigger "expire-pending-bookings" (every 15 min)
JobRunner → DB         : SELECT bookings WHERE status=PENDING AND created < now()-30min
DB        → JobRunner  : [bookingId1, bookingId2]
JobRunner → DB         : UPDATE bookings SET status=EXPIRED WHERE id IN (...)
JobRunner → DB         : UPDATE events SET booked = booked - count WHERE ...
JobRunner → Logger     : { event: 'bookings:expired', count: 2 }
```

### Checklist

- [ ] Happy path diagrammed end-to-end including external services
- [ ] Every error path that reaches a different code branch has its own diagram
- [ ] All async flows (webhooks, jobs, queues) diagrammed separately
- [ ] Race conditions identified and resolution documented
- [ ] Every external call has a timeout + failure path

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Only diagram the happy path | Error paths are where the real complexity lives |
| Skip async diagrams ("it's just a webhook") | Async flows hide ordering bugs and retry storms |
| Diagrams at system level only | Missing step-by-step reveals missing ownership |
| Build first, diagram after | Diagram exists to find problems before they're baked in |

---

## 2. Security Design

**Answers:** How is identity established, what can each role do, where is sensitive data, and how is abuse prevented?

Lock this down before designing the API. Auth strategy and roles shape every endpoint.

### Authentication Method

Pick one per project and commit. Do not leave as TBD.

| Method | When to use | When to avoid |
|---|---|---|
| **JWT (stateless)** | APIs consumed by SPAs, mobile, other services | When you need instant revocation (e.g. financial, healthcare) |
| **Session (server-side)** | Traditional web apps, instant revocation needed | Horizontal scaling without sticky sessions or Redis |
| **OAuth 2.0 / OIDC** | Third-party login (Google, GitHub), B2B federation | Simple apps with no identity provider |
| **API Keys** | Machine-to-machine, developer APIs | End-user authentication |

**Example — Event booking system:**
```
Strategy: JWT (access + refresh token pair)
Access token:  15 minutes (short — reduces exposure if leaked)
Refresh token: 7 days    (stored hashed in DB — enables revocation)
Signing:       RS256      (asymmetric — public key shareable for verification)
Storage:       access → memory (JS var); refresh → httpOnly cookie
```

**JWT payload — include only what's needed:**
```json
{
  "sub": "usr_abc123",
  "role": "ATTENDEE",
  "orgId": "org_xyz",
  "iat": 1700000000,
  "exp": 1700000900
}
```

Never put PII (email, name, phone) in the JWT payload — it is base64-encoded, not encrypted.

### Authorization Matrix

Define what each role can do for every resource before writing routes.

```
Legend: C = Create, R = Read, O = Read Own, U = Update, D = Delete, — = No access
```

| Resource | Public | Attendee | Organiser | Admin |
|---|---|---|---|---|
| Events (list/view) | R | R | R | R |
| Events (manage) | — | — | C R U D | C R U D |
| Bookings (own) | — | C O | — | R |
| Bookings (all) | — | — | R (own events) | R |
| Users | — | O (self) | O (self) | C R U D |
| Webhooks | — | — | — | R |
| Analytics | — | — | R (own events) | R |

Rules:
- Always check role server-side — never trust client-sent role claims
- Scope resources to owner: organiser can only see bookings for their own events
- Prefer allowlist (what is permitted) over blocklist (what is forbidden)

### Token Lifecycle

```
Login → issue access token (15m) + refresh token (7d, httpOnly cookie)
         ↓
Request → verify access token → proceed
         ↓ (expired)
Silent refresh → POST /auth/refresh → new access token
         ↓ (refresh expired or revoked)
Logout → force re-login
         ↓
Explicit logout → DELETE /auth/session → invalidate refresh token in DB
```

### PII Inventory

List every piece of personal data, where it lives, who can see it, and how long it's kept.

| Data | Table / Field | Who can read | Retention | Notes |
|---|---|---|---|---|
| Email | `users.email` | Self, Admin | Account lifetime | Encrypted at rest |
| Name | `users.name` | Self, Organiser (booking list) | Account lifetime | |
| Payment method | Not stored | — | Never | Stripe owns this |
| Payment status | `bookings.payment_ref` | Self, Organiser, Admin | 7 years (legal) | Ref only, no card data |
| IP address | `audit_log.ip` | Admin | 90 days | Needed for abuse detection |
| QR ticket | `bookings.qr_hash` | Self, Scanner | Event date + 30 days | Hash only |

Rules:
- Never store what you don't need
- Never log PII (email, phone, name) in application logs
- Pseudonymise or anonymise data used for analytics

### Rate Limiting

Define per endpoint, not globally. Blanket rate limits are too loose for auth endpoints and too strict for read endpoints.

| Endpoint | Limit | Window | Scope |
|---|---|---|---|
| `POST /auth/login` | 5 requests | 15 minutes | Per IP |
| `POST /auth/refresh` | 10 requests | 1 minute | Per user |
| `POST /bookings` | 20 requests | 1 minute | Per user |
| `GET /events` | 300 requests | 1 minute | Per IP |
| `POST /webhooks/*` | 1000 requests | 1 minute | Per IP |

Return `429 Too Many Requests` with `Retry-After` header.

### Security Checklist

- [ ] Authentication method chosen and documented (not TBD)
- [ ] Token storage strategy defined (memory / httpOnly cookie / secure storage)
- [ ] Authorization matrix covers every role × resource combination
- [ ] PII inventory complete with retention periods
- [ ] Rate limits defined per endpoint with scope (per-IP vs per-user)
- [ ] Refresh token invalidation strategy documented (DB hash comparison)
- [ ] No PII in JWT payload
- [ ] Webhook signature verification strategy defined

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Auth strategy left as TBD | Every layer that needs to check auth is unblocked on this |
| Authorization checked in service layer only | Inconsistent; easy to miss; leaks data across role boundaries |
| Storing refresh tokens as plaintext | DB breach → all sessions compromised |
| Same rate limit for all endpoints | Auth endpoints under-protected; read endpoints over-restricted |
| PII in JWT | Encoded, not encrypted — visible to anyone who base64-decodes the token |

---

## 3. API Design

**Answers:** What are the exact endpoints, request shapes, response shapes, and status codes?

Design the API contract before writing implementation. This document is the agreement between frontend and backend — change it only with both sides present.

### REST Conventions

**Resource naming:**
```
Plural nouns for collections:   /events, /bookings, /users
Singular noun for item:         /events/:id
Nested for ownership:           /events/:id/bookings  (bookings that belong to an event)
Avoid verbs in URLs:            /bookings/:id/cancel  ← wrong
Use PATCH + status field:       PATCH /bookings/:id { status: 'CANCELLED' }  ← right
```

**HTTP methods:**
```
GET    → read, no side effects, safe to retry
POST   → create, not idempotent (use Idempotency-Key header)
PATCH  → partial update (only send changed fields)
PUT    → full replacement (rare; use only when replacing the entire resource)
DELETE → remove (soft delete preferred; return 204 No Content)
```

**Status codes — use the right one:**

| Code | When |
|---|---|
| 200 OK | Successful GET, PATCH |
| 201 Created | Successful POST that created a resource |
| 204 No Content | Successful DELETE or action with no response body |
| 400 Bad Request | Validation error — client sent invalid data |
| 401 Unauthorized | Missing or invalid auth token |
| 403 Forbidden | Authenticated but not authorized for this resource |
| 404 Not Found | Resource does not exist |
| 409 Conflict | State conflict (duplicate, capacity full, wrong status) |
| 422 Unprocessable | Request is valid JSON but semantically wrong (e.g. end before start) |
| 429 Too Many Requests | Rate limit hit |
| 500 Internal Server Error | Unexpected server failure |

### Request / Response Shapes

**Consistent envelope across all endpoints:**

```json
// Success — single resource
{
  "data": { "id": "bkg_abc123", "status": "CONFIRMED" }
}

// Success — collection
{
  "data": [ { "id": "evt_001" }, { "id": "evt_002" } ],
  "meta": { "total": 42, "page": 1, "limit": 20, "hasNext": true }
}

// Error
{
  "error": "Event is full",
  "code": "EVENT_FULL",
  "details": { "waitlistAvailable": true }
}
```

**Example endpoints — Event booking system:**

```
POST /api/v1/bookings
  Request:
    Headers: Authorization: Bearer <token>
             Idempotency-Key: <uuid>
    Body: {
      "eventId": "evt_001",
      "attendeeCount": 2
    }
  Response 201:
    {
      "data": {
        "id": "bkg_abc123",
        "status": "PENDING",
        "clientSecret": "pi_xxx_secret_yyy"
      }
    }
  Response 409:
    {
      "error": "Event is full",
      "code": "EVENT_FULL",
      "details": { "waitlistAvailable": true }
    }

GET /api/v1/events?status=upcoming&page=1&limit=20&sort=startsAt:asc
  Response 200:
    {
      "data": [
        {
          "id": "evt_001",
          "title": "React Workshop",
          "startsAt": "2025-06-01T09:00:00Z",
          "capacity": 50,
          "available": 12
        }
      ],
      "meta": { "total": 8, "page": 1, "limit": 20, "hasNext": false }
    }
```

### Pagination, Filtering, Sorting

**Always use cursor or offset — never `page * limit` without a cap:**

```
# Offset-based (simple, less consistent under inserts)
GET /events?page=2&limit=20

# Cursor-based (consistent, better for real-time data)
GET /events?after=evt_050&limit=20
Response includes: { "meta": { "nextCursor": "evt_070" } }
```

**Filtering — use query params, never request body for GET:**
```
GET /events?status=upcoming&organiserId=org_001&from=2025-06-01&to=2025-12-31
```

**Sorting:**
```
GET /events?sort=startsAt:asc,title:desc
```

**Caps (always enforce server-side):**
```
Max limit:    100  (never allow unlimited)
Max page:     10,000
Default limit: 20
```

### OpenAPI Specification (minimal)

Write at minimum one OpenAPI spec block per endpoint. This becomes the contract.

```yaml
# openapi.yaml
openapi: 3.1.0
info:
  title: Event Booking API
  version: 1.0.0

paths:
  /bookings:
    post:
      summary: Reserve seats for an event
      security:
        - bearerAuth: []
      parameters:
        - in: header
          name: Idempotency-Key
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [eventId, attendeeCount]
              properties:
                eventId:
                  type: string
                attendeeCount:
                  type: integer
                  minimum: 1
                  maximum: 10
      responses:
        '201':
          description: Booking created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BookingResponse'
        '409':
          description: Event is full
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
```

### API Design Checklist

- [ ] Every endpoint has: method, URL, auth requirement, request shape, response shape, error responses
- [ ] All error responses use the standard envelope with `code` field
- [ ] Pagination params defined with server-enforced caps
- [ ] Idempotency-Key required on all POST/PATCH mutations
- [ ] OpenAPI spec started (at minimum one block per endpoint)
- [ ] No verbs in resource URLs
- [ ] Breaking changes version the API (`/v2/`) — never modify v1 contract

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Different response shapes per endpoint | Frontend builds per-endpoint parsing; inconsistency causes bugs |
| Verbs in URLs (`/cancelBooking`) | Not REST; harder to document and discover |
| No pagination on list endpoints | Works in dev with 10 records; crashes in prod with 100k |
| 200 for every response including errors | Clients cannot detect failure without parsing body |
| API contract "in devs' heads" | Frontend and backend drift; integration fails at the last moment |

---

## 4. Database Design

**Answers:** What data is stored, how is it structured, and how does it stay consistent?

### ERD (text format)

```
[User]
  id          PK
  email       UNIQUE NOT NULL
  name        NOT NULL
  role        ENUM(ATTENDEE, ORGANISER, ADMIN)
  passwordHash NOT NULL
  createdAt
  deletedAt   (soft delete)

[Event]
  id          PK
  organiserId FK → User.id
  title       NOT NULL
  description
  capacity    INT NOT NULL
  booked      INT NOT NULL DEFAULT 0
  startsAt    TIMESTAMPTZ NOT NULL
  endsAt      TIMESTAMPTZ NOT NULL
  status      ENUM(DRAFT, PUBLISHED, CANCELLED, COMPLETED)
  createdAt
  deletedAt

[Booking]
  id          PK
  eventId     FK → Event.id
  attendeeId  FK → User.id
  status      ENUM(PENDING, CONFIRMED, CANCELLED, EXPIRED, FAILED)
  attendeeCount INT NOT NULL DEFAULT 1
  paymentRef  VARCHAR (Stripe payment intent ID)
  idempotencyKey VARCHAR UNIQUE
  createdAt
  updatedAt

[Waitlist]
  id          PK
  eventId     FK → Event.id
  attendeeId  FK → User.id
  position    INT NOT NULL
  createdAt

[AuditLog]
  id          PK
  actorId     FK → User.id (nullable — system events)
  action      VARCHAR NOT NULL
  resourceType VARCHAR NOT NULL
  resourceId  VARCHAR NOT NULL
  meta        JSONB
  ip          VARCHAR
  createdAt

Relationships:
  User    1──* Event    (organiser creates events)
  User    1──* Booking  (attendee makes bookings)
  Event   1──* Booking
  Event   1──* Waitlist
  User    1──* Waitlist
```

### Schema Definition

Write the full schema before implementation. Use the DB's native types, not generic ones.

```sql
CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email         VARCHAR(255) UNIQUE NOT NULL,
  name          VARCHAR(255) NOT NULL,
  role          VARCHAR(20)  NOT NULL CHECK (role IN ('ATTENDEE','ORGANISER','ADMIN')),
  password_hash VARCHAR(255) NOT NULL,
  created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  deleted_at    TIMESTAMPTZ
);

CREATE TABLE events (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organiser_id UUID         NOT NULL REFERENCES users(id),
  title        VARCHAR(255) NOT NULL,
  description  TEXT,
  capacity     INTEGER      NOT NULL CHECK (capacity > 0),
  booked       INTEGER      NOT NULL DEFAULT 0 CHECK (booked >= 0),
  starts_at    TIMESTAMPTZ  NOT NULL,
  ends_at      TIMESTAMPTZ  NOT NULL CHECK (ends_at > starts_at),
  status       VARCHAR(20)  NOT NULL DEFAULT 'DRAFT',
  created_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  deleted_at   TIMESTAMPTZ,
  CONSTRAINT booked_lte_capacity CHECK (booked <= capacity)
);

CREATE TABLE bookings (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_id         UUID         NOT NULL REFERENCES events(id),
  attendee_id      UUID         NOT NULL REFERENCES users(id),
  status           VARCHAR(20)  NOT NULL DEFAULT 'PENDING',
  attendee_count   INTEGER      NOT NULL DEFAULT 1 CHECK (attendee_count > 0),
  payment_ref      VARCHAR(255),
  idempotency_key  VARCHAR(255) UNIQUE,
  created_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  updated_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
```

### Indexes

Define indexes alongside the schema. Every omitted index is a future full table scan.

```sql
-- users
CREATE INDEX idx_users_email      ON users(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_role       ON users(role)  WHERE deleted_at IS NULL;

-- events
CREATE INDEX idx_events_organiser ON events(organiser_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_events_status    ON events(status, starts_at) WHERE deleted_at IS NULL;

-- bookings
CREATE INDEX idx_bookings_event   ON bookings(event_id, status);
CREATE INDEX idx_bookings_attendee ON bookings(attendee_id, status);
CREATE INDEX idx_bookings_payment ON bookings(payment_ref) WHERE payment_ref IS NOT NULL;
CREATE INDEX idx_bookings_pending ON bookings(created_at) WHERE status = 'PENDING';
```

**Index decision rule:**
- Every foreign key → index it
- Every column in a `WHERE` clause → candidate for index
- Partial indexes (`WHERE status = 'PENDING'`) for scheduler queries that target a subset
- Composite indexes: column order matters — most selective first

### Soft Delete Pattern

```sql
-- Never DELETE — set deleted_at
UPDATE users SET deleted_at = NOW() WHERE id = $1;

-- Always filter in queries
SELECT * FROM events WHERE deleted_at IS NULL;

-- Unique constraints need partial indexes to ignore deleted rows
CREATE UNIQUE INDEX idx_events_title_active
  ON events(organiser_id, title)
  WHERE deleted_at IS NULL;
```

### Migration Strategy

```
Rules:
1. Never edit an existing migration file
2. Every schema change = new migration file
3. Migrations are append-only and immutable
4. Each migration is atomic (wrapped in a transaction)
5. Test rollback before merging
6. Seed data is a separate script — not part of migrations

Naming convention:
  YYYYMMDDHHMMSS_description.sql
  Example: 20250601120000_add_waitlist_table.sql

Migration file structure:
  -- Up
  CREATE TABLE waitlist (...);

  -- Down (rollback)
  DROP TABLE waitlist;
```

### Database Design Checklist

- [ ] ERD covers all entities and relationships
- [ ] Primary keys defined (UUID preferred for public-facing IDs)
- [ ] Foreign key constraints defined on all relationships
- [ ] Check constraints enforce business rules at DB level (capacity > 0, ends_at > starts_at)
- [ ] Indexes defined for every FK and every WHERE/ORDER BY column
- [ ] Soft delete pattern applied to all user-facing entities
- [ ] `createdAt` / `updatedAt` on every table
- [ ] Unique constraints at DB level (not only in code)
- [ ] Migration strategy documented — no editing existing files

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| No check constraints | Invalid data enters DB; caught only in application code (too late) |
| Missing indexes on FKs | Every join is a full table scan under load |
| Hard deletes | Data loss; breaks audit trails; violates compliance requirements |
| Business rules only in application code | Direct DB access or a bug bypasses them; data becomes inconsistent |
| Editing existing migrations | Other devs run different DB states; impossible to reproduce prod |
| `status` stored as INT enum | Magic numbers; impossible to read in SQL queries or logs |

---

## 5. Integration Design

**Answers:** What external systems do we call, what do they call us with, and how do we handle failure on both sides?

### External API Contracts

Document every external service call before implementing it.

```
Service: Stripe — Payment Intents API
Endpoint: POST https://api.stripe.com/v1/payment_intents
Auth: Bearer sk_live_xxxx (API key, stored in env var STRIPE_SECRET_KEY)
Called when: Booking created, before attendee confirms payment
Timeout: 10 seconds
Retry: 3 attempts with exponential backoff (1s, 2s, 4s) on 5xx only
On failure: Roll back booking reservation; return 503 to client

Request:
  amount:   integer (pence/cents — never floats)
  currency: string  (e.g. "gbp")
  metadata: { bookingId: "bkg_abc123" }

Response (success):
  { "id": "pi_xxx", "client_secret": "pi_xxx_secret_yyy", "status": "requires_payment_method" }

Response (failure):
  { "error": { "code": "card_declined", "message": "..." } }
```

**External call rules:**
- Every external call has an explicit timeout — never rely on the provider's default
- Never call an external service inside a DB transaction — holds the transaction open
- Store the external reference ID (`paymentRef`) before calling — enables idempotent retry
- Circuit breaker pattern for high-volume integrations: if >5 failures in 60s, stop calling and return a fast failure

### Webhook Handling

Webhooks are external systems calling you. Treat them as untrusted until signature is verified.

**Receipt pattern:**

```
1. Verify signature (HMAC or provider-specific)
   → 400 immediately if invalid — do not process
2. Acknowledge immediately (return 200)
   → Provider will retry if you take > 5-30s (provider-dependent)
3. Enqueue the payload for async processing
   → Never do heavy work in the webhook handler
4. Process idempotently
   → Same event can arrive multiple times; check if already processed
```

**Stripe webhook example:**

```ts
// Verify first — before any processing
const sig = req.headers['stripe-signature'];
const event = stripe.webhooks.constructEvent(req.rawBody, sig, STRIPE_WEBHOOK_SECRET);

// Idempotency check
const alreadyProcessed = await db.webhookEvent.findUnique({ where: { stripeId: event.id } });
if (alreadyProcessed) return res.sendStatus(200);  // replay — skip

// Mark as received before processing (prevents double-processing on crash)
await db.webhookEvent.create({ data: { stripeId: event.id, type: event.type } });

// Enqueue — do not process inline
await queue.add('stripe-webhook', { event });
res.sendStatus(200);  // ACK
```

**Retry handling:**
```
Assume every webhook can be delivered 2+ times.
Use the event's ID as the idempotency key.
Store processed event IDs with a 72h TTL (matching provider retry window).
```

**Webhook signature verification by provider:**
| Provider | Header | Method |
|---|---|---|
| Stripe | `Stripe-Signature` | HMAC-SHA256 of raw body + timestamp |
| GitHub | `X-Hub-Signature-256` | HMAC-SHA256 of raw body |
| SendGrid | `X-Twilio-Email-Event-Webhook-Signature` | ECDSA |
| Generic | `X-Webhook-Secret` | Compare bearer token (less secure) |

### Message Queue Schemas

If using a message queue (RabbitMQ, SQS, Kafka, BullMQ), define the message schema before writing producers or consumers.

```ts
// Queue: booking-notifications
// Producer: booking service (on status change)
// Consumer: notification service

interface BookingStatusChangedMessage {
  version: 1;                              // schema version — always include
  eventId: string;
  bookingId: string;
  attendeeId: string;
  newStatus: 'CONFIRMED' | 'CANCELLED' | 'EXPIRED' | 'FAILED';
  occurredAt: string;                      // ISO 8601
}

// Queue: stripe-webhooks
// Producer: webhook endpoint
// Consumer: payment processing worker

interface StripeWebhookMessage {
  version: 1;
  stripeEventId: string;
  type: string;
  payload: Record<string, unknown>;
  receivedAt: string;
}
```

**Queue rules:**
- Always include `version` — enables backward-compatible schema evolution
- Consumer must handle unknown `version` values gracefully (log + dead-letter, do not crash)
- Poison message → dead letter queue after N retries; alert on DLQ depth
- Message schema changes = new version field, not breaking change to existing fields

### Integration Design Checklist

- [ ] Every external API call has: endpoint, auth method, timeout, retry strategy, failure handling
- [ ] Webhook signature verification method documented per provider
- [ ] Idempotency strategy for webhook processing defined
- [ ] Message queue schemas versioned and documented
- [ ] No external calls inside DB transactions
- [ ] External reference IDs stored before making calls (enables idempotent retry)
- [ ] Dead letter queue defined for failed messages

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| No timeout on external calls | One slow provider hangs threads; cascading failure |
| Processing webhooks synchronously | Provider retries after timeout; event processed twice |
| No signature verification on webhooks | Any actor can send fake events |
| External calls inside transactions | Transaction held open for network round-trip; connection pool exhaustion |
| No DLQ for failed queue messages | Failed messages silently dropped; data loss |
| Assuming webhooks arrive exactly once | They don't — provider retries on non-2xx; always process idempotently |

---

## 6. Error & Edge Case Design

**Answers:** What can go wrong, what error does the client receive, and what does the system do to recover?

### Error Catalog

Define every error code before writing a controller. This is the API contract for failures.

| Code | HTTP Status | Message (client-facing) | When it occurs |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | "Invalid request: {field} {reason}" | Schema validation fails |
| `UNAUTHORIZED` | 401 | "Authentication required" | No token or token invalid |
| `TOKEN_EXPIRED` | 401 | "Token expired" | JWT past exp claim |
| `FORBIDDEN` | 403 | "You do not have access to this resource" | Role insufficient |
| `NOT_FOUND` | 404 | "{Resource} not found" | ID does not exist or soft-deleted |
| `EVENT_FULL` | 409 | "Event is full" | booked === capacity at reserve time |
| `BOOKING_EXISTS` | 409 | "You already have a booking for this event" | Duplicate booking attempt |
| `INVALID_STATUS_TRANSITION` | 409 | "Cannot cancel a confirmed booking after event start" | State machine violation |
| `PAYMENT_FAILED` | 402 | "Payment could not be processed" | Stripe payment_intent.failed |
| `RATE_LIMITED` | 429 | "Too many requests. Retry after {seconds}s" | Rate limit hit |
| `EXTERNAL_SERVICE_ERROR` | 503 | "Service temporarily unavailable" | Stripe/email provider down |
| `INTERNAL_ERROR` | 500 | "An unexpected error occurred" | Unhandled exception |

**Client error shape (always):**
```json
{
  "error": "Event is full",
  "code": "EVENT_FULL",
  "details": { "waitlistAvailable": true }
}
```

Never expose: stack traces, internal error codes (e.g. `P2002`), DB field names, server paths.

### State Machine Design

For any entity with a `status` field, define the valid transitions. Invalid transitions return `INVALID_STATUS_TRANSITION`.

```
Booking status transitions:

  PENDING ──(payment confirmed)──→ CONFIRMED
  PENDING ──(payment failed)─────→ FAILED
  PENDING ──(30min timeout)──────→ EXPIRED
  CONFIRMED ──(before event)─────→ CANCELLED
  CANCELLED ─────────────────────→ (terminal)
  EXPIRED ───────────────────────→ (terminal)
  FAILED ────────────────────────→ (terminal)

  Any other transition = INVALID_STATUS_TRANSITION (409)
```

```
Event status transitions:

  DRAFT ──(publish)──→ PUBLISHED
  PUBLISHED ──(cancel)──→ CANCELLED
  PUBLISHED ──(event ends)──→ COMPLETED
  CANCELLED ──→ (terminal)
  COMPLETED ──→ (terminal)
```

### Retry Strategy

| Error type | Retryable? | Strategy | Max attempts |
|---|---|---|---|
| Network timeout (external call) | Yes | Exponential backoff: 1s, 2s, 4s | 3 |
| 429 Too Many Requests | Yes | Honour `Retry-After` header | 3 |
| 5xx from external service | Yes | Exponential backoff | 3 |
| 4xx from external service | No | Client error — do not retry | 1 |
| DB connection error | Yes | Immediate retry once, then fail | 2 |
| DB constraint violation | No | Logic error — do not retry | 1 |
| Webhook delivery failure | Yes | Provider handles retry | — |
| Queue job failure | Yes | BullMQ/SQS retry with backoff | 5 |

**Exponential backoff formula:**
```
delay = base * (2 ^ attempt) + jitter
base = 1000ms, jitter = random(0, 500ms)

attempt 1: ~1000ms
attempt 2: ~2000ms
attempt 3: ~4000ms
```

Jitter prevents retry storms (all clients retrying at the same moment).

### Dead Letter Queue

After max retries, messages go to a DLQ.

```
DLQ handling:
1. Alert on DLQ depth > 0 (page on-call if critical)
2. Log the full message payload + error reason
3. Never auto-retry from DLQ without investigation
4. Provide a manual replay script for operators
5. DLQ messages retained for 14 days minimum
```

### Edge Cases — Event booking system

Document edge cases explicitly so they are designed, not discovered in production.

| Scenario | Expected behaviour |
|---|---|
| Two users book the last seat simultaneously | One succeeds (row-level lock), one gets `EVENT_FULL` |
| Payment succeeds but webhook delayed 1 hour | Booking stays PENDING until webhook arrives; attendee can check status |
| Attendee books 2 seats, cancels 1 | Partially cancel not supported in v1 — cancel full booking only |
| Organiser cancels event with 50 confirmed bookings | All bookings → CANCELLED; refund jobs enqueued; emails sent |
| Stripe webhook received twice for same payment | Idempotency key prevents double-confirmation |
| Booking expires but payment processes late | Payment captured; create new booking in CONFIRMED state; log anomaly |
| User hard-deletes their account | Soft delete only; bookings retained for audit; email anonymised |

### Error Design Checklist

- [ ] Error catalog complete: every error has a code, HTTP status, message, and trigger condition
- [ ] State machines defined for all status fields — invalid transitions handled
- [ ] Retry strategy documented per error type (retryable vs non-retryable)
- [ ] DLQ defined for queue-based jobs
- [ ] Edge cases documented for every race condition identified in sequence diagrams
- [ ] No stack traces or internal details in error responses

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| 500 for everything unexpected | Client cannot distinguish retriable from fatal |
| Exposing Prisma/DB error codes to client | Schema enumeration — security risk |
| Retrying 4xx errors | Client error — retrying will always fail; wastes resources |
| No jitter in backoff | Retry storm — all clients hit the service at the same moment |
| No DLQ | Failed jobs silently dropped; data loss; impossible to recover |
| State machine only in code | Inconsistent enforcement; direct DB writes bypass it |

---

## 7. Design Review Checklist

**Answers:** Is the design complete enough to hand to developers without creating ambiguity?

Run this before any code is written.

### Author Pre-Review Checklist

Complete this yourself before requesting team review:

**Completeness:**
- [ ] Sequence diagrams cover every user-facing flow and its error paths
- [ ] Security model defined: auth method, role matrix, PII inventory, rate limits
- [ ] Every endpoint has: method, URL, request shape, response shape, all error codes
- [ ] Database schema: all tables, columns, types, constraints, indexes
- [ ] Every external service: endpoint, auth, timeout, retry, failure path
- [ ] Error catalog complete with codes and HTTP statuses
- [ ] State machines drawn for all status fields

**Consistency:**
- [ ] Auth requirement consistent between sequence diagrams and API design
- [ ] Error codes in API design match error catalog
- [ ] DB schema supports every field returned in API responses
- [ ] No endpoint returns a field that doesn't exist in the schema

**Gaps:**
- [ ] No TBDs in auth, DB, or API sections
- [ ] Every edge case from sequence diagrams has a documented handling strategy
- [ ] Pagination defined for every list endpoint

### Team Review Questions

Ask these in the design review meeting:

```
1. Can you trace the full booking flow through the sequence diagram without asking a question?
2. Which role can access which resource — can you answer that from the auth matrix alone?
3. If Stripe goes down mid-booking, what does the attendee see? What is the system state?
4. What happens if two attendees click "book" on the last seat at the same time?
5. Which endpoints require an Idempotency-Key? Where is that enforced?
6. What happens to a booking if the attendee's account is soft-deleted?
7. Can you write the acceptance test for "booking confirmed" from this design alone?
8. What is the riskiest assumption in this design?
```

### Sign-Off Criteria

The design is ready for implementation when:

- [ ] All section checklists passed
- [ ] Author pre-review checklist complete
- [ ] Design review meeting held; all team review questions answered
- [ ] No open questions remain (questions → decisions, not "we'll figure it out")
- [ ] OpenAPI spec committed to the repository
- [ ] Design document linked from the project README
- [ ] All stakeholders who raised concerns in review have signed off

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Design review = author presenting, others nodding | Problems discovered in implementation, not review |
| Reviewing for style instead of gaps | Misses missing error handling, wrong assumptions |
| "We'll figure out the edge cases in code" | Edge cases are 80% of the implementation complexity |
| Sign-off without reading the document | Meaningless sign-off; accountability gap at launch |
| Skipping review for "small" features | Small features have the same failure modes as large ones |

---

## 8. How This Feeds into Scaffolding Guide

**Answers:** Which design decision maps to which folder, file, or layer in the implementation?

Every design decision in this guide has a direct counterpart in the Scaffolding Guide's folder structure and the Testing Guide's test placement. This mapping ensures nothing is implemented in the wrong place.

| Design decision (this guide) | Scaffolding location |
|---|---|
| Sequence diagram flow → service boundary | One service per domain: `services/booking/`, `services/event/` |
| Sequence diagram step → fn call | Controller → service → repository chain |
| Security: Auth middleware | `shared/middleware/auth.ts` |
| Security: Role check | `shared/middleware/roles.ts` — applied per route |
| Security: Rate limiting | `shared/middleware/rateLimiter.ts` — applied per route |
| API: Endpoint definition | `{svc}.routes.ts` — route + middleware only |
| API: Request parsing + response | `{svc}.controller.ts` |
| API: Business logic | `{svc}.service.ts` |
| API: OpenAPI spec | `docs/openapi.yaml` or `src/docs/` |
| Database: Schema | `prisma/schema.prisma` or `db/migrations/` |
| Database: Queries | `{svc}.repository.ts` or service layer (if no repo layer) |
| Integration: External API calls | `{svc}.service.ts` or `shared/clients/{provider}.ts` |
| Integration: Webhook handler | `{svc}.routes.ts` → `{svc}.controller.ts` → queue |
| Integration: Queue producers | `{svc}.service.ts` — after state change |
| Integration: Queue consumers | `workers/{jobName}.ts` |
| Error catalog | `shared/errors/index.ts` — typed error classes |
| Error handling | `shared/middleware/errorHandler.ts` — global, last |
| State machine | `shared/logic/{entity}StateMachine.ts` — pure fn |
| Retry logic | `shared/utils/retry.ts` — pure fn, used in services |
| Idempotency | `shared/middleware/idempotency.ts` — applied per route |

### Folder skeleton derived from design

```
src/
  services/
    booking/
      booking.routes.ts       ← API endpoints from API Design section
      booking.controller.ts   ← parses req, calls service
      booking.service.ts      ← business logic from sequence diagrams
      booking.repository.ts   ← DB queries from Database Design section
      booking.schema.ts       ← Zod schemas from API request shapes
      booking.types.ts        ← TypeScript types from API response shapes
    event/
      event.routes.ts
      event.controller.ts
      event.service.ts
      event.repository.ts
  shared/
    middleware/
      auth.ts                 ← JWT verification from Security Design
      roles.ts                ← Authorization matrix from Security Design
      rateLimiter.ts          ← Rate limits from Security Design
      idempotency.ts          ← Idempotency rules from API Design
      errorHandler.ts         ← Error catalog from Error Design
    errors/
      index.ts                ← Error classes from Error Catalog
    logic/
      bookingStateMachine.ts  ← State machine from Error Design
      eventStateMachine.ts
    clients/
      stripe.ts               ← External API from Integration Design
      email.ts
    utils/
      retry.ts                ← Retry strategy from Error Design
  workers/
    stripeWebhook.worker.ts   ← Webhook consumer from Integration Design
    expireBookings.worker.ts  ← Background job from Sequence Diagrams
  config/
    env.ts                    ← All env vars validated at startup
  docs/
    openapi.yaml              ← OpenAPI spec from API Design
```

### Design → Scaffolding checklist

- [ ] Every sequence diagram flow maps to a service file
- [ ] Every auth check from the security matrix maps to a middleware on a route
- [ ] Every DB table maps to a repository method
- [ ] Every external service call maps to a `shared/clients/` file
- [ ] Every queue consumer maps to a `workers/` file
- [ ] Every error class in the catalog maps to `shared/errors/index.ts`
- [ ] Every state machine maps to a pure function in `shared/logic/`

When the folder skeleton above is created and the checklist is complete, open the Scaffolding Guide and create every file listed before writing business logic. Then open the Testing Guide and write the first test before the first line of production code.