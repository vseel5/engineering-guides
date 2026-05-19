# Engineering Guides

A growing library of practical, language/framework-agnostic guides for engineering disciplines. Each section is self-contained — pick the discipline you need and go directly to its guide set.

Every guide is structured around gate questions, decision tables, checklists, and anti-patterns grounded in a concrete running example, so they stay operational rather than theoretical.

---

## Sections

### [Software Engineering](software-engineering/README.md)

10 guides covering the full traditional software lifecycle — from idea to production and back again.

| # | Guide | One line |
|---|---|---|
| 001 | [Ideation & Planning](software-engineering/001_ideation-planning-guide.md) | Seven gates from vague problem to signed-off backlog |
| 002 | [Tech Stack Selection](software-engineering/002_tech-stack-guide.md) | Score and document technology choices without defaulting to hype |
| 003 | [Design](software-engineering/003_design-guide.md) | Security model first, then API contracts, ERD, state machines, error catalogue |
| 004 | [Scaffolding](software-engineering/004_scaffolding-guide.md) | Layer boundaries, middleware order, idempotency keys, graceful shutdown |
| 005 | [Git Workflow](software-engineering/005_git-workflow-guide.md) | Trunk-based development, Conventional Commits, PR size limits, review SLAs |
| 006 | [Testing](software-engineering/006_testing-guide.md) | Test pyramid, real-DB integration tests, coverage thresholds, CI gates |
| 007 | [Performance & Load Testing](software-engineering/007_performance-testing-guide.md) | Baseline → load → stress → spike → soak, k6 examples, diagnosis playbook |
| 008 | [Code Quality](software-engineering/008_code-quality-guide.md) | Hard limits enforced by CI, naming, review checklist, tone guidelines |
| 009 | [Deployment & CI/CD](software-engineering/009_deployment-guide.md) | Expand/contract migrations, pipeline stages, zero-downtime strategies |
| 010 | [Operations & Maintenance](software-engineering/010_operations-maintenance-guide.md) | Golden signals, alert catalogue, runbooks, post-mortems, patching SLAs |

---

### AI Engineering *(coming soon)*

Covers the AI-specific lifecycle: data engineering, model selection (API vs fine-tune vs train), prompt engineering, evaluation (replaces traditional testing), model serving, output quality monitoring, and model lifecycle management.

---

### Project Management *(coming soon)*

Covers the non-technical lifecycle that runs alongside engineering: stakeholder management, roadmap planning, sprint ceremonies, risk registers, dependency tracking, and communication patterns.

---

## How to use this library

**Starting a new project?** Begin with the Software Engineering section in order (001 → 002 → 003...) until you reach implementation.

**Mid-project or joining an existing team?** Use the jump tables inside each section's README to go directly to what you need.

**Building an AI product?** Start with Software Engineering guides 001–005 (planning through Git workflow apply as-is), then switch to the AI Engineering section for model selection, evaluation, and serving. Return to 009–010 for deployment and operations with AI-specific additions noted there.

**Cross-discipline:** The sections are intentionally separate because the disciplines are distinct, but they share the same planning and design foundation. A project that involves both software and AI components uses guides from both sections — one does not replace the other.
