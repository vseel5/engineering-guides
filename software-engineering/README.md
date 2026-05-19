# Software Engineering Guides

A collection of 10 language/framework-agnostic engineering guides covering the full lifecycle of building and operating software systems — from idea to production and beyond. Every guide uses a consistent running example (an **Event booking system**) so concepts stay grounded in a concrete, familiar domain rather than abstract theory.

These guides were written to be **used on the job**, not read once and shelved. Each one is structured around gate questions, decision tables, anti-pattern lists, and operational checklists you can apply directly to your work.

---

## The Guides

### [001 — Ideation & Planning](001_ideation-planning-guide.md)

Seven sequential gates that transform a vague problem into a signed-off, sprint-ready backlog. Covers problem discovery, feasibility study, MoSCoW scope management, requirements engineering, technical design brief, estimation and sequencing, and stakeholder approval. Every gate has a pass/fail question — don't move forward until the current gate passes.

**Use when:** starting a new project or feature, or when you suspect the team is building the wrong thing.

---

### [002 — Tech Stack Selection](002_tech-stack-guide.md)

A structured scoring framework for choosing technologies without defaulting to hype. Covers a five-criterion scoring table (team expertise, ecosystem maturity, hiring pool, operational complexity, licensing), backend/frontend/DB/infrastructure decision tables, a buy-vs-build catalogue, and an Architecture Decision Record (ADR) template for documenting choices.

**Use when:** choosing a language, framework, database, or third-party service — or when you need to justify a technical choice to stakeholders.

---

### [003 — Design](003_design-guide.md)

Eight sections that take requirements and produce a design package ready for implementation. Notably, **Security Design comes before API Design** — the auth strategy shapes the entire endpoint surface, so you cannot finalise API contracts without a security model. Covers sequence diagrams, OpenAPI spec structure, ERD, state machines, and an error catalogue.

**Use when:** beginning implementation of any non-trivial feature, or reviewing whether an existing system's design matches its intent.

---

### [004 — Scaffolding](004_scaffolding-guide.md)

The architectural rulebook for structuring code. Defines the strict layered pattern (routes → controller → service → repository → domain logic), the layer contract table (what each layer can and cannot call), folder structure, middleware ordering, error class hierarchy, configuration management, and observability patterns. Includes dedicated sections on **Idempotency Keys** and **Graceful Shutdown**.

**Use when:** starting implementation of a new service, onboarding to an existing codebase, or evaluating whether a codebase's structure is sound.

---

### [005 — Git Workflow](005_git-workflow-guide.md)

Trunk-based development over GitFlow. Covers branch naming conventions, maximum branch lifetime (2 days for features), Conventional Commits format, PR size limits (< 400 lines), review SLAs by priority, merge strategies (squash for features, fast-forward for hotfixes), and semantic versioning. Explains why trunk-based development reduces merge conflict overhead and keeps `main` always deployable.

**Use when:** setting up a new team's workflow, onboarding contributors, or debugging why integration pain is high on an existing team.

---

### [006 — Testing](006_testing-guide.md)

The test pyramid in practice: 70% unit / 20% integration / 10% E2E. Covers what belongs in each layer, how to write tests that survive refactors (test behaviour not implementation), integration test database strategy (real DB, never mock — reseed before each run), test data factories, and CI gate placement. Includes coverage thresholds (95% for shared logic, 80% overall) and a test review checklist.

**Use when:** setting up testing on a new project, deciding whether to unit test or integration test a specific piece of logic, or diagnosing why a test suite is slow or brittle.

---

### [007 — Performance & Load Testing](007_performance-testing-guide.md)

Six test types — baseline, load, stress, spike, soak, capacity — with full k6 code examples. Covers realistic traffic distribution modelling (not uniform load), threshold definition, CI integration (automated baseline post-deploy, full load test pre-release), and a diagnosis playbook for when a load test fails. Never test against production; staging must mirror production scale.

**Use when:** before a significant release, before a known high-traffic event, after an architectural change, or after a performance regression reported in production.

---

### [008 — Code Quality](008_code-quality-guide.md)

The reader-first philosophy: code is read 10× more than written. Hard limits enforced by CI: ≤ 20 lines per function, cyclomatic complexity ≤ 5, nesting ≤ 3, params ≤ 3, file ≤ 300 lines. Covers naming conventions, error handling patterns, dependency injection, configuration management, and a full code review checklist with example reviewer comments. Explicitly excludes config, folder structure, and middleware (those live in the Scaffolding Guide).

**Use when:** conducting a code review, setting up linting/static analysis, or establishing team coding standards.

---

### [009 — Deployment & CI/CD](009_deployment-guide.md)

The pipeline from commit to production. Covers the expand/contract migration pattern for zero-downtime schema changes, pipeline stages (lint → test → build → package → deploy staging → deploy production), secrets management, liveness vs readiness probe distinction, image tagging strategy (git SHA only — never `:latest` in production), and a comparison of blue-green, rolling, and canary deployment strategies.

**Use when:** setting up a deployment pipeline, planning a release, or diagnosing why a deployment caused downtime.

---

### [010 — Operations & Maintenance](010_operations-maintenance-guide.md)

Running a system in production. Covers the four golden signals (latency, traffic, errors, saturation), alert catalogue with P1–P4 severity levels, a complete runbook template with a worked example, a blameless post-mortem template with a worked example, backup strategy (RPO/RTO), patching SLAs (Critical CVE < 24 hours), and a daily/weekly/monthly/quarterly operations checklist. Section 14 closes the loop: operational findings feed back into earlier guides.

**Use when:** an incident happens, setting up monitoring for a new system, preparing for a production launch, or running a post-mortem.

---

## Sequential vs. Non-Sequential Use

These guides have a natural forward flow for **new projects**:

```
001 Ideation → 002 Tech Stack → 003 Design → 004 Scaffolding
                                                     ↓
005 Git Workflow ──────────────────────────────────► all phases
                                                     ↓
006 Testing ←──── 007 Performance Testing ◄──── 008 Code Quality
       ↓
009 Deployment → 010 Operations ──────────────────► back to 001
```

But **most of the time you are not starting from scratch**. Use the jump table below to go directly to what you need.

---

## Jump Table — Entry Points by Situation

| Situation | Start here | Then go to |
|---|---|---|
| Starting a new project | [001 Ideation](001_ideation-planning-guide.md) | 002 → 003 → 004 in order |
| Joining an existing project | [004 Scaffolding](004_scaffolding-guide.md) §1–3 | [005 Git Workflow](005_git-workflow-guide.md) |
| Choosing a database or framework | [002 Tech Stack](002_tech-stack-guide.md) | [003 Design](003_design-guide.md) ERD section |
| Designing a new feature | [003 Design](003_design-guide.md) | [006 Testing](006_testing-guide.md) — write tests from contracts |
| Setting up a test suite from scratch | [006 Testing](006_testing-guide.md) §1–4 | [007 Performance](007_performance-testing-guide.md) baseline section |
| Preparing for a production release | [009 Deployment](009_deployment-guide.md) | [007 Performance](007_performance-testing-guide.md) pre-release checklist |
| System is slow in production | [007 Performance](007_performance-testing-guide.md) §10 diagnosis | [010 Operations](010_operations-maintenance-guide.md) §2 golden signals |
| Production incident in progress | [010 Operations](010_operations-maintenance-guide.md) §6 runbook | §8 post-mortem after it's resolved |
| Code review — reviewer | [008 Code Quality](008_code-quality-guide.md) §13 checklist | — |
| Code review — author preparing a PR | [008 Code Quality](008_code-quality-guide.md) §3–5 | [005 Git Workflow](005_git-workflow-guide.md) PR size section |
| Setting up CI/CD pipeline | [009 Deployment](009_deployment-guide.md) §3–6 | [008 Code Quality](008_code-quality-guide.md) — map quality rules to gates |
| High-traffic event coming up | [007 Performance](007_performance-testing-guide.md) §5 spike test | [010 Operations](010_operations-maintenance-guide.md) §4 alert catalogue |
| Tech debt sprint | [008 Code Quality](008_code-quality-guide.md) | [004 Scaffolding](004_scaffolding-guide.md) — re-evaluate layer boundaries |
| Justifying a technology choice | [002 Tech Stack](002_tech-stack-guide.md) ADR template | — |
| Scoping a feature to cut scope | [001 Ideation](001_ideation-planning-guide.md) §3 MoSCoW | [003 Design](003_design-guide.md) — redesign for reduced scope |
| Post-mortem after an incident | [010 Operations](010_operations-maintenance-guide.md) §8 | [001 Ideation](001_ideation-planning-guide.md) — if it reveals a requirements gap |
| Setting up a new team's process | [005 Git Workflow](005_git-workflow-guide.md) | [008 Code Quality](008_code-quality-guide.md) — agree on standards together |

---

## How the Guides Connect

Each guide produces outputs that feed directly into others. You do not need all of them to make one useful — but knowing the connections helps you know where to look when a question in one guide isn't answered there.

| Guide | Key output | Goes into |
|---|---|---|
| 001 Ideation | Signed-off requirements, MoSCoW scope, NFRs | 002 (tech constraints), 003 (design contracts) |
| 002 Tech Stack | ADRs, chosen stack, buy-vs-build decisions | 003 (ERD + API shaped by DB choice), 004 (folder structure per stack) |
| 003 Design | Sequence diagrams, OpenAPI spec, state machines, error catalogue | 004 (scaffolding follows design contracts), 006 (tests written from contracts) |
| 004 Scaffolding | Layer structure, middleware order, error classes | 006 (unit test targets), 008 (quality rules reference layer boundaries) |
| 005 Git Workflow | Branch conventions, PR process, merge strategy | 009 (CD pipeline triggers on merge to main) |
| 006 Testing | Test suite, coverage thresholds, CI placement | 009 (gates in the pipeline) |
| 007 Performance | Breaking point, scaling triggers, latency baselines | 010 (alert thresholds, scaling policies) |
| 008 Code Quality | Lint rules, complexity limits, review checklist | 009 (CI gates) |
| 009 Deployment | Pipeline configuration, migration strategy, deploy method | 010 (runbooks reference the deploy process) |
| 010 Operations | Incident learnings, SLA measurements, post-mortems | 001 (feed findings back into next planning cycle) |

---

## What These Guides Are Not

- **Not a waterfall process.** Use what applies. Skip what doesn't. Come back to skipped sections when they become relevant.
- **Not framework documentation.** There are no React hooks, no Prisma calls, no Express routes. Every pattern here works in any language and any stack.
- **Not a checklist to complete before writing code.** The checklists inside are operational — use them at the moment they apply, not all upfront.
- **Not prescriptive about team size.** A solo developer uses a subset. A ten-person team uses more. Scale the process to the team, not the other way around.
