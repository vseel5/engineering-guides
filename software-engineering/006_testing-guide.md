# Software Testing Guide

Sits after the **Git Workflow Guide** (branching and review process defined) and alongside the **Performance Testing Guide**. Write tests before — or alongside — implementation, never after. Tests written after the fact verify what was built, not what should have been built.

Language/framework-agnostic. Running example: **Event booking system**.

---

## The Core Rule

**Test behaviour, not implementation.**

Test what a unit does, not how it does it. If refactoring internals breaks tests without changing observable behaviour, the tests are wrong.

```
Wrong: "Does userService.create() call bcrypt.hash()?"
Right: "Does creating a user with a plaintext password result in a stored hash that verifies correctly?"
```

---

## 1. Test Pyramid & Philosophy

**Answers:** Which tests to write, how many, and why the ratio matters.

### The Pyramid

```
        /\
       /  \        E2E tests
      /────\       Few. Slow. High confidence on user journeys.
     /      \
    /────────\     Integration tests
   /          \    Some. Medium speed. Validates real wiring.
  /────────────\
 /              \  Unit tests
/────────────────\ Many. Fast. Validates logic in isolation.
```

**Ratio (rough target):**

| Layer | Count | Speed | Cost to write | Cost to maintain |
|---|---|---|---|---|
| Unit | 70% | < 1ms each | Low | Low |
| Integration | 20% | 100ms–5s each | Medium | Medium |
| E2E | 10% | 5s–60s each | High | High |

More E2E tests = slower pipeline, brittle suite, harder to diagnose failures. Invert the pyramid and you pay for it in CI time and flakiness.

### What each layer tests

| Layer | Tests | Does NOT test |
|---|---|---|
| Unit | Business logic, edge cases, error branches, pure functions | DB, HTTP, external services |
| Integration | Real DB queries, service orchestration, middleware chain | Browser, full user journeys |
| E2E | User journeys through real UI or HTTP client, critical happy paths | Every edge case (too slow) |

### Philosophy Rules

- Write the test before or alongside the code — never as a separate "testing phase" after
- A failing test is a specification. Write it red first, then make it green
- If a function is hard to test, the design is wrong — testing surfaces coupling
- Tests are first-class code: no magic globals, no copy-paste, same review standards as production code
- Flaky tests are bugs — fix or delete immediately, never ignore

---

## 2. Unit Testing

**Answers:** How to test individual functions and classes in complete isolation.

### What belongs in unit tests

```
shared/logic/     ← every function, every branch
shared/utils/     ← every utility function
State machines    ← every valid and invalid transition
Error classes     ← construction, message formatting
Validation schemas ← valid inputs, invalid inputs, edge cases
```

### Structure — Arrange / Act / Assert

```ts
describe('bookingStateMachine', () => {
  describe('transition()', () => {

    it('transitions PENDING → CONFIRMED on payment_succeeded', () => {
      // Arrange
      const booking = { status: 'PENDING' };

      // Act
      const result = transition(booking, 'payment_succeeded');

      // Assert
      expect(result.status).toBe('CONFIRMED');
    });

    it('throws on invalid transition CONFIRMED → CONFIRMED', () => {
      const booking = { status: 'CONFIRMED' };
      expect(() => transition(booking, 'payment_succeeded'))
        .toThrow(InvalidStatusTransitionError);
    });

    it('returns terminal state CANCELLED without accepting further transitions', () => {
      const booking = { status: 'CANCELLED' };
      expect(() => transition(booking, 'payment_succeeded'))
        .toThrow(InvalidStatusTransitionError);
    });

  });
});
```

### What to mock (and what not to)

| Mock | Don't mock |
|---|---|
| External HTTP calls (Stripe, email) | The module under test |
| DB in unit tests — use a fake/stub | Pure functions called by the unit |
| Time (`Date.now()`, `new Date()`) when testing time-dependent logic | Business logic in the same layer |
| Random values when testing determinism | The language's standard library |

**Mock rule:** if removing the mock would make the test hit a network or a disk, mock it. If removing the mock would just call another function in your own codebase, don't mock it — test the real thing.

### Testing pure functions

Pure functions (no side effects, same input → same output) need no mocking. Test exhaustively.

```ts
// shared/logic/classifyAttendance.ts
// Pure: takes args, returns result, no DB, no HTTP

describe('classifyAttendance()', () => {
  it('returns PRESENT when scanned within session window', () => {
    const result = classifyAttendance({
      scannedAt: new Date('2025-06-01T09:05:00Z'),
      sessionStart: new Date('2025-06-01T09:00:00Z'),
      lateThresholdMinutes: 10,
    });
    expect(result).toBe('PRESENT');
  });

  it('returns LATE when scanned after late threshold', () => {
    const result = classifyAttendance({
      scannedAt: new Date('2025-06-01T09:15:00Z'),
      sessionStart: new Date('2025-06-01T09:00:00Z'),
      lateThresholdMinutes: 10,
    });
    expect(result).toBe('LATE');
  });

  it('returns ABSENT when scanned after session end', () => { ... });
  it('throws when scannedAt is before session start', () => { ... });
});
```

### Edge cases to always test in unit tests

- [ ] Empty input / null / undefined
- [ ] Minimum and maximum valid values (boundary)
- [ ] One above minimum, one below maximum
- [ ] All enum values (especially for state machines)
- [ ] Division by zero / zero denominators
- [ ] Empty collections (array of length 0)
- [ ] Single-element collections

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Testing implementation (`did it call X?`) | Test breaks on refactor; doesn't verify behaviour |
| One test covers multiple behaviours | Failure message is ambiguous; fix is unclear |
| Mocking everything including the unit | Test verifies the mock, not the code |
| No test for error/failure branches | Error paths are where bugs live |
| Tests with no assertions (`expect()` calls) | Always passes; worthless |
| Shared mutable state between tests | Tests pass alone, fail in suite — order-dependent |

---

## 3. Integration Testing

**Answers:** Does the code work correctly when wired to real dependencies — real DB, real middleware, real service calls?

### What integration tests verify

```
Routes → Controller → Service → Real DB (not mocked)
Middleware chain (auth, validation, error handler)
DB constraints (unique violations, FK violations, check constraints)
Transactions (commit, rollback)
Scheduled jobs against real DB state
```

### What integration tests do NOT test

- Browser or UI behaviour (→ E2E)
- External services (Stripe, email) — still mock these
- Every edge case (→ unit tests)

### Setup rules

```
1. Use a dedicated test database — never the development DB
2. Reseed before every test suite run — known state always
3. Wrap each test in a transaction and roll back after — fastest cleanup
   OR truncate tables in afterEach — simpler but slower
4. Never share state between tests — each test sets up its own fixtures
5. Real DB engine — SQLite is not a substitute for Postgres if prod runs Postgres
```

### Structure — Event booking system

```ts
describe('POST /api/v1/bookings', () => {
  let app: Express;
  let db: PrismaClient;
  let attendeeToken: string;
  let eventId: string;

  beforeAll(async () => {
    app = await buildApp();
    db = new PrismaClient();
    attendeeToken = await getTestToken({ role: 'ATTENDEE' });
  });

  beforeEach(async () => {
    await db.$transaction([
      db.booking.deleteMany(),
      db.event.deleteMany(),
    ]);
    const event = await db.event.create({
      data: { title: 'Test Event', capacity: 10, booked: 9, status: 'PUBLISHED', ... }
    });
    eventId = event.id;
  });

  afterAll(async () => { await db.$disconnect(); });

  it('creates a booking and returns 201 with clientSecret', async () => {
    const res = await request(app)
      .post('/api/v1/bookings')
      .set('Authorization', `Bearer ${attendeeToken}`)
      .set('Idempotency-Key', 'test-key-001')
      .send({ eventId, attendeeCount: 1 });

    expect(res.status).toBe(201);
    expect(res.body.data.status).toBe('PENDING');
    expect(res.body.data.clientSecret).toBeDefined();

    const booking = await db.booking.findFirst({ where: { eventId } });
    expect(booking).not.toBeNull();
    expect(booking!.status).toBe('PENDING');
  });

  it('returns 409 when event is full', async () => {
    await db.event.update({ where: { id: eventId }, data: { booked: 10 } });

    const res = await request(app)
      .post('/api/v1/bookings')
      .set('Authorization', `Bearer ${attendeeToken}`)
      .set('Idempotency-Key', 'test-key-002')
      .send({ eventId, attendeeCount: 1 });

    expect(res.status).toBe(409);
    expect(res.body.code).toBe('EVENT_FULL');
  });

  it('returns 401 when no auth token provided', async () => {
    const res = await request(app)
      .post('/api/v1/bookings')
      .send({ eventId, attendeeCount: 1 });

    expect(res.status).toBe(401);
  });

  it('returns 403 when organiser tries to book their own event', async () => { ... });
  it('replays idempotent response on duplicate key', async () => { ... });
  it('rolls back seat reservation if payment intent creation fails', async () => { ... });
});
```

### What to cover in integration tests

For each endpoint, test:
- [ ] Happy path (correct input, correct role) → correct status + DB state
- [ ] Auth missing → 401
- [ ] Auth valid but wrong role → 403
- [ ] Validation failure (missing required field) → 400
- [ ] Resource not found → 404
- [ ] Business rule violation (event full, wrong state) → correct 4xx + code
- [ ] Idempotency: same key twice → same response, no duplicate DB records
- [ ] DB state after mutation: assert what changed AND what didn't

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Mock the DB in integration tests | Tests pass, real queries fail in prod — defeats the purpose |
| Share DB state between tests | Test A creates record; test B finds it unexpectedly; flaky |
| Test against SQLite when prod is Postgres | Different constraint behaviour, different date handling, missing extensions |
| Only test the happy path | 90% of bugs live in error branches and edge cases |
| No assertion on DB state after mutations | Service returns 201 but wrote nothing — test would pass |
| Integration tests that call real external APIs | Slow, expensive, flaky; mock external services at the client boundary |

---

## 4. End-to-End Testing

**Answers:** Do real user journeys work through the full running system?

### Scope

E2E tests run against the full stack (real server, real DB, real browser or HTTP client). They prove the system works as a whole, not that individual parts are correct.

**Cover with E2E:**
- The most critical user journeys (happy path only)
- Flows that cross multiple services or require a real browser
- Smoke tests after deployment

**Do NOT cover with E2E:**
- Every error case (→ integration tests)
- Edge cases (→ unit tests)
- Performance (→ load tests)

### Test selection — Event booking system

| Journey | E2E? | Why |
|---|---|---|
| Attendee browses events, books seat, pays, receives confirmation | Yes | Core revenue flow |
| Organiser creates event, publishes, views booking list | Yes | Core organiser flow |
| Admin creates user account | Yes | Onboarding gate |
| Wrong password shows error | No | Integration test — no browser needed |
| Event full returns 409 | No | Integration test |
| Email formatting | No | Unit test on template function |

### Structure (Playwright example)

```ts
// e2e/booking.e2e.ts
import { test, expect } from '@playwright/test';

test.describe('Event booking — happy path', () => {

  test.beforeEach(async ({ request }) => {
    // Reseed via API or direct DB call
    await request.post('/api/v1/test/reseed');
  });

  test('attendee can book a seat and see confirmation', async ({ page }) => {
    // Login
    await page.goto('/login');
    await page.getByLabel('Email').fill('attendee@test.com');
    await page.getByLabel('Password').fill('Test@1234');
    await page.getByRole('button', { name: 'Sign in' }).click();
    await expect(page).toHaveURL('/dashboard');

    // Browse and select event
    await page.getByRole('link', { name: 'Events' }).click();
    await page.getByText('React Workshop').first().click();
    await expect(page.getByText('50 seats available')).toBeVisible();

    // Book
    await page.getByRole('button', { name: 'Book now' }).click();
    await page.getByLabel('Number of seats').fill('1');
    await page.getByRole('button', { name: 'Confirm booking' }).click();

    // Payment (Stripe test mode)
    await page.getByLabel('Card number').fill('4242424242424242');
    await page.getByLabel('Expiry').fill('12/29');
    await page.getByLabel('CVC').fill('123');
    await page.getByRole('button', { name: 'Pay' }).click();

    // Confirmation
    await expect(page.getByText('Booking confirmed')).toBeVisible();
    await expect(page.getByText('React Workshop')).toBeVisible();
    await expect(page.getByRole('img', { name: 'QR ticket' })).toBeVisible();
  });

});
```

### E2E Conventions

```
Global setup:   reseed DB once before the entire suite
Per-test setup: reset only what the test changes
Selectors:      prefer getByRole, getByLabel over CSS selectors
                (accessible selectors catch real a11y regressions)
Assertions:     assert what the user sees, not internal state
Timeouts:       set explicit waits for async UI — never arbitrary sleep()
Parallelism:    E2E tests must be independent — can run in any order
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| E2E tests for every edge case | Suite takes 30+ min; no one runs it; it rots |
| CSS selector soup (`div.card > ul > li:nth-child(2)`) | Breaks on any UI refactor; not related to behaviour |
| `sleep(2000)` for async waits | Flaky on slow CI; arbitrary; masks real timing issues |
| No DB reseed between runs | Tests pass on clean DB, fail on dirty state from previous run |
| Testing implementation through the UI ("does this class exist?") | Tests break on style changes; not user behaviour |
| E2E tests that call real payment APIs | Slow, costs money, depends on external uptime |

---

## 5. API / Contract Testing

**Answers:** Does the API still honour the contract defined in the Design Guide — and does it break any consumer?

### Two types

**Provider tests** — verify your API matches the OpenAPI spec you published:
```
Run on every merge. Fast. Catches: missing fields, wrong types, wrong status codes.
```

**Consumer-driven contract tests (Pact)** — consumer defines what it expects; provider verifies it satisfies those expectations:
```
Use when: multiple teams consuming your API, or microservices calling each other.
Skip when: single frontend + single backend in same repo — integration tests cover this.
```

### OpenAPI validation testing

Every integration test response should be validated against the OpenAPI spec automatically:

```ts
import Ajv from 'ajv';
import spec from '../docs/openapi.json';

// In your test helper:
function assertMatchesSchema(responseBody: unknown, schemaRef: string) {
  const ajv = new Ajv();
  const validate = ajv.compile(spec.components.schemas[schemaRef]);
  const valid = validate(responseBody);
  if (!valid) throw new Error(`Response doesn't match schema: ${JSON.stringify(validate.errors)}`);
}

// In test:
it('GET /events returns valid event list', async () => {
  const res = await request(app).get('/api/v1/events');
  expect(res.status).toBe(200);
  assertMatchesSchema(res.body, 'EventListResponse');  // fails if shape drifts
});
```

### Breaking change detection

A breaking change is any change that causes an existing consumer to fail:

```
Breaking (requires API version bump):
  - Removing a field from a response
  - Changing a field's type (string → number)
  - Changing a required request field's name
  - Changing a status code on an existing endpoint
  - Adding a new required request field

Non-breaking (safe to ship):
  - Adding an optional request field
  - Adding a new field to a response
  - Adding a new endpoint
  - Making a required response field optional
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| No contract tests | API drifts from spec silently; frontend breaks |
| Manual API spec updates | Spec and implementation diverge immediately |
| Treating all changes as non-breaking | Clients fail on supposedly "safe" deploys |
| Consumer-driven contracts for single-team monorepo | Overhead without benefit; integration tests are sufficient |

---

## 6. Test Data Strategy

**Answers:** How is test data created, kept consistent, and cleaned up?

### Hierarchy

```
1. Factories     — programmatic builders for single objects
2. Seeds         — known baseline state for the whole suite
3. Fixtures      — static data files for specific scenarios (use sparingly)
```

### Factories

Factories create one object with sensible defaults, overridable per test. Never hardcode IDs in tests.

```ts
// tests/factories/event.factory.ts
import { faker } from '@faker-js/faker';

export function buildEvent(overrides: Partial<Event> = {}): CreateEventInput {
  return {
    title:       overrides.title       ?? faker.lorem.words(3),
    capacity:    overrides.capacity    ?? 50,
    booked:      overrides.booked      ?? 0,
    status:      overrides.status      ?? 'PUBLISHED',
    startsAt:    overrides.startsAt    ?? faker.date.future(),
    endsAt:      overrides.endsAt      ?? faker.date.future(),
    organiserId: overrides.organiserId ?? 'org_default',
  };
}

// In test:
const fullEvent = await db.event.create({ data: buildEvent({ capacity: 10, booked: 10 }) });
const draftEvent = await db.event.create({ data: buildEvent({ status: 'DRAFT' }) });
```

### Seeds

Seeds establish the baseline state that tests depend on. Run before the suite; tests mutate over this baseline.

```ts
// tests/seed.ts — run once before integration/E2E suite

export async function seed(db: PrismaClient) {
  await db.$transaction([
    db.booking.deleteMany(),
    db.event.deleteMany(),
    db.user.deleteMany(),
  ]);

  const admin = await db.user.create({
    data: { email: 'admin@test.com', role: 'ADMIN', passwordHash: hash('Admin@1234'), ... }
  });
  const organiser = await db.user.create({
    data: { email: 'organiser@test.com', role: 'ORGANISER', ... }
  });
  const attendee = await db.user.create({
    data: { email: 'attendee@test.com', role: 'ATTENDEE', ... }
  });

  await db.event.create({
    data: buildEvent({ organiserId: organiser.id, title: 'React Workshop', capacity: 50 })
  });
}
```

### Cleanup strategies (pick one per test layer)

| Strategy | How | Speed | Isolation |
|---|---|---|---|
| Transaction rollback | Wrap each test in transaction, roll back in afterEach | Fastest | Perfect |
| Truncate + reseed | Delete all rows + reseed in beforeEach | Slow | Perfect |
| Soft isolation | Each test creates own records with unique IDs | Fast | Good |
| No cleanup | Tests read-only against seeded data | Fastest | Read-only tests only |

**Transaction rollback pattern (preferred for integration tests):**
```ts
let tx: PrismaClient;

beforeEach(async () => {
  tx = await db.$begin();   // start transaction
});

afterEach(async () => {
  await tx.$rollback();     // undo all changes
});

// Pass tx to the service under test instead of db
```

### Test Data Rules

- [ ] No hardcoded IDs (`user_123`) — use factory-generated or seeded IDs via variables
- [ ] No production data in tests — never copy real emails, names, or payment refs
- [ ] No shared mutable state between tests — each test owns its data
- [ ] Faker for realistic-looking data — catches format-validation bugs that `"test"` misses
- [ ] Seed data documented in a table — team knows what baseline state looks like

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Hardcoded IDs in tests (`userId: 'abc123'`) | Works until the seed changes; breaks entire suite |
| Tests depend on order (`test B needs test A to run first`) | Suite fails in parallel or when order changes |
| Production data in test DB | GDPR violation; also makes tests non-deterministic |
| No cleanup — tests accumulate state | Test 100 fails because tests 1–99 left unexpected records |
| One giant seed with all possible data | Slow to run, hard to understand, every test is coupled to it |

---

## 7. Test Quality & Coverage

**Answers:** How do you know the tests are good, and how much coverage is enough?

### Coverage — what it means and doesn't

```
Coverage = lines/branches executed during tests / total lines/branches

What high coverage proves:   code was executed during tests
What it does NOT prove:      the behaviour is correct
                             edge cases are handled
                             the test has assertions
```

A test with no assertions gives 100% coverage of the lines it executes. Coverage is a floor, not a ceiling.

### Coverage thresholds

| Layer | Target | Rationale |
|---|---|---|
| `shared/logic/` | 95%+ | Pure functions — no excuse for gaps |
| `shared/errors/` | 90%+ | Simple classes — easy to test |
| `services/` | 80%+ | Business logic; some branches are defensive |
| `middleware/` | 85%+ | Security-critical; auth paths especially |
| `controllers/` | 70%+ | Thin layer; integration tests cover the rest |
| `repositories/` | Covered by integration tests | Unit mocking DB is pointless |
| Overall | 80%+ | Below this, significant logic is untested |

### Mutation testing

Coverage tells you code was run. Mutation testing tells you tests actually catch bugs.

```
Mutation testing: automatically introduces bugs (mutations) into your code
and checks whether your tests catch them.

Common mutations:
  > changed to >=
  + changed to -
  true changed to false
  return null added

If a mutation survives (no test fails), your tests are too weak.
```

Use when: critical business logic (payments, auth, state machines). Skip for simple getters.

### Test quality checklist

Each test should:
- [ ] Have exactly one reason to fail (tests one behaviour)
- [ ] Have a descriptive name: `{unit} {does what} when {condition}`
- [ ] Have at least one assertion
- [ ] Not depend on other tests running first
- [ ] Run in < 100ms (unit) or < 5s (integration)
- [ ] Be deterministic — same result every time

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Chasing 100% coverage | Incentivises empty tests with no assertions |
| Coverage as the only quality metric | Misses: wrong assertions, happy-path-only, no edge cases |
| Tests named `test1`, `it works`, `should pass` | Failure message gives no information |
| Giant test functions testing multiple things | One failure hides others; unclear which behaviour broke |
| Ignoring flaky tests | Flakiness = non-determinism = hidden bug or bad test design |

---

## 8. CI Test Gates

**Answers:** At what point in the pipeline does each test layer run, and what blocks a merge?

### Pipeline stages

```
On every commit / push:
  ├── lint              (< 30s)   — code style, type errors
  ├── unit tests        (< 2min)  — pure logic, no DB
  └── build             (< 3min)  — TypeScript compile, bundle

On pull request (blocks merge):
  ├── lint
  ├── unit tests + coverage gate
  ├── integration tests (< 10min) — real DB in CI container
  └── contract tests    (< 5min)  — OpenAPI validation

On merge to main:
  ├── all above
  ├── E2E tests         (< 20min) — full stack
  └── deploy to staging
```

### Coverage gate (CI enforcement)

```yaml
# jest.config.ts
coverageThreshold: {
  global: {
    lines:      80,
    branches:   75,
    functions:  80,
    statements: 80,
  },
  './src/shared/logic/': {
    lines:   95,
    branches: 90,
  }
}
```

Pipeline fails if coverage drops below threshold. This is a hard gate — no merge without meeting it.

### Test environment in CI

```yaml
# .github/workflows/test.yml (example)
services:
  postgres:
    image: postgres:16
    env:
      POSTGRES_DB: test_db
      POSTGRES_PASSWORD: test
    options: >-
      --health-cmd pg_isready
      --health-interval 10s

steps:
  - name: Run migrations
    run: npm run prisma:migrate
    env:
      DATABASE_URL: postgresql://postgres:test@localhost:5432/test_db

  - name: Seed test data
    run: npx ts-node tests/seed.ts

  - name: Run integration tests
    run: npm run test:integration
```

### Gate rules

| Gate | Blocks | Does not block |
|---|---|---|
| Lint errors | PR merge | — |
| Type errors | PR merge | — |
| Unit test failure | PR merge | — |
| Coverage below threshold | PR merge | — |
| Integration test failure | PR merge | — |
| E2E failure | Deploy to staging | PR merge (too slow for PR) |
| Contract test failure | PR merge | — |

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| No CI test gate — tests are optional | "I forgot to run tests" ships bugs |
| E2E on every commit | 20-minute feedback loop; devs stop pushing frequently |
| No coverage threshold | Coverage drifts down 1% per sprint; untested code accumulates |
| Skipping tests to meet a deadline | Technical debt compounds; next deadline is harder |
| Different DB engine in CI than prod | Tests pass in CI, fail in prod due to engine differences |
| Flaky tests allowed to block main | Developers start re-running CI to get lucky — trust collapses |

---

## 9. How This Feeds into Scaffolding

**Answers:** Where do test files live, and how does the test structure mirror the source structure?

### File placement

```
src/
  shared/
    logic/
      attendance.ts
      attendance.test.ts         ← unit test lives next to source
    utils/
      retry.ts
      retry.test.ts
  services/
    booking/
      booking.service.ts
      booking.service.test.ts    ← unit test for service logic

tests/                           ← test infrastructure only (not test cases)
  factories/
    event.factory.ts
    booking.factory.ts
    user.factory.ts
  helpers/
    auth.ts                      ← getTestToken(), buildApp()
    db.ts                        ← transaction helpers, truncate helpers
  seed.ts                        ← baseline seed for integration + E2E

src/__tests__/
  integration/
    booking.test.ts              ← integration tests (HTTP → DB)
    auth.test.ts
    events.test.ts

e2e/
  booking.e2e.ts                 ← E2E user journeys
  auth.e2e.ts
  global.setup.ts                ← reseed before full E2E run
```

### Naming conventions

```
Unit test:         {file}.test.ts
Integration test:  {feature}.test.ts  (under __tests__/integration/)
E2E test:          {journey}.e2e.ts   (under e2e/)
Factory:           {entity}.factory.ts
```

### Test scripts in package.json

```json
{
  "scripts": {
    "test":             "jest --testPathIgnorePatterns=integration --testPathIgnorePatterns=e2e",
    "test:unit":        "jest src/shared src/services --coverage",
    "test:integration": "jest src/__tests__/integration",
    "test:e2e":         "playwright test",
    "test:all":         "npm run test:unit && npm run test:integration && npm run test:e2e",
    "test:watch":       "jest --watch"
  }
}
```

### Scaffolding mapping

| Test type | Tests | Source location |
|---|---|---|
| Unit | `shared/logic/*.test.ts` | `shared/logic/` |
| Unit | `shared/utils/*.test.ts` | `shared/utils/` |
| Unit | `services/**/*.service.test.ts` | Service layer — business logic only |
| Integration | `__tests__/integration/*.test.ts` | Full route → DB stack |
| Contract | Part of integration suite | Against OpenAPI spec |
| E2E | `e2e/*.e2e.ts` | Full running system |
| Factories | `tests/factories/` | Not source — test infrastructure |
| Seed | `tests/seed.ts` | Not source — test infrastructure |

### Testing checklist (before calling a feature done)

- [ ] Unit tests for every function in `shared/logic/` covering happy path + all error branches
- [ ] Unit tests for state machine: every valid transition + every invalid transition
- [ ] Integration tests for every endpoint: happy path + auth failure + validation failure + business rule violation
- [ ] DB state asserted after every mutation test (not just response shape)
- [ ] Idempotency tested: same request twice → same response, no duplicate records
- [ ] E2E test for the primary user journey of the feature (happy path only)
- [ ] Coverage gate passes locally before pushing
- [ ] No skipped tests (`test.skip`, `xit`) merged to main
