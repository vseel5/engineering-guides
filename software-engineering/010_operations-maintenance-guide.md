# Operations & Maintenance Guide

The final guide in the pipeline. Sits after the **Deployment & CI/CD Guide**. Operations begins the moment the first deploy reaches production and never ends — it is the ongoing cost of running software for real users.

Language/framework-agnostic. Running example: **Event booking system**.

---

## 1. Core Principle

**You build it, you run it.**

The team that writes the code is responsible for operating it in production. This is not a punishment — it is the fastest feedback loop between code quality and real-world behaviour. A developer who is on-call for their own service writes better error messages, better runbooks, and better tests.

---

## 2. Monitoring & Metrics

**Answers:** Is the system healthy right now, and how do you know?

### The Four Golden Signals

These four metrics, measured for every service, tell you nearly everything about production health.

| Signal | What it measures | Alert when |
|---|---|---|
| **Latency** | Time to serve a request (p50, p95, p99) | p99 > 2× baseline for 5 minutes |
| **Traffic** | Requests per second | Drops > 30% below baseline (dead service) |
| **Errors** | % of requests returning 5xx | > 1% for 5 minutes |
| **Saturation** | How full is the system (CPU, memory, DB connections) | > 80% for 10 minutes |

Measure all four. An alert on any one of them catches most production problems.

### What to measure — Event booking system

**Application metrics:**

| Metric | Type | Labels |
|---|---|---|
| `http_requests_total` | Counter | `method`, `path`, `status_code` |
| `http_request_duration_seconds` | Histogram | `method`, `path` |
| `bookings_created_total` | Counter | `status` (pending/confirmed/failed) |
| `bookings_expired_total` | Counter | — |
| `payment_intents_total` | Counter | `outcome` (succeeded/failed) |
| `webhook_events_total` | Counter | `type`, `outcome` |
| `queue_depth` | Gauge | `queue_name` |
| `active_db_connections` | Gauge | — |

**Infrastructure metrics (collected by platform, not application):**

| Metric | Alert threshold |
|---|---|
| CPU utilisation | > 80% sustained 10 min |
| Memory usage | > 85% |
| Disk usage | > 80% |
| DB connection pool | > 80% utilised |
| DB query latency (p99) | > 500ms |

### Metric exposition (Prometheus format)

```ts
// shared/utils/metrics.ts
import { Counter, Histogram, Registry } from 'prom-client';

export const registry = new Registry();

export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'path', 'status_code'],
  buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
  registers: [registry],
});

export const bookingsCreated = new Counter({
  name: 'bookings_created_total',
  help: 'Total bookings created by outcome',
  labelNames: ['status'],
  registers: [registry],
});

// Expose metrics endpoint (authenticated — not public)
app.get('/metrics', authenticate, async (req, res) => {
  res.set('Content-Type', registry.contentType);
  res.send(await registry.metrics());
});
```

### Dashboards

Every service should have one dashboard with:

```
Row 1 — Traffic & Errors
  [ Requests/sec by endpoint ]   [ Error rate % ]   [ 5xx count ]

Row 2 — Latency
  [ p50 / p95 / p99 latency ]   [ Slowest endpoints ]

Row 3 — Business Metrics
  [ Bookings created/min ]   [ Payment success rate ]   [ Webhook processing lag ]

Row 4 — Infrastructure
  [ CPU % ]   [ Memory % ]   [ DB connections ]   [ Queue depth ]
```

**Dashboard rules:**
- One dashboard per service — not one per developer
- Committed to the repository as code (Grafana JSON, Datadog terraform) — not created manually
- Baseline "normal" annotated on graphs so anomalies are obvious

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Only monitoring infrastructure (CPU/memory) | Service can be healthy on CPU but returning 100% errors |
| No business metrics | Can't tell if "booking confirmed" is happening — only that HTTP works |
| Dashboards created manually, not as code | Next environment starts blind; dashboard lost when engineer leaves |
| Measuring only averages (not percentiles) | Average latency is 100ms but p99 is 10s — most users are fine, 1% are not |

---

## 3. Logging & Log Aggregation

**Answers:** When something goes wrong, can you reconstruct exactly what happened?

> The Scaffolding Guide's Observability section covers logging levels, what to log, and structured log format. Logger implementation details are project-specific (library choice, file transports, rotation). This section covers what to do with those logs once they exist in production.

### Log aggregation pipeline

```
Application container
  → stdout (structured JSON)
    → Log shipper (Fluentd / Logstash / Vector)
      → Log store (Elasticsearch / Loki / CloudWatch)
        → Query UI (Kibana / Grafana / CloudWatch Insights)
          → Alerts on log patterns
```

Never query raw container logs in production. All logs go through the aggregation pipeline so they are searchable, filterable, and retained.

### Log retention policy

| Log type | Retention | Reason |
|---|---|---|
| Error logs | 1 year | Incident investigation, compliance |
| Access logs (HTTP) | 90 days | Security audit, debugging |
| Application info/debug | 30 days | Active debugging only |
| Audit logs (auth events, mutations) | 7 years | Legal / compliance requirement |

### Structured log queries — Event booking system

Once logs are in an aggregator, these are the queries that matter:

```
# All errors in the last hour
level: error AND timestamp > now-1h

# Failed payments with booking context
event: "stripe:create_intent_failed" AND timestamp > now-24h

# Specific user's booking activity (incident investigation)
userId: "usr_abc123" AND timestamp > now-7d

# Slow requests (> 2s)
event: "http:request" AND duration_ms > 2000

# Auth failures by IP (detecting brute force)
event: "auth:login_failure" | stats count by ip | where count > 10

# Webhook processing lag
event: "webhook:processed" | stats avg(lag_ms) by type
```

### What a useful log entry looks like

```json
{
  "timestamp": "2025-06-01T09:15:32.123Z",
  "level": "error",
  "traceId": "a3f8c1",
  "userId": "usr_abc123",
  "universityId": "org_xyz",
  "role": "ATTENDEE",
  "service": "booking",
  "event": "stripe:create_intent_failed",
  "bookingId": "bkg_001",
  "eventId": "evt_react_workshop",
  "error": "Your card was declined.",
  "stripeCode": "card_declined",
  "ip": "192.168.1.1",
  "durationMs": 342
}
```

Given this entry, an on-call engineer can identify: who, what, when, where in the code, why it failed, and how long it took — without asking anyone.

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| `console.log('something happened')` | No structure; unsearchable; useless in aggregation |
| Logging PII (email, name, card details) | GDPR violation; security risk; cannot retain logs compliantly |
| No traceId in logs | Cannot correlate logs from the same request across services |
| Logs only on errors | Cannot reconstruct the sequence of events leading to the error |
| No log retention policy | Disk fills up; or logs deleted before a compliance audit |

---

## 4. Alerting

**Answers:** Who gets woken up, for what, and at what threshold?

### Alert design principles

```
An alert should be:
  Actionable    — the on-call engineer can do something about it
  Urgent        — it cannot wait until morning
  Novel         — not firing constantly (alert fatigue)

If an alert fires and the response is "check Grafana and wait" — it is not actionable.
If an alert fires 3+ times per week without action — it is noise; fix or remove it.
```

### Severity levels

| Severity | Meaning | Response time | Who is notified |
|---|---|---|---|
| **P1 — Critical** | Service is down or data is being lost | < 15 minutes | On-call engineer + lead |
| **P2 — High** | Significant degradation, partial outage | < 1 hour | On-call engineer |
| **P3 — Medium** | Degraded but functional; SLA at risk | < 4 hours (business hours) | Team channel |
| **P4 — Low** | Warning, trending toward problem | Next business day | Team channel |

### Alert catalogue — Event booking system

| Alert | Condition | Severity | Reason |
|---|---|---|---|
| High error rate | 5xx > 1% for 5 min | P1 | Service is broken for users |
| Service down | No successful requests for 2 min | P1 | Complete outage |
| Payment failure spike | Payment failures > 10% for 10 min | P1 | Revenue impact |
| High latency | p99 > 5s for 5 min | P2 | Users experience timeouts |
| DB connection saturation | Pool > 90% for 5 min | P2 | Imminent connection exhaustion |
| Queue depth growing | Queue depth > 1000 for 10 min | P2 | Worker falling behind |
| Webhook processing lag | Avg lag > 5 min for 10 min | P3 | Confirmations delayed |
| Disk usage high | Disk > 80% | P3 | Trending toward full |
| Certificate expiry | TLS cert expires in < 14 days | P3 | Advance warning |
| Elevated 4xx rate | 4xx > 5% for 15 min | P4 | Possible client bug or scraper |
| Slow queries | DB p99 > 500ms for 15 min | P4 | Index missing or query regression |

### Alert routing

```
P1 → PagerDuty → phone call + SMS + Slack #incidents
P2 → PagerDuty → Slack #incidents + email
P3 → Slack #ops-alerts
P4 → Slack #ops-alerts (low priority)
```

### Alert fatigue prevention

```
Rules:
1. Every alert that fires must have a runbook entry
2. Any alert that fires more than 3 times/week without action → fix the cause or raise the threshold
3. "Warning" alerts that require no action → delete them
4. Alert on symptoms (high error rate), not causes (CPU high)
   Cause-based alerts fire for harmless reasons; symptom-based alerts mean users are affected
5. Business-hours-only for P3/P4 — do not wake people for non-urgent issues
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Alert on every metric crossing a threshold | Thousands of alerts per day; on-call ignores them all |
| No runbook for an alert | Engineer woken up with no guidance on what to do |
| Alerting on causes (CPU) not symptoms (errors) | CPU spike may be harmless; 5xx spike always is not |
| Same severity for everything | P1 loses meaning; everything gets triaged the same |
| Alerts that only fire in business hours | P1 incidents happen at 3am; time-gating P1s is wrong |

---

## 5. On-Call

**Answers:** Who is responsible for production right now, and what are they expected to do?

### Rotation structure

```
Primary on-call:   first responder — responds within 15 min for P1
Secondary on-call: escalation if primary unreachable — responds within 30 min
Manager escalation: if secondary unreachable or incident > 1 hour
```

**Rotation length:** 1 week. Longer rotations cause burnout. Shorter rotations mean constant context-switching.

**Team size:** on-call only makes sense if the team is ≥ 4 people. With fewer, rotate less frequently and compensate fairly.

### On-call expectations

| Expectation | Detail |
|---|---|
| Response time | P1: < 15 min. P2: < 1 hour. Acknowledged, not resolved. |
| Availability | Reachable by phone and able to open a laptop within response time |
| Documentation | Every incident must have an entry in the incident log |
| Handoff | Outgoing on-call briefs incoming on-call on any open issues |
| Compensation | On-call outside business hours is compensated — document this in team agreement |

### Handoff template

```
## On-call handoff — week of 2025-06-01

Outgoing: @vseel5
Incoming: @teammate

Open issues:
  - Webhook processing lag spiked Thursday evening — resolved, root cause: Redis OOM.
    Watch queue depth dashboard for recurrence.

Deployments this week:
  - v1.4.2: added waitlist feature — no issues
  - v1.4.3: hotfix for email template — no issues

Known risks next week:
  - Large event "React Summit" on Wednesday — peak traffic expected 18:00–20:00
  - DB migration for v1.5.0 is destructive (expand phase only — contract in two weeks)

Runbook updates needed:
  - Redis OOM runbook needs a "check memory fragmentation" step added
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| No on-call rotation — one person always on-call | Burnout; single point of failure; knowledge siloed |
| On-call with no runbooks | Engineer is on-call but has no guidance; MTTR is high |
| No compensation for after-hours on-call | Resentment; good engineers leave |
| P1 alerts go to a team channel, not a person | Everyone assumes someone else will respond |
| No handoff process | Incoming on-call has no context; problems persist across rotations |

---

## 6. Incident Response

**Answers:** When something breaks in production, what is the process from detection to resolution?

### Incident lifecycle

```
Detection → Triage → Communicate → Investigate → Mitigate → Resolve → Post-mortem
```

### Step by step

**1. Detection (0–5 min)**
- Alert fires OR user reports a problem
- On-call acknowledges the alert (stops escalation)
- Initial severity assessment: is this P1, P2, or P3?

**2. Triage (0–10 min)**
- Check dashboards: error rate, latency, traffic
- Check recent deploys: anything in the last 2 hours?
- Check recent migrations: any schema changes?
- Decide: roll back or investigate?

**3. Communicate (< 10 min for P1)**
- Open a dedicated incident channel: `#inc-2025-06-01-booking-down`
- Post initial status update — even if you know nothing yet:
  ```
  Status: Investigating
  Impact: Booking creation failing with 503 since ~09:15 UTC
  Lead: @vseel5
  Next update: 09:30 UTC
  ```
- Update the status page if public-facing

**4. Investigate**
- Trace via logs using `traceId` of a failing request
- Check error messages — what specific error is being thrown?
- Narrow the scope: is it all users or a subset? All endpoints or one?
- Hypothesis → test → confirm or discard → next hypothesis

**5. Mitigate (stop the bleeding)**
- Mitigation restores service, not necessarily fixes root cause
- Options: roll back deploy, disable feature flag, scale up, kill a rogue job, redirect traffic
- Mitigation first, root cause analysis after service is restored

**6. Resolve**
- Service restored to normal behaviour
- Confirm with dashboards: error rate, latency, traffic all back to baseline
- Update status page: resolved
- Final incident channel message:
  ```
  Status: Resolved
  Duration: 09:15–09:52 UTC (37 minutes)
  Impact: ~40% of booking creation requests failed
  Root cause: Redis connection timeout after OOM — queue workers stopped processing
  Fix: Restarted Redis, increased memory limit
  Post-mortem: scheduled 2025-06-03
  ```

**7. Post-mortem**
- See Section 7.

### Severity classification

| Severity | Example | Expected MTTR |
|---|---|---|
| P1 | All bookings failing, payment errors, data loss | < 30 minutes |
| P2 | Booking slow (> 5s), email confirmations delayed > 30 min | < 2 hours |
| P3 | Analytics broken, admin dashboard error, non-critical feature down | < 24 hours |

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Investigating root cause before mitigating | Users affected for longer; root cause can wait |
| No incident channel — debugging in DMs | Context is lost; no one can help; no record |
| No status updates during incident | Stakeholders escalate because they have no information |
| Declaring incident resolved before dashboards confirm | Incident re-opens; trust in process erodes |
| Skipping post-mortem for "small" incidents | Small incidents recur; patterns go unnoticed |

---

## 7. Runbooks

**Answers:** When an alert fires, what does the on-call engineer do step by step?

A runbook is a documented procedure for a known scenario. It transforms "I have no idea what to do" into "follow steps 1–6." Every alert must have a runbook.

### Runbook template

```markdown
# Runbook: [Alert Name]

**Severity:** P1 / P2 / P3
**Alert condition:** [what triggers this alert]
**Last updated:** YYYY-MM-DD by @username

## Impact
What the user experiences when this alert fires.

## Likely causes
1. Most common cause (seen X times)
2. Second most common cause
3. Rare cause

## Diagnosis steps
1. Check [dashboard / log query] for [what to look for]
2. Run: [command] — expected output: [what normal looks like]
3. If [condition] → go to Mitigation A
   If [condition] → go to Mitigation B

## Mitigation A: [name]
1. [Step]
2. [Step]
Expected outcome: [what should happen]

## Mitigation B: [name]
1. [Step]
2. [Step]

## Escalate if
- Mitigation does not resolve within 15 minutes
- Data loss is suspected
- Escalate to: @oncall-lead → @team-lead → @cto

## Related
- [Link to dashboard]
- [Link to related runbook]
- [Post-mortem from last occurrence]
```

### Example — Event booking system

```markdown
# Runbook: High Payment Failure Rate

**Severity:** P1
**Alert condition:** Payment failures > 10% for 10 consecutive minutes
**Last updated:** 2025-06-01 by @vseel5

## Impact
Attendees cannot complete bookings. Seat reservations held as PENDING
until the 30-minute expiry job runs. Revenue impact.

## Likely causes
1. Stripe API degradation (most common — check status.stripe.com)
2. Invalid Stripe API key (after secret rotation)
3. Network issue between app and Stripe (check VPC/firewall)
4. Code bug in payment intent creation (after a deploy)

## Diagnosis steps
1. Check status.stripe.com — is Stripe reporting an incident?
   → Yes: go to Mitigation A (Stripe degradation)
2. Check logs: event:"stripe:create_intent_failed" in last 15 min
   → stripeCode: "authentication_required" → go to Mitigation B (key issue)
3. Check recent deploys — any deploy in the last 2 hours?
   → Yes: go to Mitigation C (roll back)
4. Check Stripe dashboard → API error rate graph

## Mitigation A: Stripe degradation
1. Post in #inc channel: "Stripe degraded — monitoring their status page"
2. Enable feature flag: disable-new-bookings (shows maintenance message)
3. Monitor Stripe status every 15 minutes
4. When Stripe recovers: disable the feature flag
5. Check queue depth — any backlog of webhook retries to process?

## Mitigation B: Invalid API key
1. Check env var STRIPE_SECRET_KEY in secrets manager matches Stripe dashboard
2. If mismatch: update secret in secrets manager → redeploy (triggers secret refresh)
3. Verify: tail logs for stripe:create_intent events — should succeed

## Mitigation C: Code regression
1. Roll back to previous image SHA (see deployment runbook)
2. Verify error rate returns to baseline
3. Open incident investigation for root cause

## Escalate if
- Mitigation A: Stripe outage > 1 hour → escalate to management for customer comms
- Any mitigation: not resolved in 30 minutes → escalate to @team-lead

## Related
- [Stripe status page]
- [Payment dashboard in Grafana]
- [Post-mortem 2025-03-15: invalid key after rotation]
```

### Runbook maintenance

- Runbooks live in the repository (`docs/runbooks/`) — not a wiki that drifts
- Updated immediately after any incident that reveals a gap or an incorrect step
- Reviewed quarterly — stale runbooks are worse than no runbooks (false confidence)
- Every new alert gets a runbook before it is enabled

---

## 8. Post-Mortems

**Answers:** How does the team learn from every incident so it never happens the same way twice?

### Blameless post-mortem

The goal is to understand what happened and fix the system — not to find who is at fault. Individuals make mistakes; systems either catch those mistakes or they don't. The system is what gets improved.

**Never in a post-mortem:** "who made the mistake," "who should have caught this," "why did X do Y."

**Always in a post-mortem:** "what conditions made this possible," "what system change prevents recurrence."

### Post-mortem template

```markdown
# Post-mortem: [Incident Title]

**Date:** YYYY-MM-DD
**Duration:** HH:MM – HH:MM UTC (N minutes)
**Severity:** P1 / P2 / P3
**Author:** @username
**Reviewers:** @teammate1, @teammate2

## Summary
One paragraph: what happened, who was affected, how it was resolved.

## Timeline (UTC)
| Time  | Event |
|-------|-------|
| 09:15 | Alert fired: high error rate on POST /bookings |
| 09:17 | On-call acknowledged, opened #inc-2025-06-01 |
| 09:22 | Identified Redis OOM as likely cause |
| 09:35 | Redis restarted, error rate returned to baseline |
| 09:52 | Incident resolved, status page updated |

## Root cause
[What was the underlying technical cause? Be specific.]
Redis ran out of memory due to a missing TTL on idempotency keys,
causing key accumulation over 72 hours until OOM killed the connection.

## Contributing factors
- No alert on Redis memory usage (monitoring gap)
- Idempotency key TTL was set correctly in code but not in the test environment
- Deploy on Thursday added 3× more traffic; brought forward the OOM by 5 days

## What went well
- Alert fired within 2 minutes of the first user impact
- Runbook correctly identified Redis as the most likely cause
- Rollback was not needed — restart was sufficient

## What went wrong
- No Redis memory dashboard — diagnosis took 15 minutes instead of 5
- Status page not updated until 09:45 (30 minutes into incident)
- Secondary on-call not informed until 09:30

## Action items
| Action | Owner | Due date |
|--------|-------|----------|
| Add Redis memory alert (P2 at 80%) | @vseel5 | 2025-06-08 |
| Add TTL to idempotency keys in test environment | @vseel5 | 2025-06-05 |
| Add Redis memory graph to main dashboard | @teammate | 2025-06-08 |
| Update status page SOP — must be updated within 10 min of P1 | @team-lead | 2025-06-10 |

## Lessons learned
[What systemic insight does this incident reveal?]
We had no observability on Redis specifically — it was monitored as part of general
infrastructure but not as a first-class dependency of the booking service.
Any service with a Redis dependency should have Redis memory in its service dashboard.
```

### Action item rules

- Every action item has an owner and a due date — not "team" and "soon"
- Action items are tracked in the issue tracker, not just the document
- Reviewed at the next team meeting — not filed and forgotten
- Post-mortem published within 5 business days of the incident

---

## 9. Backup & Recovery

**Answers:** If data is lost or corrupted, how is it restored, and how fast?

### Key terms

| Term | Definition | Target — Event booking system |
|---|---|---|
| **RPO** (Recovery Point Objective) | Maximum acceptable data loss (how old can the restored data be?) | 1 hour (lose at most 1 hour of bookings) |
| **RTO** (Recovery Time Objective) | Maximum acceptable downtime during recovery | 4 hours (service restored within 4 hours) |

### Backup strategy

```
Full backup:        Daily at 02:00 UTC (low traffic)
Incremental backup: Every 1 hour (WAL archiving for Postgres)
Backup retention:   30 days full, 7 days incremental
Backup location:    Separate cloud region from production (not same AZ)
```

**Postgres continuous archiving (WAL):**
```
WAL archiving → S3 (cross-region)
Point-in-time recovery: restore to any second within retention window
```

### Backup verification — most important rule

**A backup that has never been restored is not a backup. It is an assumption.**

```
Test schedule:
  Monthly:   restore backup to a throwaway environment, verify data integrity
  Quarterly: full disaster recovery drill — restore from backup + replay WAL → verify

Test what to check after restore:
  - Row counts match expected
  - Recent transactions present (check last 10 bookings)
  - DB constraints still intact
  - Application connects and passes health check
  - Payment records reconcile with Stripe dashboard
```

### What to back up

| Data | Backup method | Frequency | Notes |
|---|---|---|---|
| PostgreSQL | WAL archiving + daily full | Continuous + daily | Point-in-time recovery |
| Redis | RDB snapshot or AOF | Daily (or AOF continuous) | Redis is cache/queue — lower priority than DB |
| Application config | Git | Every commit | `.env.example` in repo; actual secrets in secrets manager |
| Secrets | Secrets manager replication | Continuous | Cross-region replication |
| Logs | Log aggregator retention | Per log retention policy | Immutable once written |

### Recovery procedure

```
1. Identify: what data is affected and from what point in time?
2. Choose restore point: last clean backup before the incident
3. Restore to isolated environment first — verify before touching production
4. Restore to production:
   a. Put application in maintenance mode
   b. Restore DB from backup
   c. Replay WAL to the point before corruption (if using point-in-time)
   d. Verify data integrity
   e. Remove maintenance mode
5. Reconcile: what transactions occurred between the restore point and the incident?
   — Contact affected users if bookings were lost
   — Reconcile payments with Stripe (refund or re-confirm as appropriate)
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Backups never tested | Backup is corrupt or incomplete; discovered during a disaster |
| Backup in the same region as production | Regional failure takes out both production and backup |
| No RPO/RTO defined | Recovery decisions made under pressure with no target |
| Redis not backed up | Queue state lost; in-flight jobs lost |
| Restore procedure documented but never rehearsed | Steps are wrong; tools not installed; permissions missing |

---

## 10. Capacity Planning & Scaling

**Answers:** Is the system sized for current and future load, and when should it grow?

### Scaling triggers

Define in advance — not reactively when the system is already saturated.

| Metric | Horizontal scale trigger | Vertical scale trigger |
|---|---|---|
| CPU | > 70% sustained 30 min | Single-threaded bottleneck |
| Memory | > 80% sustained 30 min | Memory-bound workload |
| DB connections | > 70% pool utilisation | Increase pool size first |
| Request queue depth | > 100ms average wait | Add instances |
| Disk | > 70% | Expand volume |

### Horizontal vs vertical scaling

```
Horizontal (more instances):
  + Linear cost scaling
  + No downtime
  + Preferred for stateless services (API servers)
  - Not possible for stateful services without distributed coordination

Vertical (bigger instance):
  + Simple — no application changes
  - Has a ceiling
  - Requires restart (brief downtime) for some platforms
  - Use for: DB, Redis, single-process bottlenecks
```

### Capacity planning process

```
Monthly review:
  1. Plot current usage trend (30-day growth rate)
  2. Project forward 3 months at current growth rate
  3. If projected usage crosses 70% threshold within 3 months → plan scaling now
  4. Scheduled scaling (known events) → provision in advance

Inputs:
  - Current resource utilisation (from dashboards)
  - Planned features that increase load (new integrations, new endpoints)
  - Known traffic events (concerts, large events, seasonal peaks)
```

### Event booking system — known traffic patterns

```
Peak times:        Monday 09:00–11:00 (event discovery), Friday 17:00–19:00
Seasonal peaks:    September (new academic year), January (new year events)
Planned events:    "React Summit" (2025-06-11) — projected 5× normal booking rate

Pre-event actions (React Summit):
  - Scale API instances from 3 → 8 the night before
  - Increase DB connection pool from 20 → 50
  - Pre-warm Redis cache
  - Increase rate limits on booking endpoints
  - Enable canary monitoring during peak
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Reactive scaling only | System is already degraded when scaling starts |
| No load testing before known peak events | Discover capacity limit during the event |
| Scaling without understanding the bottleneck | Adding API instances when DB is the bottleneck does nothing |
| Never scaling down after peaks | Unnecessary cost sustained indefinitely |

---

## 11. Dependency & Security Patching

**Answers:** How are outdated or vulnerable dependencies discovered and updated?

### Dependency audit cadence

| Action | Frequency | Tool |
|---|---|---|
| Automated vulnerability scan | Every CI run | `npm audit`, `pip audit`, `trivy` (container) |
| Dependency update review | Weekly | Dependabot / Renovate bot |
| Major version upgrades | Quarterly | Manual review + test |
| OS / base image updates | Monthly | Rebuild Docker image from latest patch release |
| Security advisory review | As published | Subscribe to CVE feeds for dependencies used |

### Severity-based patching SLA

| CVE Severity | Patch within |
|---|---|
| Critical (CVSS ≥ 9.0) | 24 hours |
| High (CVSS 7.0–8.9) | 7 days |
| Medium (CVSS 4.0–6.9) | 30 days |
| Low (CVSS < 4.0) | 90 days or next scheduled update |

A Critical CVE in a production dependency is treated as a P2 incident — it gets a dedicated PR, expedited review, and immediate deploy.

### Safe update process

```
For patch/minor versions (1.x.y → 1.x.z):
  1. Update dependency in package.json
  2. Run full test suite
  3. Deploy to staging
  4. Monitor for 24 hours
  5. Deploy to production

For major versions (1.x.y → 2.0.0):
  1. Read migration guide
  2. Check for breaking changes in code
  3. Update + fix breaking changes in a feature branch
  4. Full test suite + manual smoke test
  5. Stage for 1 week
  6. Deploy to production

For security patches (any severity):
  1. Assess: does the CVE affect our usage of the library?
  2. If yes: expedited process — patch + test + deploy within SLA
  3. If no: document why not affected + patch in next regular cycle
```

### Container base image patching

```dockerfile
# Pin to patch version, not major (auto-updates minor security patches)
FROM node:20.15.1-alpine   ← specific patch version

# Rebuild image monthly to pick up OS-level patches
# Automated by CI schedule:
on:
  schedule:
    - cron: '0 2 1 * *'   # 1st of every month at 02:00 UTC
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Never updating dependencies ("if it works, don't touch it") | Vulnerability accumulates; when forced to update, it's a breaking change |
| Updating everything at once | Cannot isolate which update caused a regression |
| No automated vulnerability scanning | Critical CVEs discovered months after publication |
| Ignoring `npm audit` warnings | Warnings that fire on every build are ignored; real CVEs are missed |
| `FROM node:latest` in Dockerfile | Unpredictable base — breaks on major Node release |

---

## 12. Operations Checklist

**Answers:** What should the team be checking on a recurring basis to stay ahead of problems?

### Daily (on-call responsibility)

- [ ] Check overnight alerts — any P3/P4 that needs follow-up?
- [ ] Review error rate dashboard — any new error types or gradual increases?
- [ ] Check queue depth — any backlog building up?
- [ ] Review failed webhook events — any DLQ depth > 0?
- [ ] Confirm all scheduled jobs ran successfully overnight (booking expiry, email digests)

### Weekly (team responsibility)

- [ ] Review dependency update PRs from Renovate/Dependabot
- [ ] Check disk usage trend — still below 70%?
- [ ] Review DB slow query log — any new queries without indexes?
- [ ] Check DB connection pool utilisation trend
- [ ] Review open action items from post-mortems — any overdue?
- [ ] Run backup restore test (can sample a single table if full restore is monthly)
- [ ] Review alert noise — any alert fired > 3 times this week without action?

### Monthly (team responsibility)

- [ ] Full backup restore test to isolated environment
- [ ] Certificate expiry check — any certs expiring in < 60 days?
- [ ] Rotate credentials that are due for rotation
- [ ] Capacity planning review — usage trends for the next 3 months
- [ ] Rebuild and push new base Docker images (OS patch updates)
- [ ] Review and prune feature flags — any that have been fully rolled out and can be removed?
- [ ] Review runbooks — any gaps found in the last month's incidents?
- [ ] Security audit — `npm audit` / `trivy` on all images, review CVE backlog

### Quarterly (team + management)

- [ ] Full disaster recovery drill (restore from backup, measure RTO)
- [ ] Major dependency version review
- [ ] SLA compliance review — did we meet uptime targets?
- [ ] On-call burden review — is the rotation fair? Is burnout a risk?
- [ ] Post-mortem themes review — what patterns keep recurring?
- [ ] Capacity plan for next 6 months

---

## 13. Anti-Patterns

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| No monitoring until something breaks | By the time you notice, users have been affected for hours | Monitoring before the first deploy |
| Monitoring infrastructure but not business metrics | CPU healthy, but "booking confirmed" broken | Add domain-specific metrics |
| Alert on everything | On-call ignores alerts; real P1 buried in noise | Only alert on actionable, urgent conditions |
| One person who knows production | Bus factor of 1; that person cannot take holidays | On-call rotation; runbooks; documented procedures |
| Post-mortem with blame | Engineers hide problems; culture of fear | Blameless post-mortems with system-level fixes |
| Backups never tested | Backup is corrupt; discovered during disaster | Monthly restore tests |
| Runbooks only in someone's head | On-call has no guidance; MTTR is high | Every alert has a written runbook |
| Reactive capacity planning | Scaling starts when the system is already failing | Monthly capacity review with 3-month projections |
| Security patches indefinitely deferred | CVE exploited months after disclosure | SLA-based patching with 24h for Critical |
| Maintenance done in business hours | Maintenance causes brief degradation; users affected | Maintenance windows during lowest-traffic periods |

---

## 14. How This Feeds Back into Development

Operations is not the end of the pipeline — it is the input to the next iteration. Every incident, every alert, every capacity constraint is a signal that feeds back into earlier guides.

| Operations finding | Feeds back into |
|---|---|
| Alert fires with no runbook → on-call lost | Add runbook → link from alert |
| Post-mortem reveals missing test coverage | Testing Guide → add test for the failure mode |
| Slow query found in production | Design Guide → add index; Scaffolding → add to repository layer |
| Feature flag never cleaned up | Code Quality Guide → code smell; add to PR checklist |
| Secret leaked in logs | Code Quality Guide → never log secrets; Scaffolding → logger config |
| Deployment caused downtime | Deployment Guide → improve rollout strategy; add health check |
| Backup restore failed | Operations → fix backup process; add to monthly checklist |
| On-call burnout due to noisy alerts | Alert catalogue → raise thresholds; remove non-actionable alerts |
| Capacity exhausted unexpectedly | Design Guide → add load test; Operations → lower scaling trigger |
| Incident caused by missing observability | Scaffolding Guide → add structured log event; add metric |

### Closing the loop

```
Ideation & Planning
      ↓
    Design
      ↓
  Scaffolding
      ↓
   Testing
      ↓
Code Quality
      ↓
 Deployment
      ↓
 Operations  ──────────────────────────────┐
      │                                    │
      └── incidents, metrics, debt ────────┘
          feed back into the next
          iteration of every guide above
```

Software is never finished. Operations is where you learn what you got wrong — and planning is where you fix it.
