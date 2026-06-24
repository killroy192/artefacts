---
name: generate-context-map
description: Turns a spec, context map and implementation plan into a structured verification plan. Use when the user asks to generate a verification plan, write a verification plan or create artefacts for AI-assisted implementation.
disable-model-invocation: true
---

# Generate Artefact

Produce **artefact only** — verification plan. Verification agent will rely on this plan during verification stage.

## Hard rules

- **Do not implement** — no application code, tests, config, or migrations.
- **Do not propose code changes** — no diffs, snippets, pseudocode patches, or “change line X to Y”.

## Inputs

| Input | Required | Notes |
|-------|----------|--------|
| Feature implementation plan | Yes | Under `artefacts/plans/` or pasted in chat |
| Context map | Yes | Under `artefacts/context-map/` or pasted in chat |
| Feature slug / name | If ambiguous | Derive from story title for artefact filenames |

## Outputs

| Artefact | Default path |
|----------|----------------|
| Verification Plan | `artefacts/verification/<feature-slug>.verification.md` |

---

## Verification Gates (Dimensions)

1. Spec compliance - Does the diff satisfy every acceptance criterion?
2. Scope control - Is anything included that was not requested?
3. Test quality - Do tests prove behavior, not just cover lines?
4. Risk - Security, performance, data or rollout concerns?
5. Maintainability - Would a new developer understand the change?
6. Evidence - Can a reviewer verify without re-reading all code?

---

## Workflow

1. Map each acceptance criterion to a verification method.
2. Identify existing tests and checks that should be reused.
3. Propose what to cover with tests if coverage is missing.
4. Identify typecheck, lint, build and smoke commands from repository files.
5. Flag acceptance criteria that are not directly verifiable yet.
6. Provide action list how each verification gate will be passed (validated).
7. At the end, do validation and provide evaluation score according to the dimensions (score 0-2).
8. Save results.
