# Spec Section Guide

Use with [SKILL.md](SKILL.md). Section titles must match exactly.

## Business Goal

One short paragraph: why this work matters to users or the product. Outcome-focused, not technical.

## User / System Problem

Pain today: what users cannot do, or what the system lacks. Tie to observable behavior.

## Current Behavior

Factual inventory from the codebase. Split by layer if helpful (frontend, API, data). No “should” language.

## Expected Behavior

User-visible and system-visible outcomes after the story is done. Bullet list is fine. Avoid naming specific functions or line numbers.

## Technical Scope

Bounded work: layers touched, kinds of changes (new UI control, new endpoint parameter), files likely involved (table OK). Explicitly state what is in scope.

Do **not** include code, diffs, or ordered implementation steps.

## Non-Goals

Explicit exclusions to prevent scope creep (other features, refactors, URL state, backend if frontend-only, etc.).

## Constraints

Stack, patterns, data limits, compatibility requirements, tests that must keep passing. Facts from the repo, not preferences.

## Acceptance Criteria

Numbered, testable statements (AC-1, AC-2, …). Prefer a table with a “Verifiable by” column (test, manual, API check).

Each criterion should be independently checkable.

## Verification Expectations

### Automated tests

What new or updated tests should prove (scenario → assertion). Name test files if known.

### Regression

Existing tests or suites that must still pass.

### Manual verification

Short checklist for things hard to automate (animations, full stack, etc.).

Repo commands only (e.g. `npm run test:run`) — not feature implementation commands.

## Open Questions

Unresolved decisions that block a safe plan. Empty or “None” if exploration resolved everything.

For each item: question, options if known, recommended default only if evidence supports it (label as recommendation, not decision).

## Assumptions

Beliefs taken as true for this spec (exact match vs range, sync vs async, no new dependencies). Distinguish from constraints (hard limits).

## Initial Context Hints

Table for the implementation-plan agent:

| Area | File | Key detail |
|------|------|------------|
| … | `path` | Fact discovered during exploration |

Pointers only — no edit instructions or code.
