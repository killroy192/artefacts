# Artifact 2 Main

**Scope:** Subagents and MCP for AI-assisted SDLC in this workspace  
**Repo URL with committed evidence:** https://github.com/killroy192/caldiy-masterclass (verifier subagent)

---

## 1. [verifier subagent](https://github.com/killroy192/caldiy-masterclass/blob/main/.cursor/agents/verifier.md)

**What the purpose of having this subagent**

- Runs a **read-only** second opinion after implementation: spec/plan compliance, scope creep, missing tests, and security/data risks.
- Separates “builder” from “skeptic” so the main agent does not self-approve merge readiness.
- Standardizes output (spec compliance → evidence → gaps → scope → risks → recommendation) for artefact5 review notes.

**Do we need it locally or as a cloud agent in CI**

| Mode | Recommendation | Rationale |
|------|----------------|-----------|
| **Local subagent (Cursor)** | **Yes — primary** | Needs repo diff, spec/plan files in `artefacts/`, and test output from the parent chat; low latency for interactive feature work. |
| **Cloud agent in CI** | Optional later | Useful for automated PR comments, but requires wiring spec/plan artefacts into CI context and careful token cost; not required for v1 masterclass. |

**Subagent execution output (example structure)**

```
1. Spec compliance — AC-1..AC-N mapped to tests or gaps
2. Verification evidence — commands run and pass/fail
3. Missing tests/checks — e.g. type-check not run
4. Scope creep — files outside plan manifest
5. Security/data risks — credential exposure, auth gaps
6. Merge readiness — approve / conditional / block
```

**Agent chat log**

- Parent agent launches verifier with `readonly: true` after implementation and test run.
- Verifier reads `artefacts/specs/`, `artefacts/plans/`, `artefacts/verification/` and the git diff; does not edit code.
- Parent incorporates findings into artefact5 (tests evidence, review notes, merge decision).
- **Caldiy copy:** https://github.com/killroy192/caldiy-masterclass/blob/main/.cursor/agents/verifier.md (same definition, committed in PR #3).

---

## 2. [MCP config](https://github.com/killroy192/caldiy-masterclass/blob/main/.cursor/mcp.json) + [miro-mcp skill](https://github.com/killroy192/caldiy-masterclass/blob/main/.cursor/skills/miro/SKILL.md)

**Why we need this MCP**

- Planning artefacts (context maps, VSM, SDLC diagrams) benefit from **visual boards**; Miro MCP lets agents read existing boards and create diagrams/docs/tables without manual export/import.
- Complements text-only skills (`generate-spec`, `plan`, `context-map`) — specs stay in git; Miro holds exploratory visuals and workshop output.
- Supports artefact4-style value-stream / SDLC documentation where a board is the source of truth for stakeholders.

**Notes about MCP configuration and safety**

| Topic | Practice |
|-------|----------|
| **Description field** | Each server entry includes a `description` documenting purpose and OAuth requirement — aids review even though Cursor primarily uses `url` / `command`. |
| **Auth** | Official Miro server uses OAuth at `https://mcp.miro.com/` — no tokens in git; complete OAuth in Cursor after clone. |
| **Duplicate servers** | Do **not** add manual `mcp.json` entry if the Miro Cursor Marketplace plugin is already installed — duplicate tools and auth sessions. |
| **Tool approval** | Keep default approval for write tools (`diagram_create`, `doc_update`, `table_sync_rows`); readonly exploration can be allowlisted per team policy. |
| **Secrets** | Never commit `MIRO_OAUTH_TOKEN` or API keys; use `${env:...}` interpolation only in local overrides, not in committed config. |

MCP output: ./mcp-output.jpg

Agent chat log: ./mcp.chatlog.md

**Challenges**

| Challenge | Mitigation |
|-----------|------------|
| OAuth per developer | Document one-time setup in this artefact; use marketplace plugin when possible. |
| Board URL vs ID | Skill documents URL parsing (`moveToWidget`, `focusWidget`); agents must pass full URLs when unsure. |
| Write vs read scope | Verifier subagent forbids external write MCP tools — verification stays repo-local. |
| Drift between skill and Miro API | Skill lists tool names aligned with MCP registry; update skill when Miro ships new tools. |
| Monorepo workspace | MCP config lives in `artefacts`; open multi-root workspace so agents see both `caldiy-masterclass` and `artefacts`. |
