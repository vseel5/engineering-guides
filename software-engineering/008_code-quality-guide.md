# Code Quality Guide

Sits alongside the **Scaffolding Guide** (structure) and **Testing Guide** (verification). These rules apply at the function and file level — they are enforced by tools where possible, and by code review where not.

Language/framework-agnostic. Running example: **Event booking system**.

---

## 1. Core Principle

**Code is read 10× more than it is written. Optimise for the reader.**

Every decision — naming, length, structure, comments — should reduce the cognitive load of the next person reading this code. That person is usually you, six months from now.

---

## 2. Style & Formatting

**Answers:** What does the code look like, and who decides?

### The rule: tool decides, humans don't argue

Formatting debates (tabs vs spaces, brace position, trailing commas) are a waste of review time. Pick a formatter, commit the config, and never discuss formatting again.

| Language | Tool | Config file |
|---|---|---|
| TypeScript / JavaScript | Prettier | `.prettierrc` |
| Python | Black + isort | `pyproject.toml` |
| Go | gofmt / goimports | built-in, no config needed |
| Rust | rustfmt | `rustfmt.toml` |
| Java / Kotlin | google-java-format | CI flag |

### Standard config (Prettier / TypeScript)

```json
// .prettierrc
{
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "bracketSpacing": true,
  "arrowParens": "always"
}
```

### Save = format (IDE enforcement)

```json
// .vscode/settings.json — committed to repo so all editors agree
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true,
    "source.organizeImports": true
  }
}
```

CI must also run the formatter in check mode — a PR that is not formatted fails the pipeline, regardless of IDE settings.

```bash
# CI step
prettier --check "src/**/*.ts"
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Manual formatting rules in the style guide | Humans apply them inconsistently; review comments on style waste time |
| Different formatter configs per developer | PRs are full of whitespace diffs; blame history is polluted |
| Formatter installed but not run on save | Depends on discipline; discipline fails under pressure |
| Formatter config not committed to repo | New developers format differently until someone notices |

---

## 3. Linting Rules

**Answers:** Which code patterns are banned, and why?

Linting is not style — it is correctness. Every rule below catches real bugs or makes debugging significantly harder.

### Must-have ruleset

| Rule | Why |
|---|---|
| No `console.log` / `print` / `debugger` | Debug output leaks into production logs; noise in log aggregators |
| No unused variables or imports | Dead code misleads readers; signals incomplete cleanup |
| No magic numbers (except `0`, `1`, `-1`) | `if (status === 3)` is unreadable; `if (status === BookingStatus.CONFIRMED)` is not |
| No nested ternaries | `a ? b ? c : d : e` — no one reads this correctly the first time |
| No empty `catch` blocks | Swallowed errors are invisible bugs |
| Explicit equality (`===` not `==` in JS) | `"0" == false` is `true`; never rely on coercion |
| No `any` in TypeScript | Disables type checking for that path; type errors become runtime errors |
| No implicit return type on public functions (TS) | Caller cannot know the contract without reading the body |
| No `var` (JS/TS) | Function-scoped; `let` / `const` are block-scoped and predictable |

### ESLint config (TypeScript example)

```json
// .eslintrc.json
{
  "rules": {
    "no-console": "error",
    "no-debugger": "error",
    "no-unused-vars": "error",
    "no-unused-expressions": "error",
    "eqeqeq": ["error", "always"],
    "no-nested-ternary": "error",
    "no-empty": "error",
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/explicit-function-return-type": ["error", {
      "allowExpressions": true
    }],
    "no-magic-numbers": ["error", {
      "ignore": [0, 1, -1],
      "ignoreArrayIndexes": true,
      "enforceConst": true
    }]
  }
}
```

### What to do instead of magic numbers

```ts
// Bad
if (booking.status === 3) { ... }
await new Promise(r => setTimeout(r, 30000));

// Good
const MAX_PENDING_MINUTES = 30;
const PENDING_TIMEOUT_MS = MAX_PENDING_MINUTES * 60 * 1000;

if (booking.status === BookingStatus.CONFIRMED) { ... }
await new Promise(r => setTimeout(r, PENDING_TIMEOUT_MS));
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Disabling lint rules inline (`// eslint-disable`) | Every disable is a rule violation hidden on purpose; review catches these |
| Treating lint warnings as optional | Warnings accumulate; "warning" becomes the new normal; errors stop meaning anything |
| Different lint config per developer | Rules enforced inconsistently across the codebase |
| No lint step in CI | Developers learn to rely on IDE only; CI is the last safety net |

---

## 4. Complexity Limits

**Answers:** How long and how complex can a function, file, or call stack be?

These are hard limits enforced by CI. When you hit a limit, the answer is to split — not to raise the limit.

### Hard limits

| Unit | Limit | Why |
|---|---|---|
| Function length | ≤ 20 lines | Fits in one screen; forces single responsibility |
| Cyclomatic complexity | ≤ 5 | Number of independent paths; above 5 = untestable in practice |
| Nesting depth | ≤ 3 levels | Deep nesting hides logic; forces early returns |
| Function parameters | ≤ 3 | More than 3 → use an options object |
| File length | ≤ 300 lines | Files over 300 lines are doing too much |

### Cyclomatic complexity explained

Complexity increases by 1 for each: `if`, `else if`, `for`, `while`, `case`, `catch`, `&&`, `||`, ternary.

```ts
// Complexity: 1 (no branches)
function formatName(first: string, last: string): string {
  return `${first} ${last}`;
}

// Complexity: 4 (if + else if + else + early return)
function classifyBooking(status: string, isPaid: boolean): string {
  if (status === 'CANCELLED') return 'cancelled';       // +1
  if (status === 'CONFIRMED' && isPaid) return 'active'; // +1 +1
  if (status === 'PENDING') return 'awaiting payment';   // +1
  return 'unknown';
}

// Complexity: 8 — split this function
function processBooking(booking: Booking, payment: Payment): void {
  if (booking.status === 'PENDING') {
    if (payment.status === 'succeeded') {
      if (booking.attendeeCount > 1) {
        // ... nested logic
      } else { ... }
    } else if (payment.status === 'failed') {
      if (booking.retryCount < 3) { ... }
      else { ... }
    }
  }
}
```

### Reducing nesting with early returns

```ts
// Bad — 4 levels deep
function confirmBooking(bookingId: string, userId: string): void {
  const booking = findBooking(bookingId);
  if (booking) {
    if (booking.attendeeId === userId) {
      if (booking.status === 'PENDING') {
        // ... actual logic here, buried 4 levels in
      }
    }
  }
}

// Good — max 1 level deep; same logic
function confirmBooking(bookingId: string, userId: string): void {
  const booking = findBooking(bookingId);
  if (!booking) throw new NotFoundError('Booking');
  if (booking.attendeeId !== userId) throw new ForbiddenError();
  if (booking.status !== 'PENDING') throw new InvalidStatusTransitionError(booking.status, 'CONFIRMED');

  // ... actual logic here, at the top level
}
```

### Reducing parameters with options objects

```ts
// Bad — 5 parameters, order matters, easy to swap
function createBooking(eventId: string, userId: string, count: number, idempotencyKey: string, organiserId: string): Booking

// Good — named, order-independent, extensible
interface CreateBookingInput {
  eventId: string;
  userId: string;
  attendeeCount: number;
  idempotencyKey: string;
}
function createBooking(input: CreateBookingInput): Booking
```

### CI enforcement (ESLint)

```json
{
  "rules": {
    "max-lines-per-function": ["error", { "max": 20, "skipBlankLines": true, "skipComments": true }],
    "max-depth": ["error", 3],
    "max-params": ["error", 3],
    "max-lines": ["error", 300],
    "complexity": ["error", 5]
  }
}
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Raising the limit instead of splitting | Limits exist to force splitting; raising them defeats the purpose |
| Long functions with a comment every 5 lines | Comments don't reduce complexity — splitting does |
| "It's all related, so it belongs in one function" | Related ≠ inseparable; split and call from a coordinator |
| Nesting happy path inside conditionals | Makes the reader parse conditions before seeing the logic |

---

## 5. Naming Rules

**Answers:** What should functions, variables, and booleans be called?

Names are the primary documentation of code. A well-named function needs no comment.

### Rules by type

| Type | Convention | Example |
|---|---|---|
| Boolean | `is`, `has`, `can`, `should` prefix | `isEventFull`, `hasAvailableSeats`, `canCancelBooking` |
| List / collection | Plural noun | `bookings`, `eventIds`, `confirmedUsers` |
| Function | Verb or verb phrase | `createBooking`, `findEventById`, `calculateRefundAmount` |
| Class | PascalCase noun | `BookingService`, `PaymentGateway`, `EventRepository` |
| Constant | SCREAMING_SNAKE_CASE | `MAX_ATTENDEES`, `DEFAULT_PAGE_SIZE` |
| Type / Interface (TS) | PascalCase noun | `CreateBookingInput`, `BookingStatus`, `PaginatedResponse` |

### Abbreviations: allowed vs banned

```
Allowed (universal):   id, url, api, db, http, html, css, json, uuid, dto, io
Banned (ambiguous):    mgr, proc, calc, svc, usr, evt, bkg, tmp, util, misc
```

When in doubt: spell it out. `bookingManager` is worse than `bookingCoordinator` but better than `bkgMgr`.

### Generic names (never use)

```ts
// Banned — tells the reader nothing
const data = await fetchBooking(id);
const obj = { status: 'CONFIRMED' };
const temp = booking.startsAt;
const thing = formatDate(temp);
const result = await createBooking(obj);
const info = await getEventDetails(id);
const val = calculateRefund(booking);

// Good
const booking = await fetchBooking(id);
const confirmedBooking = { status: 'CONFIRMED' };
const sessionStart = booking.startsAt;
const formattedStart = formatDate(sessionStart);
const newBooking = await createBooking(confirmedBooking);
const eventDetails = await getEventDetails(id);
const refundAmount = calculateRefund(booking);
```

### Naming event booking system examples

```ts
// Functions — verb phrases
async function reserveSeat(eventId: string, attendeeId: string): Promise<Booking>
async function expirePendingBookings(): Promise<number>
async function sendConfirmationEmail(bookingId: string): Promise<void>
function calculateRefundAmount(booking: Booking, policy: RefundPolicy): Money

// Booleans
const isEventPublished = event.status === 'PUBLISHED';
const hasAvailableSeats = event.capacity > event.booked;
const canCancelBooking = booking.status === 'CONFIRMED' && !isPastEventStart;
const shouldSendReminder = hoursUntilEvent < 24 && !reminderSent;

// Collections
const upcomingEvents: Event[] = [];
const confirmedBookingIds: string[] = [];
const attendeeEmailAddresses: string[] = [];
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| `isLoading` for a boolean that means "has errored" | Name lies; reader trusts the name |
| `getUsers()` that actually filters + sorts + paginates | Verb doesn't match behaviour |
| `e`, `b`, `u` as variable names outside a 2-line lambda | Unreadable beyond the immediate line |
| Abbreviations the team invents (`bkgSvc`, `evtRepo`) | Only decodable if you were there when they were invented |
| Inconsistent naming across files (`userId` vs `user_id` vs `uid`) | Reader must track conventions per file; automation can't refactor consistently |

---

## 6. Comments Policy

**Answers:** When should code have comments, and what should they say?

### The single rule

**Explain WHY, not WHAT.** Well-named code already says what it does. Comments earn their space only when the reason is non-obvious.

```ts
// Bad — restates the code
// Check if event is full
if (event.booked >= event.capacity) {
  throw new EventFullError();
}

// Good — explains a non-obvious constraint
// Use >= not === because booked can exceed capacity during concurrent requests
// that pass the initial check before the DB row lock is acquired.
if (event.booked >= event.capacity) {
  throw new EventFullError();
}
```

### When to write a comment

| Situation | Comment? |
|---|---|
| Hidden constraint (a ≥ instead of ===, a specific order required) | Yes |
| Workaround for an external bug or library quirk | Yes |
| Non-obvious business rule with no other documentation | Yes |
| WHY a design decision was made (if no ADR exists) | Yes |
| What the code does (readable from the code itself) | No |
| Which ticket or PR this relates to | No — that lives in git history |
| Who wrote this | No — that lives in git blame |
| Disabled code | No — delete it |

### Commented-out code: zero tolerance

```ts
// Bad
// const legacyBooking = await db.bookings.findOld(id);
const booking = await db.bookings.findById(id);

// const refund = booking.amount * 0.9;
const refund = calculateRefund(booking, refundPolicy);
```

Commented-out code creates noise, confuses readers, and never gets cleaned up. Git history preserves deleted code. Delete it.

### TODO format

Unformatted TODOs rot. They lose their author, their date, and their reason.

```ts
// Bad
// TODO: fix this
// TODO - handle edge case
// FIXME

// Good
// TODO(@vseel5): 2025-07-01 — handle the case where attendeeCount exceeds remaining seats
//   mid-transaction after the initial capacity check. Tracked in #234.
```

Required fields: `@username`, `YYYY-MM-DD`, reason. TODOs older than 90 days must be resolved or removed in code review.

### Docstrings / JSDoc — public APIs only

```ts
// Internal function — no docstring needed; naming is enough
function buildConfirmationUrl(bookingId: string): string { ... }

// Public API surface — document what the caller needs to know
/**
 * Reserves a seat for an event and initiates payment.
 * Does NOT confirm the booking — confirmation happens via the Stripe webhook.
 *
 * @throws {EventFullError} if no seats are available at time of reservation
 * @throws {DuplicateBookingError} if the attendee already has a booking for this event
 */
async function createBooking(input: CreateBookingInput): Promise<PendingBooking> { ... }
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Commented-out code in PRs | Noise; no one knows if it's safe to delete; it never gets cleaned up |
| `// increment counter` above `counter++` | Wastes a line; adds nothing |
| Undated, unowned TODOs | No accountability; they live forever |
| Over-commenting to compensate for bad naming | Fix the name, not the comment |
| Docstrings on every private function | Maintenance burden; they drift from the implementation |

---

## 7. Error Handling Discipline

**Answers:** What happens when something goes wrong — and is it always visible?

> Config loading, error class hierarchies, and the global error handler belong in the Scaffolding Guide. This section covers the discipline of *using* them correctly at the function level.

### Never swallow errors

```ts
// Bad — error is silently lost
try {
  await sendConfirmationEmail(bookingId);
} catch {}

// Bad — catches but doesn't re-throw; caller thinks it succeeded
try {
  await sendConfirmationEmail(bookingId);
} catch (err) {
  console.error(err);
}

// Good — caller knows it failed
try {
  await sendConfirmationEmail(bookingId);
} catch (err) {
  logger.error({ event: 'email:send_failed', bookingId, err });
  throw err;
}

// Also good — intentional swallow with explicit documentation
try {
  await sendConfirmationEmail(bookingId);
} catch (err) {
  // Email failure is non-fatal — booking is confirmed regardless.
  // Attendee can re-request confirmation from their dashboard.
  logger.warn({ event: 'email:send_failed_non_fatal', bookingId, err });
}
```

### Wrap errors with context

Raw errors lose context as they propagate up the call stack.

```ts
// Bad — stack trace points to the DB driver, not to your code
const booking = await db.booking.findUniqueOrThrow({ where: { id } });

// Good — context preserved; caller knows what failed and why
let booking: Booking;
try {
  booking = await db.booking.findUniqueOrThrow({ where: { id } });
} catch (err) {
  throw new NotFoundError(`Booking ${id} not found`);
}
```

### Business errors vs system errors

```
Business error (4xx): expected, part of the domain
  — EventFullError, DuplicateBookingError, InvalidStatusTransitionError
  — Do NOT log as error/warn (they are not system failures)
  — Log at info level if useful for analytics; skip if noise

System error (5xx): unexpected, needs investigation
  — DB connection failure, Stripe API down, OOM
  — Always log at error level with full context
  — Alert on-call for persistent 5xx
```

```ts
// Business error — log at info, or not at all
if (event.booked >= event.capacity) {
  logger.info({ event: 'booking:event_full', eventId });
  throw new EventFullError();
}

// System error — log at error, always
try {
  await stripe.paymentIntents.create({ ... });
} catch (err) {
  logger.error({ event: 'stripe:create_intent_failed', bookingId, err });
  throw new ExternalServiceError('Payment service unavailable');
}
```

### Fail fast — at startup only

Fail fast applies to configuration and initialisation — not to request handlers.

```ts
// Good — fail at startup if config is wrong
const config = z.object({ STRIPE_SECRET_KEY: z.string().min(1) }).parse(process.env);

// Bad — failing fast inside a request means one bad request kills the process
app.post('/bookings', (req, res) => {
  if (!config) process.exit(1);  // never do this in a request handler
});
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Empty `catch` blocks | Error is invisible; symptoms appear elsewhere with no trace |
| Catching `Error` and returning `null` | Caller cannot distinguish "not found" from "exception" |
| Logging and rethrowing without context | Duplicate log lines; no additional information added |
| Treating all errors as 500 | Business errors (full event, wrong status) are not system failures |
| `process.exit()` in request handlers | Kills the entire process on one bad request |

---

## 8. Side Effect Discipline

**Answers:** Which functions are pure, which are impure, and how are dependencies provided?

### Pure vs impure

```
Pure function:
  — Same input → same output, always
  — No DB, no HTTP, no files, no random, no Date.now(), no global state mutation
  — Independently unit-testable with no setup

Impure function:
  — Depends on or changes external state
  — Services, repositories, HTTP clients, email senders
  — Requires integration or mock-based tests
```

```ts
// Pure — no side effects, deterministic
function calculateRefundAmount(booking: Booking, policy: RefundPolicy): number {
  const hoursUntilEvent = differenceInHours(booking.event.startsAt, new Date());
  if (hoursUntilEvent > 48) return booking.totalAmount;
  if (hoursUntilEvent > 24) return booking.totalAmount * 0.5;
  return 0;
}

// Impure — depends on time, writes to DB, calls external service
async function processRefund(bookingId: string): Promise<void> {
  const booking = await bookingRepo.findById(bookingId);     // DB read
  const refund = calculateRefundAmount(booking, policy);     // pure call
  await stripe.refunds.create({ amount: refund, ... });      // external call
  await bookingRepo.updateStatus(bookingId, 'REFUNDED');     // DB write
}
```

### Dependency injection — pass dependencies, never instantiate inside business logic

```ts
// Bad — BookingService creates its own dependencies
class BookingService {
  private repo = new BookingRepository();        // hidden dependency
  private stripe = new Stripe(process.env.KEY!); // hidden dependency

  async createBooking(input: CreateBookingInput) {
    const event = await this.repo.findEvent(input.eventId);
    ...
  }
}

// Good — dependencies are injected; testable, swappable
class BookingService {
  constructor(
    private readonly repo: BookingRepository,
    private readonly paymentGateway: PaymentGateway,
  ) {}

  async createBooking(input: CreateBookingInput) {
    const event = await this.repo.findEvent(input.eventId);
    ...
  }
}

// In tests: inject fakes
const service = new BookingService(fakeRepo, fakePaymentGateway);
```

### Isolate impurity at the edges

```
Pure core, impure shell.

shared/logic/     ← pure only (classifyAttendance, calculateRefund, stateMachine)
services/         ← impure shell (orchestrates pure logic + repositories + clients)
repositories/     ← impure (DB)
shared/clients/   ← impure (HTTP to external APIs)
```

Side effects enter and exit at the service layer. Logic in `shared/logic/` has zero knowledge that a database or network exists.

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| `new Date()` inside a pure function | Makes the function non-deterministic; time-based tests are flaky |
| `Math.random()` inside business logic | Non-deterministic; inject a seeded random function for testability |
| Instantiating dependencies inside service constructors | Hidden coupling; cannot swap for tests or alternative implementations |
| Mixing pure logic with DB calls in the same function | Untestable without a real DB; forces integration test for unit-level behaviour |
| Global mutable state (`let currentUser = null`) | Invisible coupling; request A's state leaks into request B under concurrency |

---

## 9. No Hardcoding

**Answers:** What values are true constants that can live in code, and what must be externalised?

> The *mechanism* for loading config (validated env vars, `config/env.ts`) belongs in the Scaffolding Guide. This section covers only the *decision* of what should be configurable vs hardcoded.

### Decision table

| Category | Can hardcode? | Example |
|---|---|---|
| Mathematical constants | Yes | `Math.PI`, `SECONDS_IN_HOUR = 3600` |
| Domain constants (never change per deployment) | Yes | `FEEDBACK_RATING_MIN = 1`, `MAX_TICKET_QUANTITY = 10` |
| HTTP status codes | Yes | `404`, `401`, `201` (they are the spec) |
| Enum values | Yes | `'PENDING'`, `'CONFIRMED'` (they are the domain model) |
| External API base URLs | No | Stripe URL could change or need to point to a sandbox |
| Timeouts and retry delays | No | Need tuning in production without a redeploy |
| Secrets and credentials | Never | Obvious |
| Feature limits (page size, rate limits) | No | Need tuning per deployment |
| File paths and directories | No | Different in container vs local |
| Internal service URLs | No | Different per environment |
| Date/time values used in logic | No | Time zones and offsets vary by deployment |

### Examples — Event booking system

```ts
// Good — true domain constants (never change per deployment)
const FEEDBACK_RATING_MIN = 1;
const FEEDBACK_RATING_MAX = 5;
const MAX_BOOKING_QUANTITY = 10;
const TICKET_ID_PREFIX = 'TKT';

// Bad — operational values hardcoded
const PENDING_EXPIRY_MS = 30 * 60 * 1000;    // should be env var: BOOKING_PENDING_MINUTES
const STRIPE_WEBHOOK_SECRET = 'whsec_abc123'; // should be env var: STRIPE_WEBHOOK_SECRET
const MAX_PAGE_SIZE = 100;                     // should be env var: PAGINATION_MAX_LIMIT
const EMAIL_FROM = 'noreply@myapp.com';        // should be env var: EMAIL_FROM_ADDRESS
const RETRY_DELAY_MS = 1000;                   // should be env var: RETRY_BASE_DELAY_MS

// Bad — environment-specific values hardcoded
const API_BASE = 'https://api.stripe.com';    // breaks when pointing to sandbox
const DB_HOST = 'localhost';                  // breaks in any container
const LOG_DIR = '/var/log/app';              // breaks on Windows dev machines
```

### The test: "would this need a code change to run in a different environment?"

If yes → it must be configurable. If no → it can be a constant.

```ts
// "Would MAX_BOOKING_QUANTITY = 10 need to change for a different school?"
// No — it's a domain rule.  → Constant is fine.

// "Would PENDING_EXPIRY_MS = 1800000 need to change for a high-traffic event?"
// Yes — operations might want to tune it.  → Must be an env var.
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Hardcoding pagination limit (`limit = 20`) in query | Cannot tune without a deploy; different endpoints need different defaults |
| Hardcoding retry count (`for (let i = 0; i < 3; i++)`) | Cannot tune when a third-party is flaky without a deploy |
| "It's just a dev value" | Dev values become prod values when someone forgets to change them |
| Magic strings instead of enums (`status === 'confirmed'`) | Typo-prone; not refactorable by IDE; not discoverable |

---

## 10. Code Smells to Flag in Review

**Answers:** What patterns in a PR indicate the code will be hard to maintain or test?

These are not always bugs — they are signals that the design may need rethinking.

### Smell catalogue

**Duplication**
Same logic appears in two or more places. The second appearance is always slightly different — and the third will diverge further.
```ts
// Seen in bookingService.ts AND in waitlistService.ts:
const hoursUntil = (date: Date) => Math.floor((date.getTime() - Date.now()) / 3600000);
// → extract to shared/logic/time.ts
```

**Long conditionals**
More than 3 conditions in a single `if`/`switch` → hard to reason about, hard to test all branches.
```ts
// Smell
if (booking.status === 'CONFIRMED' && !event.isCancelled && user.role === 'ATTENDEE'
    && booking.attendeeId === user.id && event.startsAt > new Date()) { ... }

// Better: named booleans + guard clause pattern
const isOwner = booking.attendeeId === user.id;
const isUpcoming = event.startsAt > new Date();
const isActive = booking.status === 'CONFIRMED' && !event.isCancelled;
if (!isOwner || !isUpcoming || !isActive) throw new ForbiddenError();
```

**Input mutation**
Modifying a parameter the caller passed in — caller doesn't expect it.
```ts
// Bad — mutates the input object
function addMetadata(booking: Booking): void {
  booking.computedField = 'something'; // caller's object is now changed
}

// Good — return a new object
function addMetadata(booking: Booking): BookingWithMetadata {
  return { ...booking, computedField: 'something' };
}
```

**Primitive obsession**
Using raw primitives where a named type would communicate intent.
```ts
// Smell — three strings, one is an ID, one is a status, one is currency
function refund(id: string, status: string, currency: string): void { ... }

// Better — named types document intent; wrong-order bugs become type errors
function refund(bookingId: BookingId, status: BookingStatus, currency: CurrencyCode): void { ... }
```

**Deep inheritance chains**
More than 2 levels of inheritance for anything other than error classes.
```
BaseService → AbstractBookingService → ConcreteBookingService → ExtendedBookingService
```
Prefer composition. A `BookingService` that holds a `RefundCalculator` and a `NotificationSender` is easier to test and reason about than an inheritance chain.

**Boolean parameter flags**
A boolean parameter that changes what the function does is two functions pretending to be one.
```ts
// Smell — what does `true` mean here?
await createBooking(input, true);

// Bad — forces caller to know internal branching
async function createBooking(input: CreateBookingInput, isWaitlist: boolean): Promise<Booking>

// Good — two explicit functions
async function createBooking(input: CreateBookingInput): Promise<Booking>
async function addToWaitlist(input: CreateBookingInput): Promise<WaitlistEntry>
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Ignoring smells in "small" functions | Smells compound; a small function becomes a large one when features are added |
| Refactoring the smell without a test | No test to verify the refactor didn't break behaviour |
| "I'll clean it up later" | Later never comes; later becomes technical debt |
| Extracting a helper with the same complexity | Moved the problem, didn't solve it |

---

## 11. Automation vs Human Review

**Answers:** Which quality checks should be automated, and which require human judgement?

The dividing line: **computers enforce rules; humans judge intent**.

| Quality dimension | Owner | Tool / action |
|---|---|---|
| Formatting | Automated | Prettier / Black / gofmt in CI |
| Lint violations | Automated | ESLint / Pylint / golangci-lint in CI |
| Function length | Automated | `max-lines-per-function` lint rule |
| Cyclomatic complexity | Automated | `complexity` lint rule |
| Nesting depth | Automated | `max-depth` lint rule |
| Parameter count | Automated | `max-params` lint rule |
| File length | Automated | `max-lines` lint rule |
| No `any` / no `console.log` | Automated | ESLint in CI |
| Test coverage threshold | Automated | Jest / pytest coverage gate in CI |
| Naming intent (is the name clear?) | Human | Code review |
| Comment quality (does it add value?) | Human | Code review |
| Error handling discipline (is the empty catch intentional?) | Human | Code review |
| Side effect isolation (is this logic in the right layer?) | Human | Code review |
| Hardcoding decisions (should this be configurable?) | Human | Code review |
| Code smells (duplication, primitive obsession, boolean flags) | Human | Code review |
| Architecture fit (does this belong in a service or shared/logic?) | Human | Code review |

### The PR pipeline

```
push
  ↓
format check     (automated, < 30s)
  ↓
lint             (automated, < 60s)
  ↓
type check       (automated, < 2min)
  ↓
unit tests       (automated, < 2min)
  ↓
coverage gate    (automated, < 2min)
  ↓
code review      (human, asynchronous)
  ↓
integration tests (automated, < 10min, on merge)
```

If automated gates fail, the PR should not be reviewed — fix automation issues first.

---

## 12. Code Quality Checklist (before commit)

Run this before every commit. Not before the PR — before the commit.

- [ ] No `console.log`, `debugger`, or debug output left in
- [ ] No unused variables, imports, or dead code
- [ ] No magic numbers — every unexplained number is a named constant
- [ ] No commented-out code — deleted or reverted
- [ ] Every function ≤ 20 lines, every file ≤ 300 lines
- [ ] Every function name is a verb or verb phrase; every boolean uses `is/has/can`
- [ ] No generic names (`data`, `obj`, `temp`, `result`, `thing`, `info`)
- [ ] No empty `catch` blocks — every caught error is logged, re-thrown, or explicitly explained
- [ ] Every new pure function is in `shared/logic/` and has a unit test
- [ ] No values that should be configurable are hardcoded
- [ ] TODOs include `@username`, date, and reason
- [ ] Formatter ran on save (no formatting diff in the PR)

---

## 13. How This Feeds into Code Review

**Answers:** For each quality rule, what does the reviewer look for and how do they phrase feedback?

### Reviewer responsibility mapping

| Quality rule | What reviewer checks | Example review comment |
|---|---|---|
| Formatting | None — CI enforces this | (never comment on formatting) |
| No magic numbers | Every unexplained number | "What does `30000` represent? Extract as `PENDING_TIMEOUT_MS`." |
| No `any` | TypeScript `any` in new code | "`Payment` is typed as `any` — what's the actual shape? Use `Stripe.PaymentIntent`." |
| Function length > 20 lines | Scroll past one screen | "`processBooking` is 45 lines. Which part is capacity check, which is payment? Split." |
| Naming intent | Does the name match behaviour? | "`getBooking` also sends an email. Rename or split — names should not lie." |
| Boolean naming | `is/has/can` prefix | "`paid` is a boolean — rename to `isPaid` so call sites read as `if (booking.isPaid)`." |
| Generic names | `data`, `result`, `temp`, `obj` | "`result` here is a confirmed booking — name it `confirmedBooking`." |
| Comment quality | Does it explain WHY? | "This comment restates the code. Remove it, or explain why `>=` instead of `===`." |
| Commented-out code | Any `//` that looks like disabled code | "Remove the commented block — git history preserves it if needed." |
| Undated TODO | Any `// TODO` without `@user + date` | "TODO needs `@username` and a date. Add or convert to a tracked issue." |
| Empty catch | `catch {}` or `catch (e) { }` | "Silent catch — is email failure intentional? Add a comment explaining non-fatal, or re-throw." |
| Error context | Bare `throw err` without wrapping | "Re-throw wraps the DB error but loses context. `throw new NotFoundError('Booking ' + id)` tells the caller what failed." |
| Business vs system error | 500 for an expected condition | "`EventFullError` logged at `error` level — this is a business error, not a system failure. Log at `info` or skip." |
| Input mutation | Parameter object modified | "`booking.status = 'CONFIRMED'` mutates the input. Return a new object instead." |
| Hardcoded operational value | Timeouts, limits, URLs in logic | "`30 * 60 * 1000` — is this the pending expiry timeout? Should come from config, not the source." |
| DI violation | `new Dependency()` inside a class | "`new EmailClient()` inside `BookingService` — inject it via the constructor so it's testable." |
| `new Date()` in pure function | Any time reference in shared/logic/ | "`new Date()` in `calculateRefund` makes it non-deterministic. Pass `now: Date` as a parameter." |
| Duplication | Same logic in two places | "This `hoursUntil` calculation is also in `WaitlistService`. Extract to `shared/logic/time.ts`." |
| Boolean flag parameter | `fn(input, true)` | "`createBooking(input, true)` — what does `true` mean? Split into `createBooking` and `addToWaitlist`." |

### Review tone guidelines

```
Not: "This is wrong."
Yes: "This mutates the input — the caller doesn't expect their object to change.
      Return { ...booking, status: 'CONFIRMED' } instead."

Not: "Bad naming."
Yes: "getPaidStatus() returns a boolean — rename to isPaid() so it reads naturally at the call site."

Not: "Magic number."
Yes: "What does 1800000 represent? If it's the 30-minute pending timeout,
      extract as PENDING_EXPIRY_MS = 30 * 60 * 1000 with a name that explains the intent."
```

Reviewer comments name the problem, explain why it matters, and offer a concrete fix. They do not blame.

---

## How This Feeds Into Deployment

The quality rules in this guide are enforced as **CI pipeline gates** in the Deployment Guide (009). A build that violates them never reaches staging — let alone production.

| Quality rule | Pipeline gate |
|---|---|
| Linter passes (naming, imports, no-any) | `lint` step — fails the build immediately, before tests run |
| Complexity ≤ 5, function ≤ 20 lines, nesting ≤ 3 | Static analysis step (ESLint complexity rules, SonarQube) |
| Coverage ≥ 80% overall, ≥ 95% shared/logic | `test --coverage` step with threshold enforcement |
| No secrets in source | Secret scanning step (git-secrets, truffleHog) runs on every commit |
| Dependencies audited | `npm audit --audit-level=high` (or equivalent) blocks on any high/critical CVE |
| Dead code removed | Tree-shaking failures and unused exports caught by build step |
| PR size < 400 lines | Enforced by PR check (GitHub Action or similar) — oversized PRs are rejected before review |

**The feedback loop:** When the deployment pipeline catches a quality violation, the fix goes back through the Git Workflow (branch → PR → review) and must pass all quality gates again before it can merge. There is no bypass path. Quality is not a one-time gate at the start; it is continuously re-verified on every change.
