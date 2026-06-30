# SDLC AI-Assisted Use Cases

**Derived from:** [Generic SDLC Feedback Diagram.json](../artefact4-example/Generic%20SDLC%20Feedback%20Diagram.json)  
**Applied to:** Cal.diy monorepo (`caldiy-masterclass`) + `artefacts/` workflow (artefacts 1–3)

Ratings: **H** = High, **M** = Medium, **L** = Low.

---

## Use case index

| ID | Use case | SDLC hotspot | Impact | Risk | Effort |
|----|----------|--------------|--------|------|--------|
| UC-1 | Spec → plan → context package before coding | Local station, Snapshot MQA | H | M | L |
| UC-2 | Verifier subagent pre-PR gate | PR code review | H | L | L |
| UC-3 | Type-check-first agent command chain | PR build/lint, master CI | M | L | L |
| UC-4 | Verification plan with AC traceability | Quality Gate / nightly E2E | H | M | M |
| UC-5 | Context map for cross-package features | Local integration tests | H | M | M |
| UC-6 | E2E failure triage assist | Nightly UI E2E, Quality Gate sanity | M | H | H |
| UC-7 | API / tRPC breaking-change advisory | PR + master API dependency check | M | L | M |

---

## UC-1 — Spec → plan → context package before coding

**SDLC hotspot:** Local station; downstream Snapshot MQA (new feature ~2 days, bug-fix testing ~50% precision per source JSON).

**Problem:** Teams ship without shared acceptance criteria; manual QA discovers gaps late when env provisioning alone takes hours.

**Recommended AI workflow pattern:** **Chained skills with human approval gates**

1. Story → `generate-spec` skill → spec in `artefacts/specs/`
2. Spec → `plan` skill → implementation plan in `artefacts/plans/`
3. Spec + plan → `context-map` skill → hot/warm/cold context in `artefacts/context-maps/`
4. Human approves package → implementation agent runs with package attached

**Required context**

| Tier | Sources |
|------|---------|
| Hot | User story, feature spec, plan manifest |
| Warm | `AGENTS.md`, `ci-type-check-first.md`, similar merged PRs |
| Cold | Full app-store surface, unrelated packages |

**Verification approach**

- Spec AC IDs referenced in plan steps and verification plan (artefact3 pattern).
- Post-implementation: verifier subagent maps AC → test evidence.
- Manual: PO spot-check on Snapshot-equivalent staging (Cal.diy: local `yarn dx` + staging deploy).

**Ratings:** Impact **H** · Risk **M** (spec drift if not updated) · Effort **L** (skills exist in `artefacts/.cursor/skills/`)

---

## UC-2 — Verifier subagent pre-PR gate

**SDLC hotspot:** PR — Code Review (negative feedback ~70%, processing ~3600 s, two engineers mandatory in source org).

**Problem:** Human reviewers spend time on scope creep, missing tests, and spec gaps that machines can catch first.

**Recommended AI workflow pattern:** **Read-only skeptical subagent**

```
Parent agent (implementation) → run tests → launch verifier (readonly: true)
  → spec compliance → evidence → gaps → scope → security → merge readiness
  → parent fixes or documents waivers → artefact5 review notes
```

**Required context**

- Feature spec, plan, verification plan from `artefacts/`
- Git diff vs plan manifest
- Test command output (`TZ=UTC yarn vitest run …`)

**Verification approach**

- Verifier output must list each AC as pass / gap / manual-only.
- Block PR until gaps are fixed or explicitly waived in artefact5.
- Human review remains mandatory; agent reduces round-trips.

**Ratings:** Impact **H** · Risk **L** (readonly, no auto-merge) · Effort **L** ([verifier.md](https://github.com/killroy192/caldiy-masterclass/blob/main/.cursor/agents/verifier.md) committed in artefact2)

---

## UC-3 — Type-check-first agent command chain

**SDLC hotspot:** PR build/lint; master branch build (~50% precision, CI resource failures in source JSON).

**Problem:** Agents (and developers) run tests before TypeScript is clean, wasting CI minutes and producing misleading test failures.

**Recommended AI workflow pattern:** **Rule-enforced command ordering**

```
yarn type-check:ci --force  →  fix types  →  TZ=UTC yarn test <scope>  →  yarn biome check <files>
```

Enforced via `.cursor/rules/ci-type-check-first.md` + `AGENTS.md` + `.cursor/commands/pre-push-checks.md`.

**Required context**

- `ci-type-check-first.md`, changed package boundaries from plan
- Turbo filter hints when only one package changed

**Verification approach**

- Agent transcript must show type-check exit 0 before test commands.
- CI green on `type-check` job is ground truth.
- Optional: pre-push hook documents same order for humans.

**Ratings:** Impact **M** · Risk **L** · Effort **L**

---

## UC-4 — Verification plan with AC traceability

**SDLC hotspot:** Quality Gate E2E sanity (~98% negative feedback, 270 tests, results non-blocking); nightly UI E2E (~95% failure signal, 6 h runtime).

**Problem:** High E2E failure rates mix environment noise, data issues, and real regressions — teams lack a per-feature map of what is already proven at unit level.

**Recommended AI workflow pattern:** **Verification artefact + evidence matrix** (artefact3 / artefact5)

- `generate-spec` / manual spec defines AC-1…AC-N.
- Verification plan classifies each AC: automated (unit/component/integration) vs manual-only.
- artefact5 records command + result per automated AC.

**Required context**

- Feature spec AC section
- Plan file manifest (expected test files)
- Cal.diy test conventions (`testing-mocking.md`, `TZ=UTC`)

**Verification approach**

- Minimum: every **must-have** AC has owner (test name or manual step).
- Score compliance (e.g. 11/12 in instant-meeting artefact5) before merge decision.
- E2E failures triaged against matrix: if AC covered by unit tests, downgrade E2E blocker for that feature.

**Ratings:** Impact **H** · Risk **M** (false confidence if manual ACs skipped) · Effort **M**

---

## UC-5 — Context map for cross-package features

**SDLC hotspot:** Local station — integration tests mock API only (~80% negative feedback); monorepo cross-import mistakes.

**Problem:** Cal.diy features touch `packages/features`, `packages/trpc`, `apps/web`, and app-store apps — agents edit wrong layers without a navigation guide.

**Recommended AI workflow pattern:** **Hot / warm / cold context map skill**

- Hot: active files from plan manifest
- Warm: `AGENTS.md`, package boundaries, `api-thin-controllers`, `data-repository-pattern`
- Cold: unrelated app-store apps, generated files
- Ignore: `*.generated.ts`, deprecated docs

**Required context**

- Approved spec + plan (artefact3 `instant-meeting.contextmap.md` as reference)

**Verification approach**

- Diff files ⊆ plan manifest ± justified deviations.
- Verifier flags scope creep vs context map.
- Integration tests target real boundaries listed in hot context (not mocks-only when plan says otherwise).

**Ratings:** Impact **H** · Risk **M** (stale map after mid-flight plan change) · Effort **M**

---

## UC-6 — E2E failure triage assist

**SDLC hotspot:** Nightly UI E2E complete regression (~95% negative feedback, ~6 h); Quality Gate sanity TTReaction up to 1 day (source JSON notes).

**Problem:** UI developers and AQA spend sprint time on root-cause reports for parallel pipelines with data dependencies.

**Recommended AI workflow pattern:** **Explore subagent + log summarization** (human classifies, agent clusters)

1. Ingest CI artifact / Report Portal export (failure messages, stack traces, screenshots).
2. Agent clusters failures: env/provisioning, test data, flake, product defect.
3. Human owner confirms cluster before any test or code change.

**Required context**

- Nightly run ID, branch, env config diff
- Test management metadata (case ID → feature area)
- Recent merges to master

**Verification approach**

- No autonomous test edits in v1 — advisory only.
- Each cluster links to at least one exemplar failure + suggested owner (UI dev vs AQA vs DevOps).
- Success metric: time-to-first-root-cause < 4 h for P1 failures (vs 1 day baseline).

**Ratings:** Impact **M** · Risk **H** (wrong auto-fix amplifies flake) · Effort **H** (CI integration, data access)

---

## UC-7 — API / tRPC breaking-change advisory

**SDLC hotspot:** PR and master — API dependency validation (~100% precision when run, rare on PR; breaking changes announced per release on master).

**Problem:** REST v2 and tRPC router changes can break embed consumers and platform OAuth clients without a major-version bump discipline.

**Recommended AI workflow pattern:** **Diff analysis against `api-no-breaking-changes` rule**

- Agent reviews router/controller diff for removed fields, stricter validation, renamed procedures.
- Outputs advisory comment: safe / breaking / needs major version + changelog.

**Required context**

- `api-no-breaking-changes.md`, `calcom-api` skill
- Changed files under `packages/trpc`, `apps/api/v2`
- Existing OpenAPI or tRPC procedure list

**Verification approach**

- Breaking findings require human ack before merge.
- Long term: wire to CI contract test (out of scope for v1 agent-only).
- Cal.diy: platform API consumers listed in skill references.

**Ratings:** Impact **M** · Risk **L** (advisory) · Effort **M**

---

## Cross-cutting comparison

| Pattern | Best for | Avoid when |
|---------|----------|------------|
| Chained skills | Greenfield features | Hotfix < 1 file |
| Read-only subagent | Pre-PR quality gate | Agent has no spec/plan |
| Rule-enforced commands | All agent sessions | — |
| Verification matrix | Features with many ACs | Typo-only PRs |
| Context map | 3+ package touches | Single-file bugfix |
| Log triage subagent | Nightly E2E fires | Local unit test failures |
| API diff advisory | Public API changes | Internal refactor, same contract |
