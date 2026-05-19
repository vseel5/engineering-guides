# Performance & Load Testing Guide

Sits alongside the **Testing Guide** (unit/integration/E2E). Where the Testing Guide verifies correctness — does the system do the right thing — this guide verifies capacity: does the system do the right thing under load, and how does it fail when it can't?

Language/framework-agnostic. Running example: **Event booking system**.

---

## 1. Core Principle

**Test the system the way production will use it, before production does.**

Performance problems discovered in production cost 10× more to fix than those found before launch. Load tests surface capacity limits, memory leaks, connection pool exhaustion, and N+1 queries that only appear at scale — none of which unit or integration tests can find.

---

## 2. Types of Performance Tests

**Answers:** What are the different types and when does each one apply?

| Type | What it does | When to run | Key metric |
|---|---|---|---|
| **Baseline test** | Establish normal performance at low load | Before any load testing | p50/p95/p99 latency at 10 users |
| **Load test** | Verify the system handles expected peak load | Before every significant release | Latency + error rate at expected RPS |
| **Stress test** | Find the breaking point | Quarterly or before high-risk events | At what load does the system fail? |
| **Spike test** | Simulate sudden traffic surge | Before known traffic events | Recovery time after sudden spike |
| **Soak test** | Run at moderate load for hours/days | Before first production launch | Memory leaks, connection exhaustion, log disk |
| **Capacity test** | Find maximum sustained throughput | When planning infrastructure scaling | Max RPS before SLA breach |

### When to skip

| Skip when | Reason |
|---|---|
| CRUD app with < 100 concurrent users | Overhead not justified; any modern stack handles this |
| Internal tool with < 50 users | Measure first; optimise only if measured problem |
| Early prototype | Architecture may change; test the stable version |

### When NOT to skip

- Before a known high-traffic event (concert ticket sale, product launch, exam season)
- Before the first production launch of a system expected to handle > 500 concurrent users
- After a significant architectural change (new DB, new queue, new caching layer)
- After a performance regression reported in production

---

## 3. Establishing a Baseline

**Answers:** What does "normal" look like, so you can tell when something is abnormal?

Run the baseline test before any load testing. It defines the healthy state of the system.

### Baseline test parameters

```
Virtual users: 10 (low — not testing load, testing baseline behaviour)
Duration:      5 minutes
Think time:    1 second between requests (realistic user behaviour)
Target:        all critical endpoints
```

### Baseline targets — Event booking system

| Endpoint | p50 target | p95 target | p99 target | Error rate |
|---|---|---|---|---|
| `GET /events` | < 50ms | < 100ms | < 200ms | 0% |
| `POST /bookings` | < 200ms | < 500ms | < 1000ms | 0% |
| `GET /bookings/:id` | < 50ms | < 100ms | < 200ms | 0% |
| `POST /auth/login` | < 100ms | < 200ms | < 500ms | 0% |
| `GET /health/ready` | < 10ms | < 20ms | < 50ms | 0% |

These are not aspirational — they are the SLA commitments to users. If baseline fails to meet targets, fix performance before running load tests.

### What to capture

```
For every test run, record:
  - Timestamp and git SHA (which version was tested)
  - p50, p95, p99 latency per endpoint
  - Error rate (%) per endpoint
  - Throughput (requests/second)
  - DB query latency (p99)
  - DB connection pool utilisation (peak)
  - Memory usage (start vs end — detect leaks)
  - CPU utilisation (peak)
```

Store results alongside the test scripts — not in someone's head or a spreadsheet that gets lost.

---

## 4. Load Test Design

**Answers:** How do you write a load test that reflects real usage?

### Traffic model — not uniform load

Real users do not arrive uniformly. Model the actual distribution:

```
Event booking system — realistic traffic model:
  10% browse events (GET /events, GET /events/:id)
  5%  search / filter events
  60% create bookings (POST /bookings) — the critical path
  15% view own bookings (GET /bookings?attendeeId=...)
  10% organiser dashboard (GET /events/:id/bookings)
```

A load test that hammers `POST /bookings` at 100% is not realistic. One that uses this distribution surfaces real bottlenecks.

### Virtual user scenario (k6 example)

```js
// tests/load/booking-flow.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const bookingDuration = new Trend('booking_duration', true);

export const options = {
  scenarios: {
    browse_events: {
      executor: 'constant-vus',
      vus: 10,             // 10% of 100 VUs
      duration: '5m',
      exec: 'browseEvents',
    },
    create_bookings: {
      executor: 'constant-vus',
      vus: 60,             // 60% of 100 VUs
      duration: '5m',
      exec: 'createBooking',
    },
  },
  thresholds: {
    'http_req_duration{endpoint:bookings}': ['p(95)<500'],  // 95% < 500ms
    'http_req_failed': ['rate<0.01'],                        // < 1% errors
    'errors': ['rate<0.01'],
  },
};

export function browseEvents() {
  const res = http.get(`${__ENV.BASE_URL}/api/v1/events`, {
    tags: { endpoint: 'events' },
  });
  check(res, { 'events 200': (r) => r.status === 200 });
  errorRate.add(res.status !== 200);
  sleep(1);
}

export function createBooking() {
  // Login first
  const loginRes = http.post(`${__ENV.BASE_URL}/api/v1/auth/login`, JSON.stringify({
    universityId: `test-user-${__VU}`,   // unique per VU
    password: 'Test@1234',
  }), { headers: { 'Content-Type': 'application/json' } });

  const token = loginRes.json('data.accessToken');

  const start = Date.now();
  const bookingRes = http.post(
    `${__ENV.BASE_URL}/api/v1/bookings`,
    JSON.stringify({ eventId: __ENV.EVENT_ID, attendeeCount: 1 }),
    {
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
        'Idempotency-Key': `load-test-${__VU}-${__ITER}`,
      },
      tags: { endpoint: 'bookings' },
    }
  );
  bookingDuration.add(Date.now() - start);

  check(bookingRes, {
    'booking 201 or 409': (r) => r.status === 201 || r.status === 409,
  });
  errorRate.add(bookingRes.status >= 500);
  sleep(1);
}
```

### Load ramp pattern

Never go from 0 to full load instantly — that is a spike test, not a load test. Ramp up gradually:

```js
export const options = {
  stages: [
    { duration: '2m', target: 25  },   // ramp up to 25 VUs
    { duration: '5m', target: 25  },   // hold at 25 VUs
    { duration: '2m', target: 100 },   // ramp up to 100 VUs
    { duration: '10m', target: 100 },  // hold at peak load
    { duration: '2m', target: 0   },   // ramp down
  ],
};
```

---

## 5. Stress & Spike Tests

**Answers:** Where does the system break, and how does it recover?

### Stress test — find the breaking point

Increase load until the system fails. The failure point tells you the safety margin between peak expected load and collapse.

```js
export const options = {
  stages: [
    { duration: '2m', target: 50  },
    { duration: '5m', target: 100 },
    { duration: '2m', target: 200 },
    { duration: '5m', target: 200 },
    { duration: '2m', target: 300 },
    { duration: '5m', target: 300 },
    // continue until error rate > 5% or latency > 5s
    { duration: '2m', target: 0   },
  ],
};
```

**What to record:**
- At what VU count did error rate exceed 1%? 5%?
- At what VU count did p99 latency exceed 5s?
- What was the first resource to saturate (DB connections, CPU, memory)?
- Did the system recover after load dropped? How fast?

### Spike test — sudden traffic surge

```js
export const options = {
  stages: [
    { duration: '1m',  target: 10  },   // normal baseline
    { duration: '30s', target: 300 },   // sudden spike — 30× normal
    { duration: '3m',  target: 300 },   // hold spike
    { duration: '1m',  target: 10  },   // back to normal
    { duration: '3m',  target: 10  },   // observe recovery
  ],
};
```

**What to look for:**
- Does the system recover after the spike, or does it stay degraded?
- Are requests queuing or immediately rejected?
- Does the circuit breaker trigger (if one is in place)?
- Are there any connection leaks post-spike?

### Event booking system — known spike scenario

```
"React Summit" event goes on sale at 10:00 UTC.
Expected: 5,000 booking attempts in the first 60 seconds.

Spike test: ramp from 10 VUs to 500 VUs in 30 seconds,
            hold for 2 minutes, drop back to 10.

Success criteria:
  - < 5% error rate at peak (500 VUs)
  - Seat reservation is correct (no overselling — verified post-test)
  - System recovers to baseline within 2 minutes of spike end
  - No DB connection pool exhaustion
```

---

## 6. Soak Test

**Answers:** Does the system degrade over time under sustained load?

A soak test runs at moderate load (50–70% of peak) for an extended period (2–24 hours). It surfaces problems that only appear over time: memory leaks, connection exhaustion, log disk fill, cache expiry bugs, database vacuum lag.

```js
export const options = {
  stages: [
    { duration: '5m',  target: 50 },   // ramp up
    { duration: '4h',  target: 50 },   // soak — 50 VUs for 4 hours
    { duration: '5m',  target: 0  },   // ramp down
  ],
};
```

### What to monitor during a soak

| Metric | Check | Sign of a leak |
|---|---|---|
| Memory (RSS) | Should be flat | Steadily increasing |
| DB connections | Should be stable | Creeping upward |
| Log disk | Should grow slowly | Rapidly filling |
| Error rate | Should be near 0% | Gradual increase over time |
| Latency p99 | Should be stable | Gradual increase over time |
| Redis memory | Should be stable | Growing without bound |

### When to run a soak test

- Before the first production launch
- After adding a new background job or scheduler
- After any change to connection pooling, caching, or session management
- After a memory-related production incident

---

## 7. Performance Test Infrastructure

**Answers:** Where do load tests run, and what do they target?

### Never run load tests against production

```
Wrong: k6 run --env BASE_URL=https://eventbooking.com booking-flow.js
Right: k6 run --env BASE_URL=https://staging.eventbooking.com booking-flow.js
```

Load tests against production affect real users, corrupt real data, and generate real payments. Always target staging — with a production-equivalent dataset and infrastructure.

### Staging must mirror production for load tests

| Requirement | Why |
|---|---|
| Same instance sizes as production | Underspec'd staging produces misleading results |
| Same DB engine and version | Different engines have different performance characteristics |
| Production-scale dataset (anonymised) | 10 rows responds differently than 10 million rows |
| Same network topology | Latency between app and DB must match |

A load test that passes on an undersized staging environment gives false confidence.

### Test data setup

```js
// setup.js — run before load test to seed test users and events
export function setup() {
  // Create 500 test users (one per max VU)
  // Create a test event with 10,000 capacity
  // Return { eventId, userCredentials } for use in the test
}
```

Clean up after: remove test bookings created during the test to avoid polluting staging data.

### Tools

| Tool | Strengths | Language |
|---|---|---|
| **k6** | Developer-friendly, JS scripting, CI integration, cloud option | JavaScript |
| **Locust** | Python, distributed, real-time UI, easy to scale | Python |
| **Gatling** | Scala DSL, detailed HTML reports, enterprise use | Scala |
| **Artillery** | YAML config, Node.js plugins, good for API testing | YAML / JS |
| **JMeter** | GUI, widely known, many plugins | GUI / XML |

**Recommendation:** k6 for most teams. Developer-friendly, CI-native, good free tier on k6 Cloud for distributed load generation.

---

## 8. Thresholds & Pass/Fail Criteria

**Answers:** How do you know if a load test passed or failed?

Every load test must have explicit pass/fail criteria defined before running. "It felt okay" is not a criterion.

### Threshold examples (k6)

```js
export const options = {
  thresholds: {
    // 95% of all requests must complete in < 500ms
    'http_req_duration': ['p(95)<500'],

    // 99% of booking requests specifically must complete in < 1000ms
    'http_req_duration{endpoint:bookings}': ['p(99)<1000'],

    // Overall error rate must stay below 1%
    'http_req_failed': ['rate<0.01'],

    // Custom metric: booking success rate > 99%
    'booking_success_rate': ['rate>0.99'],
  },
};
```

If any threshold fails, the k6 exit code is non-zero — the CI pipeline fails.

### Standard thresholds — Event booking system

| Test type | p95 latency | p99 latency | Error rate | Pass condition |
|---|---|---|---|---|
| Baseline (10 VUs) | < 100ms | < 200ms | 0% | All three met |
| Load (100 VUs, peak) | < 500ms | < 1000ms | < 1% | All three met |
| Stress (find breaking point) | Document only | Document only | Document at which VU count | Record numbers |
| Spike (30× surge) | < 2000ms during spike | < 5000ms | < 5% during spike | Recovery within 2 min |
| Soak (50 VUs, 4 hours) | < 500ms throughout | < 1000ms throughout | < 0.1% | No upward trend |

---

## 9. CI Integration

**Answers:** When do performance tests run in the pipeline, and what do they gate?

### Where performance tests fit

```
PR pipeline:        Unit + integration tests (fast, < 15 min)
Merge to main:      All tests + deploy to staging
Post-staging deploy: Baseline load test (automated, < 10 min)
Pre-release:        Full load test suite (manual trigger, 30–60 min)
Quarterly:          Stress test + soak test (manual trigger, 4+ hours)
```

Full load tests are not run on every PR — they are too slow and require production-scale staging infrastructure. The baseline test runs automatically post-deploy to catch regressions. Full load tests run before significant releases.

### Automated baseline test in CI

```yaml
# .github/workflows/staging-post-deploy.yml
post_deploy_baseline:
  needs: deploy_staging
  steps:
    - name: Run baseline load test
      run: |
        k6 run \
          --env BASE_URL=https://staging.eventbooking.com \
          --env EVENT_ID=${{ vars.LOAD_TEST_EVENT_ID }} \
          tests/load/baseline.js
      # k6 exits non-zero if thresholds fail → pipeline fails
```

### Triggering full load tests

```yaml
# Manual trigger on GitHub Actions
workflow_dispatch:
  inputs:
    test_type:
      type: choice
      options: [load, stress, spike, soak]
    vus:
      type: number
      default: 100
```

Full load tests require human approval — they consume significant infrastructure and run for 30+ minutes.

---

## 10. Diagnosing Performance Problems

**Answers:** When a load test fails, where do you look?

### Diagnosis order

```
1. Check error rate first — what errors are returning?
   → 5xx: system failure — look at application logs
   → 429: rate limited — check rate limit thresholds vs load test VU count
   → 409: business logic (event full) — may be expected
   → timeout: connection exhaustion or slow query — look at DB

2. Check latency breakdown — which endpoint is slow?
   → Use k6 tags to isolate per endpoint

3. Check DB metrics during the test run
   → DB connection pool utilisation > 80%? Pool exhaustion.
   → Slow query log: any query > 100ms at load?
   → Lock wait time spiking? Concurrency problem in seat reservation.

4. Check application metrics during the test run
   → CPU > 90%? Node.js is CPU-bound (unlikely for I/O-heavy API)
   → Memory growing? Possible leak — run soak test.
   → Queue depth growing? Workers can't keep up.

5. Check external dependencies
   → Stripe latency? External slowness isn't your bug.
   → Redis latency? Check connection count + memory.
```

### Common causes by symptom

| Symptom | Most likely cause | Check |
|---|---|---|
| Latency spikes at load | DB connection pool exhaustion | `active_db_connections` metric |
| Increasing latency over time | Memory leak or DB vacuum lag | Memory RSS over time; `VACUUM` log |
| High error rate at moderate load | Missing index — full table scan under concurrency | `pg_stat_statements`; slow query log |
| 5xx at high VUs but not low VUs | Connection timeout — pool too small | DB pool config vs VU count |
| Correct totals but slow | N+1 query | Count DB queries per request in logs |
| Spike recovery takes > 5 min | No circuit breaker; queue backed up | Queue depth; add circuit breaker |

### N+1 detection under load

```
Under load, N+1 queries become immediately visible:
  10 VUs, each requesting a list of 20 events with organisers
  → 10 × (1 + 20) = 210 queries/second just for this endpoint

Detect by:
  1. Enable query logging during load test
  2. Count DB queries per HTTP request
  3. If queries-per-request > 3, investigate for N+1

Fix: add `include` / `JOIN` to fetch related data in one query
```

---

## 11. Performance Testing Checklist

### Before running any load test
- [ ] Staging mirrors production (same instance sizes, same DB engine, same dataset scale)
- [ ] Test data seeded (test users, test events with sufficient capacity)
- [ ] Baseline established for comparison
- [ ] Pass/fail thresholds defined before running
- [ ] Monitoring dashboards open to observe during the test

### After a load test
- [ ] Results recorded (timestamp, git SHA, VU count, duration, p50/p95/p99, error rate)
- [ ] Thresholds passed — or failures triaged and documented
- [ ] DB slow query log reviewed
- [ ] Memory usage: start vs end (flat = no leak)
- [ ] DB connection pool: peak utilisation noted
- [ ] Test data cleaned up (remove load test bookings from staging)

### Pre-release load test gate
- [ ] Load test passed at expected peak load (100 VUs or project-specific)
- [ ] Spike test passed (5× peak surge, recovery < 2 min)
- [ ] No memory leak detected (soak test if first launch)
- [ ] All thresholds green — no manual override

---

## 12. Anti-Patterns

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| Running load tests against production | Real users affected; real money charged; real data corrupted | Always target staging |
| Uniform load (everyone hits the same endpoint) | Not realistic; hides the real bottleneck | Model actual traffic distribution |
| No ramp-up — jump straight to full VUs | Spike test masquerading as a load test; breaks differently | Always ramp up gradually |
| Testing with 10 rows in the DB | Queries that hit an index with 10 rows miss it with 1 million | Seed staging with production-scale data |
| No pass/fail thresholds | "It looked okay" — meaningless; no regression detection | Define thresholds before running |
| Only running load tests before launch | Performance regressions introduced in later releases go undetected | Automated baseline after every staging deploy |
| Treating load test results as exact numbers | Load tests have variance; compare trends, not single data points | Run 3 times; compare average |
| Optimising before measuring | Premature optimisation — fixing things that aren't problems | Baseline first; optimise only what is measured slow |

---

## 13. How This Feeds into Operations

Performance test results inform production operations directly.

| Load test finding | Operations action |
|---|---|
| Breaking point at 300 VUs | Scale trigger set at 200 VUs — 33% safety margin |
| DB pool exhaustion at 200 VUs | Operations alert added: pool > 80% utilisation → P2 |
| Memory grows 50MB/hour in soak | Memory alert added; investigate and fix before launch |
| p99 > 1s at 100 VUs | Latency alert threshold set at 1s; investigate slow query |
| Recovery takes 5 min after spike | Circuit breaker added; queue depth alert added |
| Baseline latency increased vs last release | Performance regression — block release until root cause found |
