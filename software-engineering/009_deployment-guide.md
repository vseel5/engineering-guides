# Deployment & CI/CD Guide

Sits after the **Scaffolding**, **Testing**, and **Code Quality** guides. Feeds into the **Operations Guide** (monitoring, alerting, on-call). This guide covers how code gets from a developer's machine to production reliably and repeatably.

Language/framework-agnostic. Running example: **Event booking system**.

---

## 1. Core Principle

**Code is not shipped until it is deployed. Deploy often, deploy small.**

A feature merged to main but not deployed has unknown production behaviour. Small, frequent deploys reduce risk: less to review, less to break, less to roll back. The goal is to make deploying so routine and safe that it is boring.

---

## 2. Pipeline Stages

**Answers:** What happens between a developer pushing code and it running in production?

### Order

```
Lint → Test → Build → Package → Deploy to Staging → Deploy to Production
```

Each stage is a gate. A stage failure stops the pipeline. Nothing skips a stage.

| Stage | What it does | Fails on | Approx. time |
|---|---|---|---|
| **Lint** | Static analysis, formatting check | Lint errors, formatting diff | < 1 min |
| **Test** | Unit + integration tests, coverage gate | Test failure, coverage below threshold | < 10 min |
| **Build** | Compile, type-check, bundle | Type errors, compile errors | < 3 min |
| **Package** | Build Docker image, tag it, push to registry | Image build failure | < 5 min |
| **Deploy to Staging** | Roll out new image to staging environment | Health check failure, smoke test failure | < 5 min |
| **Deploy to Production** | Roll out to production after approval | Health check failure | < 5 min |

### Stage dependencies

```
Lint ─┐
Test ─┼─→ Build → Package → Deploy Staging → [manual gate] → Deploy Production
      │
      └─ (Lint and Test can run in parallel — neither depends on the other)
```

Lint and test are independent — run them in parallel to save time. Build only starts when both pass.

### Example — Event booking system

```yaml
# Pipeline triggered on: push to any branch (CI), merge to main (CD)

stages:
  - lint_and_test:      # parallel
      lint:   eslint + prettier --check
      test:   jest --coverage (unit + integration against test DB)

  - build:              # after lint_and_test passes
      tsc --noEmit      # type check
      vite build        # or webpack, esbuild, etc.

  - package:            # after build passes
      docker build -t event-booking:${GIT_SHA} .
      docker push registry/event-booking:${GIT_SHA}

  - deploy_staging:     # after package passes, automatic
      kubectl set image deployment/event-booking app=event-booking:${GIT_SHA}
      wait for health check

  - deploy_production:  # after staging passes, manual approval required
      [approval gate]
      kubectl set image deployment/event-booking app=event-booking:${GIT_SHA}
      wait for health check
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Stages run sequentially when they could be parallel | Unnecessary wait time; developers context-switch |
| Skipping stages "just this once" | "Just this once" becomes the norm; the gate is meaningless |
| No timeout per stage | A hung test suite blocks the pipeline indefinitely |
| Build artifact created twice (once for staging, once for prod) | Different artifacts = untested code in production |

---

## 3. Environment Promotion

**Answers:** What are the environments, what is each one for, and what data lives in each?

### Environment chain

```
Local dev → Dev → Staging → Production
```

| Environment | Purpose | Data | Who deploys | How often |
|---|---|---|---|---|
| **Local** | Individual development, fast iteration | Seeded fake data | Developer | Continuously |
| **Dev** | Integration between developers, shared branch testing | Seeded fake data, reset on demand | CI on branch push | Many times per day |
| **Staging** | Pre-production validation, QA, stakeholder review | Anonymised copy of production data (or realistic seed) | CI on merge to main | On every merge |
| **Production** | Real users, real data | Live data | CD pipeline after approval | Scheduled or on-demand |

### Rules per environment

**Local:**
- Developer owns this entirely
- Can break freely — no one else is affected
- Uses `.env` file (never committed)
- DB is local or a personal dev container

**Dev:**
- Shared — breaking dev breaks everyone on the team
- Merges to a `develop` branch trigger deploys here
- DB is reset to seed state on every deploy
- Not shown to stakeholders

**Staging:**
- Must mirror production configuration exactly (same image, same env vars structure, same DB engine)
- Uses anonymised production data or a realistic seed — never real PII
- Stakeholders review here before production approval
- E2E tests run here after deploy

**Production:**
- Real users — every deploy is a risk
- Manual approval gate before deploy
- Change management log for every deploy (what, who, when, why)
- Never use as a debugging environment

### Data rules

```
Production data NEVER flows to dev or local.
Staging may use anonymised production data — never raw PII.
Dev and local always use synthetic seed data.
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Staging and production have different configs | Staging passes, production fails — defeats the purpose of staging |
| Real user data in staging | PII exposure; developers see sensitive data they shouldn't |
| No staging — deploy straight from dev to prod | No pre-production validation; every deploy is a gamble |
| Staging DB rarely reset | State accumulates; staging drifts from production; tests fail for wrong reasons |

---

## 4. CI Pipeline (on every commit / PR)

**Answers:** What runs automatically on every pushed commit, and what must pass before a PR can merge?

### Trigger: every push to every branch

```
developer pushes → CI triggers immediately → result posted to PR within 15 minutes
```

### What runs

```yaml
on: [push, pull_request]

jobs:
  lint:
    timeout-minutes: 5
    steps:
      - prettier --check "src/**/*.ts"
      - eslint "src/**/*.ts"

  test:
    timeout-minutes: 15
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_DB: test_db, POSTGRES_PASSWORD: test }
    steps:
      - npm run prisma:migrate      # apply migrations to test DB
      - npx ts-node tests/seed.ts  # seed baseline data
      - jest --coverage             # unit + integration
      - coverage gate: 80% lines, 75% branches

  build:
    needs: [lint, test]             # only runs if both pass
    timeout-minutes: 10
    steps:
      - tsc --noEmit
      - npm run build
```

### What blocks merge (required checks)

| Check | Blocks merge? |
|---|---|
| Lint passes | Yes |
| Type check passes | Yes |
| Unit tests pass | Yes |
| Integration tests pass | Yes |
| Coverage above threshold | Yes |
| Build succeeds | Yes |
| At least 1 approving review | Yes |
| E2E tests pass | No — runs after merge (too slow for PR) |
| Deploy to staging succeeds | No — runs after merge |

### Timeouts

Every CI job must have an explicit timeout. A hung job without a timeout blocks the pipeline forever.

```
Lint:        5 minutes
Test:        15 minutes
Build:       10 minutes
Package:     10 minutes
Deploy:      15 minutes (including health check wait)
```

If a job consistently runs near its timeout, the timeout is too low — investigate and fix, do not just raise it.

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| No timeout on jobs | Hung test suite blocks pipeline for hours |
| E2E tests on every PR | 20-minute feedback loop; developers stop pushing small commits |
| Required checks that are always green | Meaningless gates; team learns to ignore them |
| CI only runs on `main` | Bugs discovered at merge, not at commit — expensive to fix |

---

## 5. CD Pipeline (on merge to main)

**Answers:** How does merged code get to staging and production automatically and safely?

### Trigger: merge to main

```
PR merged → Build + Package → Auto-deploy to staging → E2E on staging
          → [Manual approval] → Deploy to production → Health check
```

### Automatic staging deploy

```yaml
on:
  push:
    branches: [main]

jobs:
  package:
    steps:
      - docker build -t registry/event-booking:${GITHUB_SHA} .
      - docker build -t registry/event-booking:latest .
      - docker push registry/event-booking:${GITHUB_SHA}
      - docker push registry/event-booking:latest

  deploy_staging:
    needs: package
    environment: staging
    steps:
      - kubectl set image deployment/event-booking app=registry/event-booking:${GITHUB_SHA}
      - kubectl rollout status deployment/event-booking --timeout=5m
      - run smoke tests against staging
      - run E2E suite against staging
```

### Manual production gate

Automatic to staging. Manual approval to production. This is not bureaucracy — it is the last human checkpoint before real users are affected.

```yaml
  deploy_production:
    needs: deploy_staging
    environment:
      name: production
      url: https://eventbooking.com
    # GitHub / GitLab: requires manual approval from a designated reviewer
    steps:
      - kubectl set image deployment/event-booking app=registry/event-booking:${GITHUB_SHA}
      - kubectl rollout status deployment/event-booking --timeout=5m
      - run production smoke test (health endpoint only)
```

### Feature flags — deploy without releasing

High-risk changes should reach production turned off. Code is deployed; the feature is not visible until the flag is enabled. This decouples deploy from release.

```ts
// Feature flag check — flag evaluated at runtime, not build time
if (featureFlags.isEnabled('waitlist-v2', userId)) {
  return await waitlistServiceV2.add(input);
}
return await waitlistService.add(input);
```

Benefits:
- Instant rollback without a redeploy: disable the flag
- Gradual rollout: enable for 5% of users, then 25%, then 100%
- Test in production against real traffic without affecting all users

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Auto-deploy directly to production | No human checkpoint; one bad merge goes live immediately |
| Manual deploy to staging | Staging is skipped under pressure; production gets untested code |
| No E2E on staging before production approval | Staging exists to catch what CI cannot — skipping it makes staging pointless |
| Feature flags never cleaned up | Flag logic accumulates; codebase becomes a maze of dead branches |

---

## 6. Containerisation

**Answers:** How is the application packaged, and what makes a container production-ready?

### Dockerfile rules

**1. Start from a minimal base image**
```dockerfile
# Bad — 1.1GB image with everything installed
FROM node:20

# Good — 180MB image, production dependencies only
FROM node:20-alpine
```

**2. Layer caching — dependencies before source code**

Docker rebuilds from the first changed layer. Put what changes least at the top.

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app

# Copy package files first — cached unless dependencies change
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Copy source last — changes on every commit, but layer above is cached
COPY . .
RUN npm run build
```

**3. Multi-stage build — dev tools never reach the final image**

```dockerfile
# Stage 1: build (includes dev dependencies, TypeScript compiler)
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci                          # all deps including devDependencies
COPY . .
RUN npm run build                   # compile TypeScript → dist/

# Stage 2: production image (only compiled output + prod deps)
FROM node:20-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production        # production deps only
COPY --from=builder /app/dist ./dist
EXPOSE 3000
USER node                           # non-root
CMD ["node", "dist/server.js"]
```

**4. Non-root user — always**

```dockerfile
# Bad — process runs as root inside the container
CMD ["node", "dist/server.js"]

# Good — process runs as an unprivileged user
USER node
CMD ["node", "dist/server.js"]
```

A process running as root inside a container is root on the host if the container escapes. There is no valid reason for a web server to run as root.

**5. No secrets in the image**

```dockerfile
# Bad — secret baked into the image layer, visible in docker history
ENV STRIPE_SECRET_KEY=sk_live_xxxx
RUN npm install

# Good — injected at runtime, never in the image
# (see Secrets Management section)
```

### Image tagging strategy

```
registry/event-booking:{git-sha}    ← immutable, traceable to exact commit
registry/event-booking:latest       ← always points to most recent main build
registry/event-booking:1.4.2        ← semantic version tag for releases (optional)
```

**Rules:**
- Always deploy by git SHA — not `latest`. `latest` is ambiguous; SHA is exact.
- `latest` tag is for convenience (local pulls, quick reference) — never for production deploys
- Never overwrite an existing SHA tag — immutable artifact, immutable tag

```bash
# Bad — what commit is this?
kubectl set image deployment/event-booking app=registry/event-booking:latest

# Good — exact, auditable, rollback-safe
kubectl set image deployment/event-booking app=registry/event-booking:a3f8c12
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| `FROM ubuntu` or `FROM node` (full image) | 1GB+ images; slow pull times; large attack surface |
| Source code before dependencies in Dockerfile | Cache miss on every source change rebuilds all deps |
| Running as root | Security risk; unnecessary privilege |
| Secrets as ENV in Dockerfile | Secret is in every layer of the image history |
| Deploying `:latest` in production | `latest` can point to a different commit per-pull; non-deterministic |
| Dev dependencies in production image | Larger image; unnecessary packages are an attack surface |

---

## 7. Secrets Management

**Answers:** Where do secrets live, how do they get into the running application, and what never happens?

### Where secrets must NOT be

```
❌ Committed .env files
❌ Docker image layers (ENV in Dockerfile)
❌ CI/CD logs (never echo a secret)
❌ Application logs
❌ Build artifacts
❌ Source code (no matter how "temporary")
❌ Slack, email, or any communication tool
```

If a secret was ever committed to git, rotate it immediately — even if the commit was reverted. The secret is in git history.

### Where secrets live

| Secret type | Where to store | How to access |
|---|---|---|
| API keys, DB passwords, JWT secrets | Cloud secrets manager | Injected as env vars at container start |
| CI/CD pipeline secrets | CI platform secrets (GitHub Secrets, GitLab CI Variables) | Available as env vars during pipeline only |
| Local development | `.env` file (git-ignored) | Loaded by dotenv — never committed |
| Kubernetes deployments | Kubernetes Secrets or external secrets operator | Mounted as env vars or files |

### How secrets are injected at runtime (not build time)

```yaml
# Kubernetes — secret defined in the cluster, not in the image
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: event-booking
          image: registry/event-booking:a3f8c12
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: event-booking-secrets
                  key: database-url
            - name: STRIPE_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: event-booking-secrets
                  key: stripe-secret-key
```

The image contains zero secrets. The running container receives them at start from the secrets manager. A stolen image reveals nothing.

### Secret rotation

```
1. Generate new secret value
2. Add new value to secrets manager alongside old value (dual-write period)
3. Deploy application — it picks up both values (or the new one)
4. Verify no errors
5. Remove old value from secrets manager
6. Revoke old secret at the provider (Stripe, DB, etc.)
```

Never rotate by: stop service → update → start service. This causes downtime. Dual-write first.

### `.env.example` — always committed

```bash
# .env.example — committed, no real values
DATABASE_URL=postgresql://user:password@localhost:5432/event_booking
JWT_SECRET=replace-with-32-char-minimum-secret
STRIPE_SECRET_KEY=sk_test_replace_with_real_key
STRIPE_WEBHOOK_SECRET=whsec_replace_with_real_secret
EMAIL_FROM_ADDRESS=noreply@yourdomain.com
```

`.env.example` documents every variable the application needs. A new developer copies it, fills in values, and runs the app. No tribal knowledge required.

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| `.env` committed to git | Secret is in history forever; rotate immediately if this happens |
| Secrets in CI/CD environment variables printed in logs | Anyone with log access has the secret |
| Hardcoded secrets "for testing" | Test secrets become prod secrets; no one cleans up |
| Secrets stored in the image | Stolen image = stolen credentials |
| No secret rotation process | Credentials leaked months ago are still valid |

---

## 8. Database Migrations in CI/CD

**Answers:** When do migrations run relative to code deployment, and how are they kept safe?

### The rule: migrations run before new code starts

```
Old code running
       ↓
Run migration (DB is now in new schema state)
       ↓
New code deploys (expects new schema — it's there)
       ↓
Old code is replaced by new code
```

If you deploy new code first, the new code runs against the old schema and crashes.

### The problem: old code breaks on new schema

This is where "run migrations before new code" breaks down for destructive changes.

**Scenario:** rename column `status` → `booking_status` in the same deploy.

```
Step 1: run migration (rename column)
Step 2: during rolling deploy, old pods are still running
Step 3: old pods try to query column `status` → column does not exist → crash
```

### Solution: Expand / Contract pattern

Never rename or drop in the same deploy as the code that removes the old reference. Use two deploys.

```
Deploy 1 — Expand:
  - Migration: ADD COLUMN booking_status (copy of status)
  - Code: writes to BOTH status AND booking_status
           reads from booking_status (new column)

Deploy 2 — Contract (after Deploy 1 is stable):
  - Migration: DROP COLUMN status
  - Code: only references booking_status

Between deploys: both columns exist; old and new code are compatible.
```

**Expand/contract applies to:**
- Renaming columns
- Dropping columns
- Changing column types
- Renaming tables

**Does NOT need expand/contract:**
- Adding a new nullable column (old code ignores it)
- Adding a new table (old code doesn't know it exists)
- Adding an index (transparent to application code)

### Migration in the pipeline

```yaml
deploy_staging:
  steps:
    - pull new image
    - run migrations:   kubectl exec migration-job -- npm run prisma:migrate
    - verify migration: check exit code 0
    - deploy new code:  kubectl set image deployment/...
    - health check
```

Migrations run as a separate job, not as part of application startup. If the migration fails, the deploy stops — new code is never started.

### Rollback strategy

```
Rollback scenario:
  Current: v1.4.2 running, migration 20250601 applied
  Problem: v1.4.2 has a critical bug
  Action:  Roll back to v1.4.1

For additive migrations (new columns, new tables):
  → Roll back the code image only
  → Old code ignores new columns
  → Schema rollback is optional

For destructive migrations (dropped/renamed columns):
  → Cannot roll back schema without restoring data
  → Roll back requires: restore DB from backup + redeploy old image
  → This is why destructive migrations need expand/contract — avoid this situation
```

**One-command rollback (code):**
```bash
# Roll back to the previous known-good image
kubectl set image deployment/event-booking app=registry/event-booking:{previous-sha}
kubectl rollout status deployment/event-booking
```

**One-command rollback (schema — only if non-destructive):**
```bash
npm run prisma:migrate rollback  # or equivalent for your ORM
```

### Rules

- [ ] Migrations are always additive in the same deploy as the code that needs them
- [ ] Destructive changes (drop/rename) use expand/contract across two deploys
- [ ] Migration script exits non-zero on failure — pipeline stops
- [ ] Never edit an existing migration file — always create a new one
- [ ] Rollback plan documented before the migration is merged
- [ ] Test rollback in staging before production

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Deploy new code, then run migrations | New code runs against old schema; crashes immediately |
| Drop/rename column in the same deploy as code change | Old pods crash during rolling deploy |
| Migrations run at application startup | Multiple pods run migrations concurrently; race conditions, duplicate runs |
| No rollback tested | Rollback discovered to be broken during an incident — worst time |
| Edit existing migration files | Other developers have already run the old version; DB state diverges |

---

## 9. Rollback Strategy

**Answers:** What happens when a bad deploy reaches production, and how fast can you recover?

### When to roll back vs roll forward

| Situation | Action | Reason |
|---|---|---|
| Critical bug affecting all users | Roll back immediately | Fastest path to stability |
| Bug affecting < 5% of users (behind feature flag) | Disable feature flag | No deploy needed; instant |
| Bug with a simple, known fix | Roll forward with a hotfix | Rollback + redeploy = same time |
| Data corruption | Roll back + restore backup | Code rollback alone does not fix data |
| Performance degradation | Roll back while investigating | User experience over root cause |

**Default:** roll back first, investigate after. A working v1.4.1 is better than a broken v1.4.2 while debugging.

### Code rollback

Every deploy records what it replaced. Rollback is a re-deploy of the previous image.

```bash
# One command — deploy the previous known-good SHA
kubectl set image deployment/event-booking \
  app=registry/event-booking:{previous-sha}

# Verify rollout
kubectl rollout status deployment/event-booking --timeout=5m

# Confirm health
curl https://staging.eventbooking.com/api/v1/health
```

This completes in under 5 minutes. The target is < 15 minutes from "incident detected" to "stable version running."

### Schema rollback

Schema rollback is only possible for non-destructive migrations. Destructive migrations (DROP COLUMN, DROP TABLE) cannot be undone without a data restore.

```
Additive migration (safe to roll back):
  Code rollback: redeploy old image — old code ignores new columns
  Schema rollback: optional, run down migration

Destructive migration (cannot roll back schema):
  If the column was dropped: data is gone
  Only option: restore DB from last backup before the migration
  Expected data loss: time between backup and incident
```

This is why the expand/contract pattern exists. A destructive migration that needs rollback requires restoring a backup — a multi-hour, high-risk operation.

### Rollback runbook (document before every deploy)

```markdown
## Rollback: Event booking v1.4.2

Previous stable version: v1.4.1 (sha: a3f8c12)
Migration applied: 20250601_add_waitlist_position_index.sql (additive — safe to roll back)

Code rollback:
  kubectl set image deployment/event-booking app=registry/event-booking:a3f8c12

Schema rollback (if needed):
  npx prisma migrate resolve --rolled-back 20250601_add_waitlist_position_index

Verify:
  curl https://eventbooking.com/api/v1/health → 200
  Manual smoke test: create booking, confirm payment flow

Escalate to: @oncall-lead if rollback does not stabilise within 15 minutes
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| No rollback plan before deploying | Rollback plan written during an incident — wrong time to think clearly |
| Rollback plan never tested | Discovered to be wrong at 2am during an incident |
| Waiting to roll back while debugging | Every minute of debugging is a minute users are affected |
| Rollback by deploying a new commit | Adds another untested change on top of the broken one |

---

## 10. Zero-Downtime Deploys

**Answers:** How does new code replace old code without users seeing errors or dropped requests?

### The problem

A naive deploy: stop old container, start new container. Gap between stop and start = downtime.

### Health checks

Kubernetes (and most orchestrators) use two health checks:

| Check | Name | Purpose | What it tests |
|---|---|---|---|
| **Liveness** | Is the process alive? | Restart a crashed pod | Process is running and not deadlocked |
| **Readiness** | Is the pod ready for traffic? | Remove from load balancer until ready | DB connected, cache warm, migrations complete |

```yaml
# Kubernetes health check config
livenessProbe:
  httpGet:
    path: /api/v1/health/live
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3      # 3 failures → restart pod

readinessProbe:
  httpGet:
    path: /api/v1/health/ready
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2      # 2 failures → remove from load balancer
```

```ts
// /api/v1/health/live — is the process running?
app.get('/health/live', (req, res) => {
  res.status(200).json({ status: 'alive' });
});

// /api/v1/health/ready — is it ready to receive traffic?
app.get('/health/ready', async (req, res) => {
  try {
    await db.$queryRaw`SELECT 1`;     // DB reachable
    res.status(200).json({ status: 'ready' });
  } catch {
    res.status(503).json({ status: 'not ready', reason: 'db unreachable' });
  }
});
```

Traffic only reaches a pod when its readiness probe passes. During deploy, new pods become ready before old pods are stopped.

### Graceful shutdown

The Scaffolding Guide covers the graceful shutdown implementation. In deployment context: the orchestrator sends `SIGTERM` when removing a pod from the rotation. The process must finish in-flight requests before exiting. Configure `terminationGracePeriodSeconds` to be longer than your longest expected request.

```yaml
spec:
  terminationGracePeriodSeconds: 60   # must exceed graceful shutdown timeout (30s in Scaffolding)
```

### Rolling update

The default deploy strategy. New pods come up one at a time; old pods are removed after new ones are ready.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1           # one extra pod above desired count during deploy
    maxUnavailable: 0     # no pods removed until replacement is ready
```

With `maxUnavailable: 0`, at least N pods are always serving traffic. No downtime.

### Blue-green deploy

Run two identical production environments (blue = current, green = new). Switch the load balancer from blue to green atomically. Rollback = switch back to blue.

```
Load Balancer → Blue (v1.4.1, running)

Deploy:
  1. Start Green (v1.4.2) — runs in parallel with Blue
  2. Run health checks and smoke tests on Green
  3. Switch load balancer: → Green (v1.4.2)
  4. Blue stays running (idle) for 15 minutes — instant rollback if needed
  5. Decommission Blue

Rollback:
  Switch load balancer back to Blue — < 30 seconds
```

Blue-green is more expensive (double the infra during deploy) but gives the cleanest rollback story.

| Strategy | Rollback speed | Cost | Complexity | Use when |
|---|---|---|---|---|
| Rolling update | Moderate (re-deploy old) | Low | Low | Standard deploys |
| Blue-green | Instant (switch LB) | High (double infra) | Medium | High-risk releases |
| Canary | N/A (disable canary) | Low | Medium | Large traffic, risk-averse |

### Canary deploy

Send a small percentage of traffic to the new version. Observe metrics. Expand gradually.

```
100% → v1.4.1

Canary start:
  5%  → v1.4.2
  95% → v1.4.1

After 30 minutes with no errors:
  25%  → v1.4.2
  75%  → v1.4.1

After 1 hour stable:
  100% → v1.4.2
  0%   → v1.4.1 (decommission)
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| No readiness probe | Pod receives traffic before DB connection is established; requests fail |
| Liveness probe checks DB | DB slowdown → pod restarts → DB overloaded → cascade failure |
| `terminationGracePeriodSeconds` < shutdown timeout | Pods killed mid-request; requests return 502 |
| `maxUnavailable: 1` without `maxSurge` | One pod always down during deploy; reduced capacity |
| Blue-green without DB compatibility | Green code runs against Blue's DB schema — may not be compatible |

---

## 11. Deployment Checklist (before merging a feature)

Run this before raising a PR that will go to production.

**Code:**
- [ ] Feature is behind a feature flag if it is high-risk or incomplete
- [ ] No secrets hardcoded — all config comes from env vars
- [ ] Observability added: structured log events for new state changes (see Operations guide for detail)
- [ ] Health check endpoints still return 200 with the new code

**Database:**
- [ ] Migration is additive, OR expand/contract plan is documented across two PRs
- [ ] Migration has been tested on a copy of the staging DB
- [ ] Rollback plan for the migration is written and tested
- [ ] No `DROP COLUMN` / `DROP TABLE` / `RENAME` in the same deploy as code that removes the reference

**Deployment:**
- [ ] Rollback plan written: previous image SHA, rollback command, verification steps
- [ ] Staging deploy succeeded and E2E tests passed
- [ ] Stakeholder sign-off received (if user-facing change)
- [ ] Canary or ramp plan documented for high-risk changes

**After deploy to production:**
- [ ] Smoke test run manually against production
- [ ] No spike in error rate or latency (check dashboards — see Operations guide)
- [ ] Feature flag enabled for a subset of users first (if applicable)

---

## 12. Anti-Patterns

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| Manual deploys to production | Not repeatable; no audit trail; "works on my machine" | Automate — only the pipeline deploys to production |
| Secrets in build artifacts or image layers | Stolen image = stolen credentials | Inject at runtime via secrets manager |
| Deploying migrations after new code starts | New code queries columns that don't exist yet → crash | Migrations always run before new code |
| No rollback tested | Discovered to be broken at 2am during an incident | Test rollback in staging before every production deploy |
| Deploying `:latest` image tag in production | Non-deterministic; different machines pull different code | Always deploy by git SHA |
| Running as root in containers | Container escape = root on host | `USER node` in Dockerfile |
| No health checks | Orchestrator routes traffic to pods that aren't ready | Liveness + readiness probes on every service |
| All deploys are high-risk | Fear of deploying → infrequent deploys → larger changes → more risk | Feature flags + small PRs + automated pipeline = routine deploys |
| Staging skipped under deadline pressure | Production gets untested code | Staging deploy is automatic and mandatory; it cannot be bypassed |
| No DB migration rollback plan | Migration fails in prod; no path back | Write rollback plan before merge; test it in staging |

---

## 13. How This Feeds into Operations

Deployment is the boundary between engineering and operations. After a successful deploy, the Operations guide takes over.

| Deployment concern | Operations handoff |
|---|---|
| Health check endpoints defined here | Operations sets up uptime monitoring against these endpoints |
| Structured log events wired during development | Operations configures log aggregation, alerting on error rates |
| Rollback plan documented per deploy | Operations escalation runbook references rollback procedure |
| Feature flags deployed but disabled | Operations (or on-call) enables flags for canary rollout |
| Canary deploy started | Operations monitors error rate and latency during ramp-up |
| Deploy completes successfully | Operations verifies dashboards show no regression |
| Incident detected post-deploy | Operations triggers rollback runbook → back to Rollback Strategy section |

The Operations guide covers: monitoring dashboards, alerting thresholds, on-call rotation, incident response, post-mortems.
