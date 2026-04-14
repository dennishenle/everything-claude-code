---
description: Create a structured implementation plan with individual task files per task. Saves approved plan to .claude/plans/<feature-slug>/. Use /implement to execute tasks.
argument-hint: <feature-description>
---

# Plan Tasks Command

Creates an implementation plan and saves each task as a separate Markdown file so they can be executed incrementally (or all at once) with `/implement`.

Arguments: `$ARGUMENTS`

---

## Phase 1 — PLAN

You are an expert planning specialist. Analyze the feature description and produce a comprehensive, actionable implementation plan.

Feature to plan: `$ARGUMENTS`

### 1a. Requirements Analysis

- Understand the feature request completely
- Ask the user clarifying questions if anything is ambiguous — do not guess
- Identify success criteria
- List assumptions and constraints

### 1b. Architecture Review

Before planning, explore the codebase to ground the plan in reality:

- Analyze existing codebase structure (use Glob, Grep, Read)
- Identify affected components and files
- Review similar implementations and reusable patterns
- Note naming conventions, project structure, and established patterns

### 1c. Risk Assessment

Identify potential blockers, unknowns, and edge cases:

- Backwards-compatibility concerns (APIs, DB schemas, events)
- Performance bottlenecks
- Missing error handling or edge cases
- External dependencies or integration risks

### 1d. Task Breakdown

Break the feature into an ordered list of tasks. Each task must include:

- **Title** — short imperative phrase (e.g. "Add user model and migration")
- **Goal** — one or two sentences describing what "done" looks like
- **Context** — relevant existing files, patterns, and interfaces to follow. Be specific — list file paths and function names where known.
- **Acceptance criteria** — verifiable checklist items
- **Dependencies** — which prior task numbers must be complete first (empty list if none)
- **Estimated complexity** — Low / Medium / High

### Task Sizing and Ordering

- Size each task so it is independently implementable in a focused session. Avoid tasks that are too coarse (entire subsystems) or too fine (single-line changes).
- Prioritize by dependencies — a task's dependencies must have lower task numbers.
- Group related changes to minimize context switching.
- Enable incremental testing — each task should leave the codebase in a working state.

### Phasing Large Features

When the feature is large, break it into independently deliverable phases:

- **Phase 1**: Minimum viable — smallest slice that provides value
- **Phase 2**: Core experience — complete happy path
- **Phase 3**: Edge cases — error handling, edge cases, polish
- **Phase 4**: Optimization — performance, monitoring, analytics

Each phase should be mergeable independently. Avoid plans that require all phases to complete before anything works.

### Planning Best Practices

1. **Be Specific**: Use exact file paths, function names, variable names
2. **Consider Edge Cases**: Think about error scenarios, null values, empty states
3. **Minimize Changes**: Prefer extending existing code over rewriting
4. **Maintain Patterns**: Follow existing project conventions
5. **Enable Testing**: Structure changes to be easily testable
6. **Think Incrementally**: Each step should be verifiable
7. **Document Decisions**: Explain why, not just what

### Red Flags to Watch For

- Large functions (>50 lines)
- Deep nesting (>4 levels)
- Duplicated code
- Missing error handling
- Hardcoded values
- Missing tests
- Plans with no testing strategy
- Steps without clear file paths
- Phases that cannot be delivered independently

### Worked Example

Here is a complete plan showing the level of detail expected:

```markdown
# Plan: Stripe Subscription Billing

## Summary
Add subscription billing with free/pro/enterprise tiers. Users upgrade via
Stripe Checkout, and webhook events keep subscription status in sync.

## Risks
- Webhook events may arrive out of order — use event timestamps, idempotent updates
- User upgrades but webhook fails — poll Stripe as fallback, show "processing" state

## Tasks

### Task 01: Create subscription migration  [Low]
**Goal:** Store billing state server-side with a subscriptions table.
**Depends on:** none
**Context:** `supabase/migrations/` — follow existing migration naming. Table needs
user_id, stripe_customer_id, stripe_subscription_id, status, tier columns with RLS policies.
**Acceptance criteria:**
- [ ] Migration creates subscriptions table with all required columns
- [ ] RLS policies restrict access to own subscriptions

### Task 02: Create Stripe webhook handler  [High]
**Goal:** Keep subscription status in sync with Stripe via webhook events.
**Depends on:** 01
**Context:** `src/app/api/` — follow existing route conventions. Must handle
checkout.session.completed, customer.subscription.updated, customer.subscription.deleted.
**Acceptance criteria:**
- [ ] Webhook verifies Stripe signature
- [ ] Handles all three event types
- [ ] Updates subscriptions table idempotently

### Task 03: Create checkout API route  [Medium]
**Goal:** Server-side Stripe Checkout session creation to prevent price tampering.
**Depends on:** 01
**Context:** `src/app/api/` — needs price_id and success/cancel URLs.
**Acceptance criteria:**
- [ ] Validates user is authenticated
- [ ] Creates Checkout session with correct price
- [ ] Returns session URL

### Task 04: Build pricing page  [Low]
**Goal:** User-facing upgrade flow displaying three tiers.
**Depends on:** 03
**Context:** `src/components/` — follow existing component patterns.
**Acceptance criteria:**
- [ ] Displays Free, Pro, Enterprise tiers with feature comparison
- [ ] Upgrade buttons trigger checkout flow

### Task 05: Add tier-based middleware  [Medium]
**Goal:** Enforce tier limits server-side on protected routes.
**Depends on:** 01, 02
**Context:** `src/middleware.ts` — extend existing middleware.
**Acceptance criteria:**
- [ ] Checks subscription tier on protected routes
- [ ] Redirects free users from pro-only features
- [ ] Handles edge cases (expired, past_due)
```

---

## Phase 2 — PRESENT AND AWAIT CONFIRMATION

Present the complete plan to the user clearly, formatted as:

```
# Plan: <Feature Name>

## Summary
<1–3 sentences>

## Risks
- <risk 1>
- <risk 2>

## Tasks

### Task 01: <Title>  [Low | Medium | High]
**Goal:** <goal>
**Depends on:** none | tasks 01, 02
**Context:** <relevant files and patterns>
**Acceptance criteria:**
- [ ] <criterion>

### Task 02: ...
```

Then display:

```
WAITING FOR CONFIRMATION
Proceed with saving this plan? (yes / no / modify: <your changes>)
```

**Do not write any files until the user responds.**

If the user requests modifications (`modify: ...`), regenerate the affected parts of the plan and re-present. Repeat until the user confirms with "yes", "proceed", or equivalent affirmative.

---

## Phase 3 — SAVE PLAN FILES

On confirmation, derive `<feature-slug>` from the feature description:
- lowercase, kebab-case
- strip stop words and punctuation
- max 40 characters
- Example: "Add real-time notifications for market resolution" → `realtime-market-notifications`

Create the plan directory: `.claude/plans/<feature-slug>/`

### File: `_index.md`

```markdown
---
feature: <Full Feature Name>
slug: <feature-slug>
created: <YYYY-MM-DD>
status: pending
---

# <Full Feature Name>

## Summary
<1–3 sentence description>

## Risks
<bullet list>

## Tasks

| # | Title | Status | Complexity | Depends On |
|---|-------|--------|------------|------------|
| 01 | <title> | pending | Low | — |
| 02 | <title> | pending | Medium | 01 |
```

### File: `task-NN.md` (one file per task, zero-padded to two digits)

```markdown
---
task: "NN"
title: "<Task Title>"
status: pending
complexity: Low | Medium | High
depends_on: []
---

# Task NN: <Title>

## Goal
<What "done" looks like for this task.>

## Context
<Relevant files, interfaces, naming conventions, and patterns to follow.
Be specific — list file paths and function names where known.>

## Acceptance Criteria
- [ ] <criterion 1>
- [ ] <criterion 2>

## Notes
<Warnings, gotchas, design decisions, or open questions.
Leave blank if none.>
```

---

## Phase 4 — REPORT

After writing all files, report:

```
Plan saved: .claude/plans/<feature-slug>/

  _index.md        overview and task table
  task-01.md       <title>
  task-02.md       <title>
  ...

Run:
  /implement <feature-slug> 1      implement a single task
  /implement <feature-slug> all    implement all tasks sequentially
```
