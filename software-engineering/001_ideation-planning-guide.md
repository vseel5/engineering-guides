# Software Ideation & Planning Guide

Language/framework-agnostic. Based on industry practices (Shape Up, RICE, MoSCoW, RFC process). Copy to any project.

---

## How to use this guide

Work through all 7 sections in order. Each section has a **gate question** at the bottom. Do not proceed to the next section until that question is answered YES. Skipping gates is the #1 cause of rework.

Running example throughout: **Event booking system** (users browse events, reserve seats, pay, receive confirmation).

---

## 1. Problem Discovery

**Answers:** What problem exists, who has it, and why does it matter now?

This is the most important section. A well-defined problem is 80% of the solution. Most failed projects started with a solution, not a problem.

### Problem Statement Template

```
[Who] struggle to [do what] because [root cause],
which results in [measurable negative outcome].

Today they solve this by [current workaround],
which is painful because [specific friction].
```

### Example — Event booking system

```
Event organisers struggle to manage seat reservations manually because
they rely on spreadsheets and WhatsApp messages, which results in
double-bookings and 30% no-show rates.

Today they track attendance via phone calls and printed lists,
which is painful because it takes 3+ hours per event and still produces errors.
```

### Discovery Questions

Answer all before writing a line of requirements:

| Question | Why it matters |
|---|---|
| Who specifically has this problem? (job title, context) | Vague audience → vague product |
| How often does the problem occur? | Frequency determines priority |
| What is the cost of NOT solving it? (time, money, risk) | Justifies investment |
| How do they solve it today? | The workaround is your real competitor |
| Have you talked to 3+ real users? | Assumed problems are dangerous |
| Is this problem getting worse, stable, or going away? | Timing matters |

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Start with "we need an app that does X" | Solution-first skips validating the problem exists |
| Define the problem in one sentence with no evidence | Survives no scrutiny, misleads the team |
| Interview only internal stakeholders | They describe their assumptions, not user pain |
| Document the problem but never revisit it | Problem evolves; solution drifts from reality |

### Gate Question

> **Can you describe the problem in one paragraph, name a real person who has it, and explain what it costs them today?**

---

## 2. Feasibility Study

**Answers:** Can we actually build this — technically, legally, and financially?

Feasibility is not "is this possible in theory." It is "can THIS team build this in THIS timeline with THESE constraints."

### Feasibility Dimensions

| Dimension | Questions to answer |
|---|---|
| **Technical** | Does the technology exist? Does the team know it? What are the hardest unknowns? |
| **Legal / Compliance** | Does the product handle personal data (GDPR)? Payments (PCI-DSS)? Health data (HIPAA)? |
| **Operational** | Who runs it after launch? What infrastructure does it need? |
| **Financial** | What does it cost to build? What does it cost to run per month? What is the break-even? |
| **Time** | What is the hard deadline (if any)? Is it external (contract) or internal (preference)? |

### Risk Register Template

```
| Risk | Likelihood (H/M/L) | Impact (H/M/L) | Mitigation |
|------|-------------------|----------------|------------|
| ...  | ...               | ...            | ...        |
```

### Example — Event booking system

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Payment gateway integration complexity | M | H | Prototype Stripe integration in week 1 |
| No-show rate data needed for SLA | M | M | Collect manually for 2 events before automating |
| GDPR — storing attendee PII | H | H | Data minimisation from day 1; DPA with payment processor |
| Team has no mobile experience | L | M | Web-only MVP; mobile is v2 |

### Technical Spike

For every high-impact unknown: timebox a spike (1–3 days). Build only enough to answer the question. Throw away the code. Document the answer.

```
Spike: Can we integrate Stripe webhooks with our chosen framework?
Timebox: 2 days
Question to answer: Can we reliably confirm payment before issuing a ticket?
Output: yes/no + code sample + gotchas found
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Skip feasibility entirely ("we'll figure it out") | Unknown unknowns kill timelines |
| Mark all risks as Low | Creates false confidence; blindsides the team |
| Treat a spike as production code | Wastes time; the goal is the answer, not the code |
| Ignore legal/compliance until launch | Compliance retrofits are extremely expensive |

### Gate Question

> **Have you identified the top 3 risks, assigned a mitigation for each, and confirmed no hard blockers exist?**

---

## 3. Scope Management

**Answers:** What is in this version, what is explicitly out, and what is deferred?

Scope creep is not a planning failure — it is a documentation failure. If what's out is not written down, everything is in.

### MoSCoW Table

| Priority | Meaning | Rule |
|---|---|---|
| **Must have** | Product fails without this | MVP cannot ship without it |
| **Should have** | High value, not critical for launch | Target v1.1 |
| **Could have** | Nice to have | Target v2 |
| **Won't have (this time)** | Explicitly deferred | Write it down so it stops being debated |

### Example — Event booking system

| Priority | Feature |
|---|---|
| Must | Browse events, reserve seats, receive confirmation email |
| Must | Organiser dashboard: view reservations, export list |
| Should | Waitlist when event is full |
| Should | QR code ticket + scanner at door |
| Could | Recurring events |
| Could | Social sharing |
| Won't (v1) | Mobile app, refunds, multi-currency, loyalty points |

### Out-of-Scope Statement

Write this explicitly and get sign-off:

```
OUT OF SCOPE for v1:
- Mobile application (iOS/Android)
- Refund processing (manual refunds only via support)
- Multi-currency support (GBP only)
- Third-party calendar sync (Google Calendar, Outlook)
```

### Scope Creep Defence

When a new request arrives mid-build:

```
1. Write it down — never lose the idea
2. Classify it (MoSCoW)
3. Estimate the cost (time + scope impact)
4. Decide: defer, swap, or expand scope with stakeholder sign-off
5. Never silently absorb it
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| No explicit "won't have" list | Every omission becomes a future negotiation |
| All features are Must | Removes triage when time runs short |
| Scope agreed verbally, not written | "That's not what I meant" arguments at launch |
| Deferring scope decisions ("we'll decide later") | Deferred decisions block implementation |

### Gate Question

> **Is there a written, agreed Must/Should/Could/Won't list with explicit out-of-scope items signed off by stakeholders?**

---

## 4. Requirements Engineering

**Answers:** What exactly does the system do, from whose perspective, and how do we know when it's done?

Requirements without acceptance criteria are wishes, not requirements.

### User Story Format

```
As a [role]
I want to [action]
So that [outcome / value]

Acceptance criteria:
- Given [context], when [action], then [result]
- Given [context], when [action], then [result]
```

### Example — Event booking system

```
As an attendee
I want to reserve a seat for an event
So that I am guaranteed a place and receive confirmation

Acceptance criteria:
- Given the event has available seats, when I submit my details,
  then my seat count decreases by 1 and I receive a confirmation email within 60 seconds
- Given the event is full, when I attempt to reserve,
  then I am offered the waitlist option and no payment is taken
- Given my payment fails, when the card is declined,
  then no seat is reserved and I see an actionable error message
```

### Non-Functional Requirements (NFRs)

Always define these — they constrain architecture. Vague NFRs become arguments at launch.

| Category | Example requirement |
|---|---|
| Performance | Search results load in < 1s at p95 under 500 concurrent users |
| Availability | 99.9% uptime (allows ~8.7h downtime/year) |
| Security | All PII encrypted at rest; no plaintext passwords stored |
| Scalability | Support 10k events and 1M attendees without schema changes |
| Accessibility | WCAG 2.1 AA compliance for all user-facing pages |
| Data retention | Booking records retained 7 years (legal requirement) |

### Requirement Quality Checklist

A good requirement is:
- [ ] **Specific** — unambiguous; one interpretation only
- [ ] **Measurable** — has a number or testable condition
- [ ] **Achievable** — feasible within constraints
- [ ] **Traceable** — linked to a user story or business rule
- [ ] **Testable** — acceptance criteria can be written for it

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| "The system should be fast" | Not testable; what is fast? |
| Requirements written by one person without user input | Describes assumptions, not needs |
| No acceptance criteria | Team ships a feature that passes technically but fails the user |
| NFRs treated as optional | Architecture built for wrong load; security gaps discovered late |
| Requirements as a static document | Needs evolve; stale requirements mislead |

### Gate Question

> **Does every Must-have feature have at least one user story with written acceptance criteria and measurable NFRs defined?**

---

## 5. Technical Design Brief

**Answers:** What is the system structure, how do components interact, and what are the key technical decisions?

This is not a full technical spec. It is the minimum shared understanding the team needs to build consistently.

### Architecture Sketch

Draw (even rough ASCII) the major components and how data flows between them. Everyone on the team must be able to read it.

```
Example — Event booking system

[Browser / Mobile Web]
       ↓ HTTPS
[API Gateway / Load Balancer]
       ↓
[Application Server — REST API]
   ↓            ↓            ↓
[Auth Service] [Booking Svc] [Notification Svc]
       ↓            ↓            ↓
  [Postgres]    [Postgres]    [Email Provider]
                    ↓
              [Payment Gateway]
                (Stripe)
```

### Data Model Sketch

Key entities and their relationships. Not a full ERD — just enough to reveal conflicts early.

```
Event { id, title, organiser_id, capacity, starts_at, ends_at }
Booking { id, event_id, attendee_id, status, payment_ref, created_at }
Attendee { id, email, name }
Waitlist { id, event_id, attendee_id, position }
```

### Architecture Decision Records (ADRs)

For every non-obvious technical choice, document:

```
# ADR-001: Use Stripe for payments

Status: Accepted
Date: 2025-01-15

Context:
  Need to process card payments with minimal PCI scope.

Decision:
  Use Stripe Checkout (hosted page). We never handle raw card data.

Consequences:
  + PCI scope reduced to SAQ A
  + Stripe webhook needed for async payment confirmation
  - Less UI control over checkout experience
```

### Technical Design Checklist

- [ ] Architecture diagram shared and reviewed by full team
- [ ] Database schema (key entities + relationships) sketched
- [ ] Third-party services identified + evaluated (cost, reliability, lock-in)
- [ ] ADR written for every significant non-obvious choice
- [ ] Deployment environment defined (cloud provider, containers, managed vs self-hosted DB)
- [ ] Auth strategy defined (JWT, session, OAuth — not "TBD")
- [ ] API style defined (REST, GraphQL, gRPC)
- [ ] Monitoring strategy defined (logs, metrics, alerting)

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| No diagram at all | Each dev builds a different mental model; integration breaks |
| Over-engineering the design before writing code | Premature abstraction; design without feedback from reality |
| Leaving auth as "TBD" | Auth affects every layer; late decisions cause rewrites |
| No ADRs — decisions made in Slack | Impossible to onboard new devs; decisions get re-litigated |
| Designing for scale you won't have | Complexity with no benefit; optimise when measured |

### Gate Question

> **Can every team member describe the system's components, how they interact, and where their work fits — without asking anyone?**

---

## 6. Estimation & Sequencing

**Answers:** How long will this take, in what order should it be built, and what does the critical path look like?

Estimation is not prediction. It is a communication tool. The goal is to surface risk and align expectations, not to be right.

### Estimation Format

Never give a single number. Give a range with a confidence level.

```
Task: Payment integration
Best case:  3 days  (everything works, Stripe docs are clear)
Likely:     5 days  (one integration issue to debug)
Worst case: 9 days  (webhook reliability issue + retry logic needed)
Confidence: Medium
```

### Sequencing Principles

Build in this order:
1. **Skeleton first** — get data flowing end-to-end (even with fake data) before polishing any single layer
2. **Riskiest thing first** — tackle the biggest unknown early when there is still time to adapt
3. **Blocks first** — anything other tasks depend on ships before those tasks start
4. **Polish last** — UI fit-and-finish, edge cases, and optimisations come after the happy path works

### Dependency Map

```
Example — Event booking system

[1] DB schema + migrations          (blocks everything)
    ↓
[2] Auth (login/register)           (blocks all protected routes)
    ↓
[3] Event listing API               (blocks booking)
[3] Attendee profile API            (blocks booking)
    ↓
[4] Booking API (without payment)   (blocks payment integration)
    ↓
[5] Stripe integration              (blocks confirmation flow)
    ↓
[6] Email confirmation              (can overlap with [5])
[6] Organiser dashboard             (can overlap with [5])
    ↓
[7] QR code tickets + scanner       (should-have, post-MVP)
```

### RICE Prioritisation (for competing features)

```
Score = (Reach × Impact × Confidence) / Effort

Reach:      how many users affected (per period)
Impact:     3 = massive, 2 = high, 1 = medium, 0.5 = low
Confidence: 1.0 = high, 0.8 = medium, 0.5 = low
Effort:     person-weeks to ship
```

| Feature | Reach | Impact | Confidence | Effort | Score |
|---|---|---|---|---|---|
| Waitlist | 500 | 2 | 0.8 | 1 | 800 |
| QR scanner | 200 | 2 | 1.0 | 3 | 133 |
| Social sharing | 1000 | 0.5 | 0.5 | 1 | 250 |

Higher score = higher priority.

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Single-point estimates ("3 days") | No range = no risk signal |
| Estimating without breaking tasks down | Large tasks hide unknowns; estimates are fiction |
| Sequencing by preference, not dependency | Blockers discovered late; parallel work wasted |
| Treating estimates as commitments | Devs stop surfacing problems to protect the date |
| No buffer for integration + review | Everything estimated as greenfield; integration always takes longer |

### Gate Question

> **Does every Must-have story have an estimate range, a known owner, and a dependency order that reveals the critical path?**

---

## 7. Approval & Handoff

**Answers:** Who has signed off, what does the team need to start, and how will progress be tracked?

A handoff is not "sending the document." It is a synchronous conversation where blockers surface and questions get answered before the first line of code.

### Sign-Off Matrix

| Stakeholder | Role | What they approve | Sign-off date |
|---|---|---|---|
| Product Owner | Scope + priorities | MoSCoW list, user stories | |
| Tech Lead | Architecture | Design brief, ADRs, stack choices | |
| Legal / Compliance | Risk | Data handling, PII, payment compliance | |
| Operations | Infra | Deployment target, monitoring, runbooks | |

### Definition of Ready

A story is ready to be picked up when:
- [ ] User story written with acceptance criteria
- [ ] Designs/wireframes attached (if UI work)
- [ ] Dependencies identified and unblocked
- [ ] Estimate agreed
- [ ] Edge cases documented (what happens when payment fails, event is full, etc.)
- [ ] Test data / seed strategy known

### Definition of Done

A story is done when:
- [ ] Code reviewed and merged to main branch
- [ ] Unit tests written and passing
- [ ] Integration / E2E test covering the acceptance criteria
- [ ] No regressions in related features
- [ ] Deployed to staging and smoke-tested
- [ ] Relevant docs updated (API spec, README, runbook)

### Handoff Meeting Agenda (30–60 min)

```
1. Problem recap (5 min)          — remind everyone why this exists
2. Scope walkthrough (10 min)     — Must/Should/Won't; confirm shared understanding
3. Architecture walk (10 min)     — diagram walkthrough; questions on design decisions
4. Story walkthrough (15 min)     — walk through sprint 1 stories; surface blockers
5. Open questions (10 min)        — anything unresolved that could stop work
6. First milestone (5 min)        — agree on what "done" looks like in 2 weeks
```

### Communication Cadence

Define upfront — not when something goes wrong:

| Touchpoint | Frequency | Format | Owner |
|---|---|---|---|
| Progress update | Weekly | Written async (Slack / email) | Tech lead |
| Blocker escalation | As needed | Immediate (Slack / call) | Anyone |
| Scope change request | As needed | Written + stakeholder sign-off | Product owner |
| Demo / review | Per sprint | Live meeting | Full team |
| Retrospective | Per sprint | Meeting with action items | Full team |

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Handoff = sending a document | Unread documents create misaligned teams |
| Sign-off by a single person | Decisions later challenged by those not included |
| No Definition of Done | "Done" means different things to different people; scope creep disguised as completion |
| No communication cadence defined | Problems surface too late to course-correct |
| Starting before open questions are resolved | Builds on unstable ground; expensive to revert |

### Gate Question

> **Have all stakeholders signed off, every team member attended the handoff, Definition of Done is written, and sprint 1 stories are Ready?**

---

## Quick Reference — Gates Summary

| # | Section | Gate Question |
|---|---|---|
| 1 | Problem Discovery | Can you name the problem, who has it, and what it costs them? |
| 2 | Feasibility Study | Top 3 risks identified with mitigations; no hard blockers? |
| 3 | Scope Management | Written MoSCoW list with explicit out-of-scope, signed off? |
| 4 | Requirements Engineering | Every Must-have has user stories, acceptance criteria, and NFRs? |
| 5 | Technical Design Brief | Every team member can describe system components and where their work fits? |
| 6 | Estimation & Sequencing | Every Must-have story has an estimate range, owner, and dependency order? |
| 7 | Approval & Handoff | All stakeholders signed off, DoD written, sprint 1 stories Ready? |

---

## How This Feeds Into Tech Stack Selection

The outputs of this guide are the primary inputs to the **Tech Stack Selection Guide (002)**. Every planning gate produces a constraint that narrows technology options:

| Planning output | Tech stack implication |
|---|---|
| Problem domain & scale (§1) | Determines whether you need relational, document, or time-series data; rules out underpowered options |
| Feasibility constraints — team skills, budget, timeline (§2) | Filters out technologies the team cannot operate; budget rules out managed services with high per-seat cost |
| Must-have vs. Must-not scope (§3) | Identifies domains where a buy (SaaS/library) beats a build; removes gold-plated options |
| NFRs — latency, throughput, uptime (§4) | Sets the performance bar; a p99 < 200 ms requirement under 10k rps immediately rules out certain runtimes |
| Technical Design Brief — component boundaries (§5) | Shapes the service topology; a single-team, single-repo project rarely needs a service mesh |
| Estimation — time budget per component (§6) | A 2-week spike budget means boring, well-documented technology only; no room for immature ecosystems |
| Stakeholder sign-off (§7) | Locks in constraints so tech stack scoring uses stable requirements, not moving targets |

Do not open the Tech Stack Guide until all seven gates are answered YES. A tech stack chosen before the problem is fully understood will be optimised for the wrong thing.
