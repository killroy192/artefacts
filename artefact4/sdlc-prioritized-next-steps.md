# Prioritized Next Steps

**Goal:** Adopt AI-assisted SDLC practices in Cal.diy with highest impact-to-effort ratio, aligned to [sdlc-ai-use-cases.md](./sdlc-ai-use-cases.md) and the [value stream map](./sdlc-value-stream-map.md).

Scoring: **Priority** = f(impact ÷ effort), adjusted for risk. **Gate** = condition to promote to production team norm.

---

## Roadmap

| Priority | Step | Use cases | Owner | Effort | Gate to adopt |
|----------|------|-----------|-------|--------|---------------|
| **P0** | Mandate story → spec → plan before agent coding | UC-1 | Feature lead + agent user | 1 sprint policy | 1 feature shipped with full artefact3 package (instant-meeting ✅) |
| **P0** | Verifier subagent before every agent-authored PR | UC-2 | Agent user | Days | Verifier run logged in artefact5 for 2 consecutive PRs |
| **P1** | Enforce type-check-first in agent sessions | UC-3 | All agent users | Days | `ci-type-check-first` cited in 100% agent PR descriptions |
| **P1** | Verification plan + AC matrix for features ≥ 5 ACs | UC-4 | Feature lead | 1 sprint | artefact5-style table on next medium feature |
| **P2** | Context map for cross-package features (3+ packages) | UC-5 | Agent user | 1 sprint | instant-meeting map reused as template |
| **P3** | API breaking-change agent review on `packages/trpc` / API v2 diffs | UC-7 | API owner | 2 sprints | Pilot on 3 API PRs without false-positive blockers |
| **P4** | E2E failure triage assist (advisory only) | UC-6 | AQA + UI lead | 1 quarter | CI log export → cluster doc in < 4 h for one nightly run |

---

## P0 detail — Spec package + verifier

**Why first:** Addresses two largest rework loops — PR code review (70% negative feedback signal) and late MQA discovery (1–2 day Snapshot cycles).

**Actions**

1. Copy artefact3 workflow to `artefacts/` conventions:
   - `stories/` → `specs/` → `plans/` → `context-maps/` → `verification/`
2. Add PR template checkbox: “Verifier subagent output attached (artefact5 section).”
3. Block agent self-merge language in `AGENTS.md` PR section (human merge only).

**Verification**

- [ ] Next feature PR links spec, plan, verification plan
- [ ] artefact5 includes verifier summary
- [ ] Zero “surprise AC” items in manual QA

---

## P1 detail — Type-check-first + AC matrix

**Actions**

1. Reference `.cursor/commands/pre-push-checks.md` in agent kickoff for every task.
2. For each feature, produce verification plan before implementation (artefact3 `instant-meeting.verification.md` template).
3. Map each AC to `yarn vitest run <file>` or explicit manual step.

**Verification**

- [ ] CI type-check job green before test job on agent PRs
- [ ] ≥ 80% must-have ACs have automated row in artefact5 table

---

## P2 detail — Context maps

**Actions**

1. Require context map when plan manifest spans `apps/web` + `packages/trpc` + `packages/features`.
2. Verifier checks diff ⊆ manifest.

**Verification**

- [ ] No scope-creep files in last 2 cross-package PRs without plan amendment

---

## P3–P4 — Deferred with entry criteria

| Item | Start when | Stop if |
|------|------------|---------|
| UC-7 API advisory | First API v2 or tRPC router PR after P1 done | > 50% false positive breaking flags |
| UC-6 E2E triage | Nightly E2E owned by squad + Report Portal export available | No CI log access after 2 weeks |

---

## Anti-patterns (do not prioritize)

| Temptation | Why defer |
|------------|-----------|
| Agent auto-fixes flaky E2E | High risk (UC-6); masks infra debt |
| Skip human code review | Source org requires 2 engineers; AI augments, not replaces |
| Full SDLC JSON in Miro before P0 | Visualization without workflow change adds no lead-time cut |
| Generate specs for one-line fixes | UC-1 effort > benefit |

---

## 30 / 60 / 90 day outcomes

| Horizon | Outcome |
|---------|---------|
| **30 days** | P0 live: spec package + verifier on all agent feature PRs |
| **60 days** | P1 live: AC matrices standard; type-check-first in agent playbooks |
| **90 days** | P2 context maps for cross-package work; measure PR review round reduction |

**Success review:** Compare PR review comments tagged “missing test / scope creep” before and after P0 — target ≥ 30% reduction on agent-authored PRs.
