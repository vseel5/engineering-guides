# Git Workflow Guide

Sits between the **Scaffolding Guide** (structure defined) and the **Testing Guide** (tests written). Defines how code moves from a developer's machine through review and into the main branch. Every team member follows these rules from day one — not after the first broken main branch.

Language/framework-agnostic. Running example: **Event booking system**.

---

## 1. Core Principle

**The main branch is always deployable. Every commit on main could go to production right now.**

This is not aspirational — it is a hard rule. If main is broken, fixing it is the highest priority for the entire team. Nothing else ships until main is green.

---

## 2. Branching Strategy

**Answers:** How are branches named, how long do they live, and where do they merge?

### Trunk-based development (recommended)

Short-lived feature branches → merge frequently to main → main always green.

```
main
 ├── feat/add-waitlist      (< 2 days)
 ├── fix/payment-timeout    (< 1 day)
 └── chore/update-deps      (hours)
```

**Why not GitFlow?** GitFlow (develop, release, hotfix branches) adds coordination overhead that only pays off for teams shipping versioned software (mobile apps, desktop software, open source libraries). For web services that deploy continuously, trunk-based development is simpler and faster.

### Branch naming

```
{type}/{short-description-in-kebab-case}

Types:
  feat/     new feature or user-facing addition
  fix/      bug fix
  chore/    maintenance, dependency updates, config
  docs/     documentation only
  test/     adding or fixing tests
  refactor/ internal restructuring with no behaviour change
  perf/     performance improvement
  ci/       CI/CD pipeline changes
```

```
Good:
  feat/add-waitlist-feature
  fix/payment-webhook-retry
  chore/update-stripe-sdk
  docs/add-booking-api-spec

Bad:
  my-branch
  aseel-changes
  fix
  new-stuff-2
  FEAT_WAITLIST_V2_FINAL
```

### Branch lifetime

| Branch type | Max lifetime | If longer → |
|---|---|---|
| Feature (feat/) | 2 days | Break into smaller pieces or use feature flag |
| Bug fix (fix/) | 1 day | It is a simple fix or it is not a bug fix |
| Chore / docs | Hours | No reason to sit open |
| Hotfix | Hours | Emergency — must merge fast |

Long-lived branches diverge from main, accumulate conflicts, and become expensive to merge. A branch open for a week is a warning sign.

### Branch rules

- **Never commit directly to main** — all changes go through a PR
- **One concern per branch** — a branch that fixes a bug AND adds a feature is two branches
- **Delete after merge** — merged branches are dead; keep the remote clean

### Event booking system examples

```
feat/add-waitlist-position-tracking
feat/organiser-dashboard-csv-export
fix/booking-expiry-job-missing-refund
fix/stripe-webhook-duplicate-processing
chore/upgrade-prisma-v6
docs/update-booking-api-openapi-spec
test/add-waitlist-integration-tests
refactor/extract-payment-to-shared-client
ci/add-trivy-container-scan
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| Branches open for weeks | Massive merge conflicts; diverged from reality |
| `my-branch`, `test`, `fix2` | No one knows what it does without reading the diff |
| Committing directly to main | Bypasses review; breaks the "main is deployable" rule |
| One giant branch for an entire feature | Cannot review; cannot bisect; cannot roll back part of it |

---

## 3. Commit Conventions

**Answers:** What does a commit message look like, and what does one commit contain?

### Conventional Commits format

```
{type}({scope}): {subject}

{body — optional}

{footer — optional}
```

**Type:** feat, fix, chore, docs, test, refactor, perf, ci
**Scope:** the module or area affected (optional but helpful)
**Subject:** imperative mood, ≤ 72 characters, no period at end

```
Imperative mood = "add" not "added" or "adds"
Read as: "This commit will {subject}"

  feat(booking): add waitlist position tracking
  fix(webhook): prevent duplicate payment confirmation
  chore(deps): upgrade Stripe SDK to v14
  docs(api): add OpenAPI spec for booking endpoints
  test(booking): add integration tests for seat reservation race condition
  refactor(payment): extract Stripe client to shared/clients/stripe.ts
```

### Body — explain WHY, not WHAT

```
feat(booking): add waitlist position tracking

Organisers requested visibility into waitlist queue depth so they can
decide whether to increase event capacity. Position is recalculated
on every booking cancellation to maintain accurate ordering.

Closes #142
```

### Footer — breaking changes and issue references

```
feat(api)!: change booking response envelope to nested data field

BREAKING CHANGE: All booking endpoints now return { data: { ... } }
instead of the flat object. Frontend must update all booking API calls.

Closes #189
```

### Atomic commits — one logical change per commit

```
Good (atomic):
  fix(booking): correct seat count decrement on cancellation
  test(booking): add regression test for seat count on cancellation

Bad (non-atomic):
  fix stuff
  WIP
  asdf
  final fix
  fix for real this time
  various improvements
```

**Atomic rule:** if you revert this commit, the codebase still works. A commit that half-implements a feature fails this test.

### Cleaning up before a PR

Before opening a PR, squash or reword any debug/WIP commits on your branch:

```bash
# Interactive rebase to clean up last 4 commits
git rebase -i HEAD~4

# Squash WIP commits into the logical commit they belong to
# Reword commit messages that don't follow convention
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| `fix stuff`, `WIP`, `asdf` | Useless history; git bisect cannot find the breaking commit |
| Huge commits touching 20 files | Cannot review; cannot revert one change without reverting all |
| Committing generated files, lock file noise | Pollutes history; reviewer attention wasted |
| Past tense ("added feature") | Convention is imperative; tooling (changelog generators) expect it |
| No scope | Harder to filter history by area; changelog is a flat list |

---

## 4. Pull Request Guidelines

**Answers:** What does a PR look like, and what makes it ready for review?

### PR size limit

**< 400 lines changed** — excluding generated files, migrations, and lock files.

```
Count toward the limit:    source code, tests, config files
Do NOT count:              package-lock.json, schema migrations (but review them separately),
                           generated OpenAPI specs, vendored code
```

A 400-line PR takes 30–45 minutes to review properly. Larger PRs get rubber-stamped or take days to merge — both outcomes are bad.

If a feature requires > 400 lines: break it into sequential PRs (infrastructure first, then logic, then UI), or use a feature flag to merge incomplete work safely.

### PR title

Same format as a commit message: `{type}({scope}): {subject}`

```
feat(booking): add waitlist position tracking
fix(webhook): prevent duplicate payment confirmation on retry
chore(deps): upgrade Stripe SDK to v14.2.0
```

### PR description template

```markdown
## What changed
Brief description of the change. Link to the relevant issue.

Closes #142

## Why
Context the reviewer needs. What problem does this solve?
What alternatives were considered and rejected?

## How to test
1. Step-by-step instructions for verifying the change works
2. Include test credentials / seed data if needed
3. Edge cases to verify

## Checklist
- [ ] Tests added / updated
- [ ] Documentation updated (if API changed)
- [ ] No secrets hardcoded
- [ ] Migration is additive (or expand/contract documented)
- [ ] Feature flag added (if high-risk)
```

### Draft PRs

Open a draft PR early — even before the implementation is complete. Benefits:

```
→ CI runs on every push, catching issues early
→ Reviewer can see direction and flag concerns before work is done
→ Signals "in progress" without blocking others
→ Forces writing the PR description early (clarifies thinking)

Convert to ready-for-review when:
  → All checklist items complete
  → CI is green
  → You would be comfortable merging this if approved
```

### Anti-Patterns

| What people do wrong | Why it fails |
|---|---|
| PRs with 2000+ line diffs | Review is impossible; approvals are rubber stamps |
| PR opened with no description | Reviewer has no context; review takes 3× longer |
| "Fix PR comments" as separate commits | Pollutes history; use `git commit --fixup` or rebase |
| Mixing refactor + feature in one PR | Impossible to tell what caused a regression |
| Opening a PR then going on holiday | PR rots; conflicts accumulate; reviewer context lost |

---

## 5. Code Review Process

**Answers:** Who reviews, when, and how do reviewers and authors communicate?

### Review turnaround SLA

| Priority | SLA |
|---|---|
| Hotfix / P1 incident fix | < 2 hours, interrupt reviewer |
| Standard feature / fix | < 24 hours on business days |
| Chore / docs / dependency update | < 48 hours on business days |

A PR that sits unreviewed for 48+ hours is a process failure — not a reviewer failure. Escalate to the team lead.

### Reviewer assignment

| Change type | Required approvers |
|---|---|
| Standard feature / fix | 1 approver |
| Auth, security, payments | 2 approvers (one must be senior) |
| DB schema changes | 2 approvers (one must review migration) |
| CI/CD pipeline changes | 1 approver + team lead |
| Public API contract changes | 1 approver + frontend affected party |

### Comment types

| Type | Prefix | Meaning |
|---|---|---|
| Blocking | *(none or "Blocking:")* | Must be resolved before merge |
| Non-blocking suggestion | `nit:` | Author's call; can merge without acting |
| Question | `q:` | Author should answer; may or may not require change |
| Praise | `+1:` | Calling out something done well |

```
Examples:

"Blocking: This will crash if attendeeCount is 0 — add a guard."

"nit: Consider naming this `isPastEventStart` instead of `pastStart`
 for consistency with the boolean naming convention."

"q: Is there a reason we're using a transaction here? The two writes
 don't seem to depend on each other."

"+1: Great use of early returns here — much easier to follow than
 the nested version."
```

### Author responsibilities

- Respond to every comment — even if the response is "acknowledged" or "won't fix because X"
- Author resolves their own comment threads (not the reviewer)
- Do not merge until all blocking comments are resolved
- If you disagree with a blocking comment: discuss in the PR, escalate to team lead if unresolved

### Reviewer responsibilities

- Review the whole PR before leaving comments — do not drip-feed blocking issues one per day
- Distinguish blocking from non-blocking explicitly
- Approve only when you would be comfortable with this merging to main right now
- Do not approve and then add blocking comments in the same review

### Never merge your own PR

Exception: hotfix when no reviewers are available + you document why in the PR description.

```
# Hotfix self-merge template:
Self-merged due to: P1 incident — bookings failing for all users
Reviewer unavailable: 2am local time
Post-merge review: @teammate to review within 24 hours
Incident: #inc-2025-06-01-booking-down
```

---

## 6. Merge Strategy

**Answers:** How does a branch become part of main, and what does the resulting history look like?

### Strategies by scenario

| Scenario | Strategy | Why |
|---|---|---|
| Feature / fix branch → main | **Squash merge** | One clean commit per feature; readable history |
| Hotfix → main | **Fast-forward merge** | Small change; preserve exact commit |
| Release branch → main | **Merge commit** | Preserve full release history |

**Never rebase a public branch.** Rebasing rewrites commit hashes. If anyone has pulled your branch, their history diverges and they face a confusing force-push situation.

### Squash merge result

```
Before squash:
  feat/add-waitlist:
    abc1234  feat(booking): add waitlist table migration
    def5678  feat(booking): add waitlist service methods
    ghi9012  fix: resolve type error in waitlist position calc
    jkl3456  test: add waitlist integration tests
    mno7890  nit: rename variable per review feedback

After squash merge to main:
  main:
    pqr1234  feat(booking): add waitlist position tracking (#142)
```

Clean, readable, bisectable. One feature = one commit on main.

### Delete after merge

```bash
# Automatically on GitHub: enable "Automatically delete head branches" in repo settings
# Manually:
git branch -d feat/add-waitlist          # local
git push origin --delete feat/add-waitlist  # remote
```

Merged branches are dead. Keeping them creates noise and confusion about what is active work.

---

## 7. Keeping Branches Up to Date

**Answers:** How do you prevent a feature branch from drifting so far from main that merging becomes a nightmare?

### Rebase — not merge — onto main

```bash
# Wrong — creates a merge commit; pollutes PR diff
git checkout feat/add-waitlist
git merge main

# Right — replays your commits on top of latest main; clean history
git checkout feat/add-waitlist
git fetch origin
git rebase origin/main
```

### Rebase cadence

```
Rebase at minimum:
  → Before opening a PR
  → If main has received significant changes overnight
  → If you hit a conflict that gets worse the longer you wait

Rebase rule: never let a branch diverge more than 2 days from main
             without rebasing onto it
```

### Conflict resolution rule

When you hit a merge conflict on code you did not write, ask the author of the conflicting code before resolving. Assumptions about intent cause silent logic bugs.

```
Wrong: "I'll just take mine / take theirs"
Right: "@teammate, there's a conflict in bookingService.ts between
        your change to createBooking() and mine. Can you take a look
        before I resolve?"
```

---

## 8. Release Tagging & Versioning

**Answers:** How is a production release identified, and how does the version number communicate the change?

### Semantic versioning

```
MAJOR.MINOR.PATCH

MAJOR: Breaking change — existing API clients must update
MINOR: New feature, backward-compatible — clients can ignore it
PATCH: Bug fix, backward-compatible — transparent to clients

Examples:
  1.4.2 → 1.4.3  bug fix (patch)
  1.4.3 → 1.5.0  new waitlist feature added (minor)
  1.5.0 → 2.0.0  booking API response envelope changed (major, breaking)
```

### Tagging

```bash
# Annotated tag — always; includes message and tagger identity
git tag -a v1.5.0 -m "Add waitlist feature (#142, #145)"

# Push tag to remote
git push origin v1.5.0

# Never: lightweight tags on releases (no metadata)
git tag v1.5.0   ← wrong for releases
```

Tags are created on main only — never on feature branches.

### Changelog

Generated automatically from conventional commit messages:

```bash
npx conventional-changelog -p angular -i CHANGELOG.md -s
```

This only works if commit messages follow the convention in Section 3. Every `feat:` becomes a feature entry; every `fix:` becomes a bug fix entry; `BREAKING CHANGE:` gets a separate section.

---

## 9. Hotfix Process

**Answers:** When something breaks in production, how does the fix get there without disrupting in-progress feature work?

```
Production breaks → cut hotfix branch from main → fix + test → merge to main
                                                              → tag new patch version
                                                              → backport to any long-lived branches (if applicable)
```

### Rules

- Hotfix branches are cut from main (not from any feature branch)
- Naming: `hotfix/short-description`
- Every hotfix **must** include a regression test — the same bug must not recur
- Hotfix PR needs review (minimum 1 approver, 2 for security)
- Exception for self-merge: documented, follow-up review within 24 hours

```bash
# Cut from main
git checkout main
git pull
git checkout -b hotfix/prevent-double-booking-confirmation

# Fix + test
# ... make the change ...
# ... add the regression test ...

git commit -m "fix(booking): prevent duplicate confirmation on webhook retry"
git commit -m "test(booking): add regression test for duplicate webhook"

# Open PR → merge → tag
git tag -a v1.4.3 -m "Hotfix: prevent duplicate booking confirmation"
git push origin v1.4.3
```

---

## 10. Git Workflow Checklist (before opening a PR)

- [ ] Branch name follows `{type}/{description}` convention
- [ ] All commits follow Conventional Commits format
- [ ] No WIP, "fix", "asdf", or debug commits — squashed or reworded
- [ ] Branch rebased on latest main — no conflicts
- [ ] PR < 400 lines changed (excluding generated / lock files)
- [ ] PR title follows commit message format
- [ ] PR description complete: what, why, how to test, checklist
- [ ] CI checks passing (lint, type check, tests)
- [ ] Relevant reviewers assigned
- [ ] Linked to issue or ticket

---

## 11. Anti-Patterns

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| Long-lived feature branches (> 2 days) | Conflicts compound; diverges from reality | Feature flags + smaller slices |
| `fix stuff`, `WIP`, `asdf` commits | History is useless; bisect cannot find regressions | Atomic commits with conventional messages |
| Committing directly to main | No review; breaks deployability guarantee | Branch protection rules — require PR |
| Massive PRs (> 1000 lines) | Review is impossible; becomes a rubber stamp | Split by concern; feature flags for incomplete work |
| Force-pushing to shared branches | Rewrites history others have pulled; causes confusion | Only rebase local branches; never force-push main |
| Reviewing your own PR | Same blind spots that wrote the code review it | 1 required approver minimum; 2 for sensitive changes |
| Never deleting merged branches | Remote is a graveyard; nobody knows what is active | Auto-delete on merge (GitHub repo setting) |
| Merge conflicts resolved by "take mine" | Silent logic regression | Always ask the author of the conflicting code |

---

## 12. How This Feeds into Deployment

Git workflow decisions directly shape the CI/CD pipeline.

| Git workflow decision | Deployment impact |
|---|---|
| Main is always deployable | CD pipeline auto-deploys every merge to main — no manual "is this ready?" check |
| Conventional commits | CI auto-bumps semantic version; changelog auto-generated; no manual release notes |
| Squash merge | One clean commit per feature in main; `git bisect` finds regressions in minutes |
| Short-lived branches | Small diffs → small deploys → smaller blast radius per deploy |
| Hotfix process | Separate fast-path pipeline for emergency fixes; bypasses staging queue |
| Feature flags | Code merges to main (and deploys) before the feature is visible — decouple deploy from release |
| Branch protection (require PR + CI) | Pipeline cannot be bypassed; no unreviewed code reaches main |
