---
name: generate-plan
description: Turns spec into a structured implementation plan. Use when the user asks to create a plan, create artefacts for AI-assisted implementation or uses plan mode.
disable-model-invocation: true
---

# Generate Spec

Use Plan Mode to produce **artefact only** — a structured implementation plan. Another agent uses plan and spec (already created) to create a context map and verification plan.

## Hard rules

- **Do not implement** — no application code, tests, config, or migrations.
- **Do not propose code changes** — no diffs, snippets, or “change line X to Y”.

## Inputs

| Input | Required | Notes |
|-------|----------|--------|
| Feature spec | Yes | Under `artefacts/specs/` or pasted in chat |
| Feature slug / name | If ambiguous | Derive from spec title for artefact filenames |

## Outputs

| Artefact | Default path |
|----------|----------------|
| Implementation Plan | `artefacts/plans/<feature-slug>.plan.md` |

---

## Workflow

Before planning:

- inspect relevant files
- identify current behavior
- ask blocking questions if needed
The plan must include:
- confirmed facts
- assumptions
- files/modules involved
- Atomic, unambiguous implementation steps that clearly describe what needs to change. Avoid premature implementation details.
- risks
- explicit non-goals
- verification mapped to acceptance criteria
- smallest safe first step
