---
name: generate-spec
description: Turns a user story into a structured implementation spec (no code). Use when the user asks to generate a spec, write a story spec, create artefacts for AI-assisted implementation, or prepare work for an implementation-plan agent.
disable-model-invocation: true
---

# Generate Spec

Produce **artefact only** — a structured spec. Another agent uses these to write a safe implementation plan.

## Hard rules

- **Do not implement** — no application code, tests, config, or migrations.
- **Do not propose code changes** — no diffs, snippets, pseudocode patches, or “change line X to Y”.
- **Do not write an implementation plan** — no step-by-step coding tasks, file edit sequences, or PR breakdowns.
- **Clarify, don’t build** — inventory behavior, scope, risks, and verification so a later agent can plan safely.

If the user asks to implement in the same turn, complete spec artefact first and stop unless they explicitly cancel spec-only mode.

## Inputs

| Input | Required | Notes |
|-------|----------|--------|
| User story | Yes | Under `artefacts/stories/` or pasted in chat |
| Story slug / name | If ambiguous | Derive from story title for artefact filenames |
| Approach constraints | Optional | e.g. “frontend only” — record in spec, do not decide by coding |

## Outputs

| Artefact | Default path |
|----------|----------------|
| Spec | `artefacts/specs/<feature-slug>.spec.md` |

---

## Workflow

Copy and track progress:

```
Spec generation:
- [ ] Step 1 — Read story description and explore available resources
- [ ] Step 2 — Write structured spec from story
- [ ] Step 3 — Critique and Self-check (no implementation leakage)
```

### Step 2 Details — Structured spec from story

Transform the story into a spec using **exactly** these sections (headings and order):

1. **Business Goal**
2. **User / System Problem**
3. **Current Behavior**
4. **Expected Behavior**
5. **Technical Scope**
6. **Non-Goals**
7. **Constraints**
8. **Acceptance Criteria**
9. **Verification Expectations**
10. **Open Questions**
11. **Assumptions**
12. **Initial Context Hints**

Use the [spec section guide](spec-section-guide.md) for what belongs in each section. Keep the spec readable for product and engineering.

Resolve **Open Questions** when exploration answers them; leave only genuine unresolved items.

### Step 3 — Critique and Self-check

Review this spec as if you are the engineer who will later create a development plan. Do not implement anything. Find:

1. Blocking questions
2. Scope creep risks
3. Risky assumptions
4. Weak acceptance criteria
5. Missing verification evidence
6. Suggested improvements
