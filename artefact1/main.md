# Artifact 1 Main

**Scope:** AI agent baseline for the Cal.diy codebase  
**Repo URL with committed evidence:** https://github.com/killroy192/caldiy-masterclass  
**Evidence commit:** https://github.com/killroy192/caldiy-masterclass/commit/e106909e67 (merged via PR #3)

---

## 1. [AGENTS.md](https://github.com/killroy192/caldiy-masterclass/blob/main/AGENTS.md)

**Why we need this content in AGENTS.md**

- Gives every agent session the same engineering contract: type safety, security boundaries, PR size limits, and repo layout without re-prompting.
- Centralizes runnable commands (`yarn type-check:ci --force`, `TZ=UTC yarn test`, `yarn biome`) so agents do not guess from `package.json` scripts.
- Defines hard boundaries (never expose `credential.key`, never use `as any`, never put auth in `layout.tsx`) that are easy to violate without explicit guardrails.

**Tradeoffs**

- **Pros:** Single entry point; works across Cursor, Claude Code, and other tools that read `AGENTS.md`; low maintenance compared to duplicating rules in chat.
- **Cons:** Grows long over time; mixes “always do” with command reference; agents may skim and miss modular rules in `.cursor/rules/`.
- **Mitigation:** Link out to focused rule files (e.g. `quality-code-comments.md`) instead of inlining every pattern in `AGENTS.md`.

---

## 2. [calcom-api skill](https://github.com/killroy192/caldiy-masterclass/tree/main/.cursor/skills/calcom-api)

**Why we need this skill**

- Cal.diy API v2 has auth modes (API key vs platform OAuth), versioned headers, and resource-specific payloads that generic LLM knowledge gets wrong.
- The skill bundles reference docs (bookings, event types, slots, webhooks) and required env vars (`CAL_API_KEY`, etc.) so integration work stays consistent.
- Triggers only when building external integrations — avoids loading API surface area during unrelated UI or Prisma work.

**Tradeoffs**

- **Pros:** Accurate, repo-local API guidance; env schema in frontmatter documents secrets without hardcoding them.
- **Cons:** Another large skill directory to keep in sync with API changes; overlaps slightly with human docs in `docs/`.
- **Alternative considered:** Rely on OpenAPI only — rejected because agents need workflow context (when to use which auth header, common pitfalls).

---

## 3. [ci-type-check-first rule](https://github.com/killroy192/caldiy-masterclass/blob/main/.cursor/rules/ci-type-check-first.md)

**Why we need this rule**

- Test failures in this monorepo are often downstream of TypeScript errors; running tests first wastes agent turns.
- Encodes the team’s CI mental model: `yarn type-check:ci --force` → fix types → `TZ=UTC yarn test`.
- Modular rule format (YAML frontmatter + incorrect/correct examples) is easier for agents to retrieve than a paragraph buried in `AGENTS.md`.

**Why we avoided adding more to this rule**

- Branch comparison bash recipes are useful for humans but noisy as an always-on agent rule — kept as optional snippet in the rule file only.
- Did not duplicate full PR creation or Biome workflows here; those live in `ci-git-workflow.md` and `quality-pr-creation.md` to keep rules single-purpose.

**Tradeoffs**

- **Pros:** High impact, small file; directly reduces false “test-only” debugging loops.
- **Cons:** 40+ rules total — agents must rely on section prefixes (`ci-`, `quality-`, `data-`) to find the right one; risk of rule sprawl if every workflow gets its own file.

