# Tech Stack Selection Guide

Sits between the **Ideation & Planning Guide** (requirements and feasibility complete) and the **Design Guide** (system design starts). You cannot design a system without first choosing the technology it runs on. Every design decision in the next guide is constrained by the choices made here.

Language/framework-agnostic. Running example: **Event booking system**.

---

## 1. Core Principle

**Choose boring technology. The best stack is the one your team knows, that has a proven track record, and that you can hire for.**

New technology carries hidden costs: unknown failure modes, thin documentation, sparse StackOverflow answers, and a smaller hiring pool. A system built on mature, well-understood tools ships faster, runs more reliably, and is easier to hand off. Reserve new technology for problems that existing tools genuinely cannot solve.

---

## 2. Decision Framework

**Answers:** How do you evaluate a technology choice objectively, without bias toward novelty?

### Five evaluation criteria

Score each candidate technology 1–3 on each criterion. Pick the highest total. Ties go to the option with more team familiarity.

| Criterion | 1 — Poor | 2 — Acceptable | 3 — Strong |
|---|---|---|---|
| **Team expertise** | Nobody has used it | 1–2 people have used it | Most of the team knows it |
| **Ecosystem maturity** | < 3 years old, breaking changes frequent | 3–7 years, stable API | 7+ years, battle-tested at scale |
| **Hiring pool** | Rare skill — hard to find | Moderate availability | Widely available |
| **Operational complexity** | Requires significant ops knowledge | Manageable with docs | Managed service available |
| **Licensing & cost** | Commercial, expensive, restrictive | Open source with paid support | Free, permissive license |

### Example evaluation — event booking system database choice

| Criterion | PostgreSQL | MongoDB | DynamoDB |
|---|---|---|---|
| Team expertise | 3 (everyone knows SQL) | 1 (nobody used it) | 1 (nobody used it) |
| Ecosystem maturity | 3 (25+ years) | 2 (15 years, API changes) | 3 (AWS managed) |
| Hiring pool | 3 (universal) | 2 (moderate) | 1 (AWS-specific) |
| Operational complexity | 2 (manageable) | 2 (manageable) | 3 (fully managed) |
| Licensing & cost | 3 (free) | 2 (SSPL — some restrictions) | 1 (pay-per-request, expensive at scale) |
| **Total** | **14** | **8** | **9** |

**Decision: PostgreSQL.**

### The spike rule

Any technology the team has never used in production gets a timebox spike before committing.

```
Spike: Can BullMQ handle our webhook queue requirements?
Timebox: 3 days maximum
Question: Can we enqueue, process, retry, and dead-letter at the throughput we need?
Output: yes/no + discovered gotchas + estimated integration effort
Outcome: commit or reject — no partial adoption
```

Spike code is thrown away. The output is a decision, not a prototype to build on.

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Choosing a technology because it is trending | Trend ≠ production-ready; novelty adds risk, not value |
| Choosing based on conference talks | Speaker's context ≠ your context; their scale ≠ your scale |
| Different language per service on a small team | Context-switching overhead kills velocity; hire for one stack |
| No spike before committing to unknown tech | Unknown unknowns surface mid-implementation |
| Decision made in Slack, never documented | Impossible to remember why; relitigated every 6 months |

---

## 3. Backend Language & Framework

**Answers:** What language and framework runs the server, and why?

### Language decision table

| Language | Strengths | Weaknesses | Use when |
|---|---|---|---|
| **Node.js / TypeScript** | Full-stack JS, huge ecosystem, great async, fast to iterate | Single-threaded CPU-bound perf | API servers, real-time, full-stack teams |
| **Python** | Data science ecosystem, readable, fast prototyping | GIL limits true parallelism, slower than compiled | ML/data pipelines, scripting-heavy backends |
| **Go** | Compiled performance, low memory, great concurrency | Verbose, smaller ecosystem, no generics until recently | High-throughput services, systems programming |
| **Java / Kotlin** | Mature ecosystem, strong typing, enterprise tooling | JVM startup time, verbose boilerplate | Large enterprise systems, existing Java teams |
| **Rust** | Maximum performance, memory safety, no GC pauses | Steep learning curve, slow compile | Systems programming, performance-critical services |

### Framework selection criteria

Once the language is chosen, pick a framework on:

| Criterion | What to look for |
|---|---|
| Routing | Expressive, supports middleware chain, path params |
| Middleware | Standard pattern for auth, validation, error handling |
| Async support | Native async/await, no callback hell |
| ORM ecosystem | Mature ORM with migration support (Prisma, TypeORM, SQLAlchemy) |
| TypeScript support | First-class types, not bolted on |
| Community size | StackOverflow coverage, GitHub activity, release cadence |

### Example — Event booking system

```
Language:  TypeScript (Node.js)
Reason:    Full team knows JavaScript; TypeScript adds safety;
           single language across frontend and backend;
           massive npm ecosystem; Prisma ORM is best-in-class for Node

Framework: Express 5
Reason:    Team familiarity; minimal, composable middleware;
           battle-tested; 14+ years in production;
           no magic — behaviour is explicit and traceable

ORM:       Prisma
Reason:    Type-safe queries generated from schema;
           migration system built in;
           excellent TypeScript integration
```

**ADR-001: TypeScript over JavaScript**
```
Status: Accepted

Context:
  Team ships a multi-role platform with complex domain objects.
  Runtime type errors are a significant source of bugs in untyped JS.

Decision:
  Use TypeScript with strict mode enabled.

Consequences:
  + Compile-time type checking catches entire classes of bugs
  + IDE autocomplete is accurate for domain objects
  - Slightly longer build step
  - New developers need TypeScript familiarity
```

---

## 4. Frontend Stack

**Answers:** What renders the UI, and is it a SPA, SSR, or static site?

### Rendering strategy decision table

| Strategy | How it works | Use when | Avoid when |
|---|---|---|---|
| **SPA (React, Vue, Angular)** | JS renders in browser; API serves JSON | App-like UIs, role-based dashboards, real-time updates | SEO-critical public pages |
| **SSR (Next.js, Nuxt, Remix)** | Server renders HTML per request | SEO matters, content-heavy, fast first paint needed | Highly interactive, real-time app behaviour |
| **Static (Astro, Hugo, Jekyll)** | HTML pre-built at deploy time | Marketing sites, docs, blogs | User-specific content, authenticated pages |
| **Hybrid (Next.js App Router)** | Mix SSR + static + client components | Complex apps needing both SEO and interactivity | Small teams new to the framework |

### Framework comparison

| Framework | Rendering | Learning curve | Ecosystem | TypeScript |
|---|---|---|---|---|
| **React + Vite** | SPA | Low–medium | Largest | Excellent |
| **Next.js** | SSR + SPA hybrid | Medium | Large | Excellent |
| **Vue + Vite** | SPA | Low | Large | Good |
| **Nuxt** | SSR + SPA hybrid | Medium | Medium | Good |
| **Angular** | SPA | High | Large | Native |
| **SvelteKit** | SSR + SPA hybrid | Low | Small | Good |

### Example — Event booking system

```
Framework: React + Vite
Reason:    Role-based dashboards (attendee, organiser, admin) are
           app-like — SEO not needed for authenticated pages;
           team knows React; Vite is fast in development;
           no SSR complexity needed for an internal-facing app

State:     Zustand (auth state) + TanStack Query (server state)
Reason:    Separation of concerns — client state vs server cache;
           TanStack Query handles loading/error/refetch automatically
```

---

## 5. Database Selection

**Answers:** Where does the data live, and what kind of store does each type of data need?

### Primary database decision table

| Database | Model | Use when | Avoid when |
|---|---|---|---|
| **PostgreSQL** | Relational | Structured data, relationships, transactions, reporting | Truly schema-less data (rare) |
| **MySQL** | Relational | Same as Postgres; more common in legacy stacks | New greenfield projects (Postgres is strictly better) |
| **MongoDB** | Document | Truly variable schema, no relationships, JSON-native API | You actually have relationships — use Postgres |
| **SQLite** | Relational (embedded) | Local dev, single-user apps, testing | Multi-process writes, production scale |
| **DynamoDB** | Key-value / document | Massive scale, simple access patterns, AWS-native | Complex queries, reporting, joins |

**Default rule: start with PostgreSQL.** If you later discover you need a different store, you will have a proven, working system to migrate from. Pre-optimising for a database you might need is waste.

### Secondary stores

Only add a secondary store when PostgreSQL cannot satisfy a specific requirement.

| Store | Purpose | Add it when |
|---|---|---|
| **Redis** | Cache, session store, pub/sub, job queue | You need sub-millisecond reads, distributed locking, or a message queue |
| **S3 / object storage** | Files, images, exports | Users upload files or you generate large binary output |
| **Elasticsearch / Typesense** | Full-text search | Postgres `ILIKE` is too slow for your search volume |
| **InfluxDB / TimescaleDB** | Time-series metrics | You are storing sensor data or high-frequency telemetry |

### Example — Event booking system

```
Primary:   PostgreSQL 16
Reason:    Events, bookings, users, waitlists — all relational;
           transactions critical (seat reservation);
           Postgres row-level locking solves race conditions

Secondary: Redis (BullMQ queue + idempotency key store)
Reason:    Webhook processing queue; idempotency cache (24h TTL);
           Redis TTL auto-expiry fits the use case perfectly

Files:     S3 (QR ticket images)
Reason:    Binary files do not belong in Postgres;
           S3 is cheap, durable, and CDN-friendly
```

**ADR-002: PostgreSQL over MongoDB**
```
Status: Accepted

Context:
  Team evaluated MongoDB for schema flexibility (events have varying fields).
  After mapping the domain, every entity has a clear, stable schema.
  Bookings have hard transactional requirements (seat reservation).

Decision:
  Use PostgreSQL. JSONB column added to Event for optional metadata fields
  where true schema flexibility is needed.

Consequences:
  + ACID transactions for seat reservation
  + Strong referential integrity
  + Familiar SQL tooling for the team
  - Less flexible for truly variable schemas (mitigated by JSONB)
```

---

## 6. Infrastructure & Hosting

**Answers:** Where does the application run, and how is it managed?

### Hosting decision table

| Option | Control | Ops burden | Cost | Use when |
|---|---|---|---|---|
| **Bare metal / VPS** | Full | High | Low | You have dedicated ops; cost is primary constraint |
| **Managed cloud (AWS/GCP/Azure)** | High | Medium | Medium–high | Scale matters; team has cloud experience |
| **PaaS (Railway, Render, Fly.io)** | Low | Very low | Medium | Small team; move fast; no dedicated DevOps |
| **Serverless (Lambda, Cloud Run)** | Low | Low | Pay-per-use | Spiky traffic, event-driven, no persistent connections |

### Container orchestration

```
Docker:     Always — consistent dev/prod parity, reproducible builds
Kubernetes: Only when you have > 5 services OR > 10 instances to manage
            Kubernetes is powerful and complex — do not adopt it for simplicity

Small team alternative:
  Docker Compose (local) + Railway/Render/Fly.io (production)
  → Zero Kubernetes operational overhead
  → Full containerisation benefits
  → Scales to moderate traffic before needing more
```

### Example — Event booking system

```
Small team path (< 5 engineers):
  Local:       Docker Compose (Postgres + Redis + app)
  Production:  Railway or Render
    → Managed Postgres, managed Redis
    → Deploy from Docker image
    → No Kubernetes complexity
    → Auto-scaling to moderate traffic

Scaling path (if traffic demands it):
  AWS ECS (Fargate) — managed container orchestration without full Kubernetes
  RDS (managed Postgres) + ElastiCache (managed Redis)
```

---

## 7. Third-Party Services

**Answers:** What do you buy instead of build, and why?

### Buy vs build rule

**If it is not your core business, buy it.** Building payment processing, email delivery, or authentication from scratch is months of work that does not differentiate your product. Buy commodity functionality; build differentiated functionality.

### Service catalogue

| Category | Recommended options | Self-host when |
|---|---|---|
| **Payments** | Stripe, Paddle, Adyen | Almost never — PCI compliance is a full-time job |
| **Email delivery** | Resend, SendGrid, Postmark | Almost never — deliverability is a specialised problem |
| **Transactional auth (OAuth)** | Auth0, Clerk, Supabase Auth | You have strict data residency requirements |
| **File storage** | S3, Cloudflare R2, GCS | Almost never — object storage is commodity |
| **Job queue** | BullMQ (Redis-backed), SQS, Inngest | BullMQ is the right default for Node; no separate infra needed |
| **Error tracking** | Sentry, Honeybadger | Sentry has a generous free tier; self-host only for compliance |
| **Metrics & dashboards** | Grafana Cloud, Datadog, New Relic | Grafana OSS if you have ops bandwidth |
| **Feature flags** | LaunchDarkly, Unleash, GrowthBook | Unleash OSS for cost-sensitive teams |
| **Search** | Typesense, Algolia | Typesense is self-hostable and excellent for most use cases |

### Example — Event booking system

```
Payments:       Stripe
  Why: PCI-DSS SAQ A compliance, webhook ecosystem, test mode

Email:          Resend
  Why: Developer-friendly API, React Email for templates, generous free tier

Queue:          BullMQ (backed by Redis already in the stack)
  Why: No additional infrastructure; Redis already present for idempotency

Error tracking: Sentry (free tier)
  Why: First error tracking, source maps, performance tracing included

Metrics:        Prometheus + Grafana (self-hosted)
  Why: Team has ops bandwidth; avoids Datadog cost at early scale
```

---

## 8. ADR — Recording Stack Decisions

**Answers:** How do you document technology choices so future team members understand why, not just what?

Every non-obvious stack choice needs an ADR (Architecture Decision Record). "Non-obvious" means: a reasonable engineer could have chosen differently.

### ADR template

```markdown
# ADR-{number}: {title}

**Status:** Proposed / Accepted / Deprecated / Superseded by ADR-{n}
**Date:** YYYY-MM-DD
**Deciders:** @username1, @username2

## Context
What is the situation that requires a decision?
What constraints exist (team size, timeline, existing systems)?

## Decision
What was chosen and why?
What alternatives were considered and rejected?

## Consequences
Positive: what does this choice enable?
Negative: what does this choice cost or prevent?
Neutral: what changes as a result?
```

### Example — Event booking system

```markdown
# ADR-003: Use BullMQ over SQS for the webhook processing queue

**Status:** Accepted
**Date:** 2025-05-01
**Deciders:** @vseel5, @team-lead

## Context
Stripe webhooks must be processed asynchronously with retry logic
and dead-letter handling. Two options evaluated: BullMQ (Redis-backed,
self-managed) and AWS SQS (managed, pay-per-message).

## Decision
Use BullMQ. Redis is already in the stack for idempotency key storage.
Adding BullMQ adds no new infrastructure. SQS would require AWS SDK,
IAM configuration, and a second messaging system alongside Redis.

## Consequences
+ No additional infrastructure or cloud cost
+ BullMQ dashboard available for queue monitoring
+ Retry and dead-letter logic built-in
- Redis is now a single point of failure for both idempotency and queuing
- If Redis goes down, webhooks stop processing (mitigated by Redis HA)
```

### ADR storage

```
docs/
  adr/
    ADR-001-typescript-over-javascript.md
    ADR-002-postgresql-over-mongodb.md
    ADR-003-bullmq-over-sqs.md
```

ADRs are committed to the repository. They are never deleted — deprecated ADRs keep their record and are marked superseded.

---

## 9. Stack Decision Checklist

Run this before moving to the Design Guide.

**For every technology chosen:**
- [ ] Team familiarity assessed — at least 2 people can operate it without googling basics
- [ ] Proven in production elsewhere — not a first-GA release
- [ ] Hiring pool checked — can you find engineers who know this?
- [ ] Operational complexity understood — who manages it in production?
- [ ] Licensing reviewed — no surprise commercial restrictions
- [ ] Spike completed for any technology nobody has used before (≤ 3 days)
- [ ] ADR written for every non-obvious choice

**Stack coherence:**
- [ ] Single backend language across all services (no polyglot on small teams)
- [ ] Consistent ORM across all services
- [ ] Secondary stores only added where primary cannot satisfy the requirement
- [ ] Third-party services chosen for commodity functions (payments, email, auth)

---

## 10. Anti-Patterns

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| Choosing the newest framework | Novelty adds hidden cost: thin docs, unknown failure modes, no StackOverflow | Score against evaluation criteria; novelty is not a criterion |
| Microservices from day one | Distributed systems complexity without the scale to justify it | Start monolith; extract services when a specific boundary demands it |
| Different language per service (small team) | Every context switch is overhead; hiring becomes harder | One primary language; switch only for a compelling, specific reason |
| MongoDB "for flexibility" | You almost always have a relational domain; flexibility becomes inconsistency | Map your domain; use Postgres + JSONB for genuinely variable fields |
| Kubernetes for 2 services | Ops overhead without benefit; consumes engineering time that should go to features | Docker + PaaS until you have 5+ services or 10+ instances |
| Self-hosting email | Email deliverability is a solved, specialised problem | Use Resend, SendGrid, Postmark — not your own SMTP server |
| No ADR — decision made in Slack | Impossible to recall why; relitigated every time someone joins | Write the ADR at decision time, not after |
| Building auth from scratch | Auth is complex, security-critical, and not your core business | Use a managed auth service or a mature library |

---

## 11. How This Feeds into Design

Every stack decision in this guide creates a constraint in the Design Guide. The design must work within the chosen stack, not despite it.

| Stack decision | Design constraint |
|---|---|
| PostgreSQL chosen | Design Guide uses relational ERD, SQL schema, row-level locking for race conditions |
| Redis in stack | Idempotency keys stored in Redis; BullMQ queue backed by Redis |
| React SPA | API Design must return JSON — no server-rendered HTML |
| Stripe for payments | Integration Design uses Stripe payment intents + webhook pattern |
| BullMQ for queues | Integration Design uses Redis-backed message schema; worker pattern |
| TypeScript + Prisma | Database Design schema maps directly to Prisma schema.prisma syntax |
| JWT auth chosen | Security Design uses stateless JWT; no session store needed |
| PaaS hosting (Railway) | Deployment Guide targets Docker image + env vars; no Kubernetes config |
