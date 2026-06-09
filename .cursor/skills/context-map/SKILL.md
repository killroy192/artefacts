---
name: generate-context-map
description: Turns a feature spec and implementation plan into a structured context map document (no code). Use when the user asks to generate a context map, write a context map or create artefacts for AI-assisted implementation.
disable-model-invocation: true
---

# Generate Spec

Produce **artefact only** — a structured context map. Another agent uses these as part of implementation context package.

## Hard rules

- **Do not implement** — no application code, tests, config, or migrations.
- **Do not propose code changes** — no diffs, snippets, pseudocode patches, or “change line X to Y”.

## Inputs

| Input | Required | Notes |
|-------|----------|--------|
| Feature spec | Yes | Under `artefacts/specs/` or pasted in chat |
| Feature implementation plan | Yes | Under `artefacts/plans/` or pasted in chat |
| Feature slug / name | If ambiguous | Derive from story title for artefact filenames |

## Outputs

| Artefact | Default path |
|----------|----------------|
| Context map | `artefacts/context-maps/<feature-slug>.contextmap.md` |

---

## Workflow

Using the spec and approved plan, inspect the relevant onboarding flow.

Map:

- files involved
- current loading behavior
- data flow
- likely change points
- files not to touch
- missing context
- risky assumptions
- smallest safe implementation boundary
