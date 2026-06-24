# Context Map: Start Instant Meeting (Host-Initiated)

**Feature slug:** `feature-start-instant-meeting`  
**Story:** [`Feature.StartInstantMeeting.Story.md`](../stories/Feature.StartInstantMeeting.Story.md)  
**Spec:** [`feature-start-instant-meeting.spec.md`](../specs/feature-start-instant-meeting.spec.md)  
**Plan:** _Not yet produced_ (`artefacts/plans/` is empty)

---

## Hot Context — directly required for implementation

### Spec & Story

| Artefact | Path | Notes |
|----------|------|-------|
| User story | `artefacts/stories/Feature.StartInstantMeeting.Story.md` | Solo-host "start meeting now" with Cal Video; 30-minute fixed duration; shareable link |
| Technical spec | `artefacts/specs/feature-start-instant-meeting.spec.md` | Full spec with decisions D-1 (hidden event type), D-2 (fail if no calendar), D-3 (standard pipeline via `EventManager.create`); 11 acceptance criteria; 4 open questions |

### Active Files — Backend / API

| File | Role | Why it matters |
|------|------|----------------|
| `packages/trpc/server/routers/loggedInViewer/_router.tsx` | tRPC router definition | Pattern for adding the new `authedProcedure` mutation; currently hosts `connectAndJoin`, `eventTypeOrder`, `markNoShow`, etc. |
| `packages/features/bookings/lib/EventManager.ts` | Video + calendar orchestrator | `create(event)` is the D-3 entry point: creates Daily room, calendar events, returns `{ results, referencesToCreate }` for DB persistence |
| `packages/features/bookings/lib/service/RegularBookingService.ts` | Booking creation service (~2.6k lines) | Post-create reference wiring pattern to extract: `eventManager.create()` → `booking.update({ references: { createMany } })` + metadata/location |
| `packages/features/bookings/lib/conflictChecker/checkForConflicts.ts` | Pure conflict-check function | Given buffered busy times, slot start, and event length, returns boolean overlap; used by availability gate |
| `packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts` | Availability gate | Loads per-user availability, runs `checkForConflicts`, throws `ErrorCode.NoAvailableUsersFound` if conflict found |

### Active Files — Video / Calendar

| File | Role | Why it matters |
|------|------|----------------|
| `packages/app-store/dailyvideo/lib/VideoApiAdapter.ts` | Daily.co video adapter | `createMeeting` for normal bookings; also has internal `createInstantMeeting(endTime)` exposed as `createInstantCalVideoRoom` (public room, knocking, expiry = end + 1h) |
| `packages/app-store/dailyvideo/lib/getDailyAppKeys.ts` | Daily API key loader | Reads + Zod-parses `api_key` and `scale_plan` from `daily-video` app slug; throws if missing |
| `packages/app-store/constants.ts` | Location type constants | `DailyLocationType = "integrations:daily"` — the location value for Cal Video bookings |

### Active Files — Frontend

| File | Role | Why it matters |
|------|------|----------------|
| `apps/web/modules/bookings/views/bookings-view.tsx` | Bookings shell (client) | Wraps list/calendar views in `DataTableProvider`; likely insertion point for "Start meeting" action button |
| `apps/web/modules/bookings/components/JoinMeetingButton.tsx` | Join link button | Uses `useJoinableLocation` to resolve joinable URL from `booking.location` / `metadata.videoCallUrl`; reusable in success state |
| `apps/web/app/(use-page-wrapper)/(main-nav)/bookings/[status]/page.tsx` | Bookings page (server) | Auth check + feature flags; permission checks live here per repo conventions |
| `packages/lib/hooks/useCopy.ts` | Clipboard hook | Returns `{ isCopied, copyToClipboard }` with Safari-safe async clipboard and 3s auto-reset; reuse for "Copy link" |

### Active Files — Data Model

| File | Role | Why it matters |
|------|------|----------------|
| `packages/prisma/schema.prisma` | Database schema | `CreationSource` enum (WEBAPP, API_V1, API_V2); `EventType.isInstantEvent` + legacy instant fields; `InstantMeetingToken` model (team-scoped, out of scope); `Booking.eventTypeId` is optional |

### Active Files — i18n

| File | Role | Why it matters |
|------|------|----------------|
| `packages/i18n/locales/en/common.json` | Translation strings | Existing keys: `instant_meeting`, `instant_meetings`, `instant_meeting_with_title`, `join_meeting`; new host-action keys likely needed (e.g. `start_meeting`, `meeting_started`) |

### Legacy / Reference-Only (do not extend)

| File | Role | Why it's reference-only |
|------|------|------------------------|
| `packages/trpc/server/routers/loggedInViewer/connectAndJoin.handler.ts` | Legacy team instant meeting handler | Consumes `InstantMeetingToken`, requires org membership, assigns racing host — explicitly out of scope |
| `apps/web/modules/connect-and-join/connect-and-join-view.tsx` | Legacy connect-and-join UI | Token-based; org-gated; shows upgrade message for non-org users; not the new flow |
| `packages/features/conferencing/lib/videoClient.ts` | Video client helpers | Exports `createInstantMeetingWithCalVideo(endTime)` — defined but **zero callers** in codebase; spec says do not use |

---

## Warm Context — reusable guidance relevant to the feature

### AGENTS.md / CLAUDE.md (repo root)

Both files are identical. Key rules affecting this feature:

- Use `select` not `include` in Prisma queries
- Use `import type` for type imports; direct paths, no barrel imports
- Use `ErrorWithCode` in services, `TRPCError` only in tRPC routers
- Permission checks in `page.tsx`, never `layout.tsx`
- Conventional commits for PR titles (`feat:`, `fix:`)
- PRs < 500 lines, < 10 code files
- All UI strings in `packages/i18n/locales/en/common.json`
- Run `yarn type-check:ci --force` before pushing

### Agent Rules (`.cursor/rules/` → `agents/rules/`)

Rules with direct relevance to this feature:

| Rule file | Relevance |
|-----------|-----------|
| `quality-error-handling.md` | `ErrorWithCode` in the new service; `TRPCError` in the tRPC mutation |
| `quality-pr-creation.md` | Draft PR, < 500 lines, conventional commit title |
| `data-prefer-select-over-include.md` | All Prisma queries in the service must use `select` |
| `data-repository-pattern.md` | DB access in repositories, business logic in services |
| `architecture-page-level-auth.md` | Auth check in the bookings page, not layout |
| `quality-avoid-barrel-imports.md` | Import from source files directly |
| `quality-code-comments.md` | Comment only non-obvious "why" |
| `ci-type-check-first.md` | Run type check before assuming test failures |
| `testing-timezone.md` | Run tests with `TZ=UTC` |
| `testing-coverage-requirements.md` | 80%+ coverage on new code |
| `patterns-factory-pattern.md` | May apply if branching between event-type creation vs reuse |

### Commands Reference (`agents/commands.md`)

| Command | When to use |
|---------|-------------|
| `yarn type-check:ci --force` | Before every push |
| `yarn biome check --write .` | Lint + format changed files |
| `TZ=UTC yarn test` | Run unit tests |
| `yarn prisma generate` | If schema changes are needed (not expected for this feature) |
| `yarn dev` | Local dev server |

### Knowledge Base (`agents/knowledge-base.md`)

| Topic | Relevance |
|-------|-----------|
| Cal.diy calendar events | iCalUID ending in `@Cal.diy` identifies Cal.diy bookings in external calendars |
| UI locations | Bookings view alignment patterns for tabs/filters |
| Round-robin | Not applicable (single host), but `getLuckyUser.ts` shows availability check patterns |
| Workflows vs webhooks | `BOOKING_CREATED` webhook may fire via standard path (OQ-4 from spec) |

### Spec Workflow (`SPEC-WORKFLOW.md`)

Opt-in spec-driven development with artifacts under `specs/{feature}/`. ADR-style decisions in `decisions.md`. The instant meeting spec already follows this pattern with D-1, D-2, D-3 decisions embedded in the spec itself (in `artefacts/specs/` rather than `caldiy-masterclass/specs/`).

### Rate Limiting

| File | Detail |
|------|--------|
| `packages/lib/rateLimit.ts` | Pre-defined `instantMeeting` bucket: 1 request per 10 minutes; **not wired to any caller** yet; can be activated in the new mutation if abuse prevention is desired |

---

## Cold Context — investigate only if needed during implementation

### Relevant Repo Areas

| Area | Path | When to investigate |
|------|------|---------------------|
| Booking creation internals | `packages/features/bookings/lib/handleNewBooking/` | If extracting creation slice from `RegularBookingService` is non-trivial |
| EventType creation/lookup | `packages/features/eventtypes/` | When implementing D-1 (hidden event type per host — find-or-create pattern) |
| User availability helpers | `packages/features/bookings/lib/getUserAvailability.ts` | If `ensureAvailableUsers` alone is insufficient for single-host busy check |
| Destination calendar lookup | `packages/features/bookings/lib/getBookingData*`, calendar service files | Needed for D-2 pre-flight (fail if no destination calendar) |
| Calendar credentials | `packages/features/bookings/lib/getCalendar*` | Understanding how `EventManager` resolves calendar credentials for the host |
| App-store app enablement | `packages/app-store/` app config patterns | Checking if Daily Video app is enabled on the instance |
| Public event instant-meeting gating | `packages/features/eventtypes/lib/getPublicEvent.ts` | Legacy `isCurrentlyAvailable` / `isInstantEvent` — reference only for understanding old behavior |
| Webhook trigger on booking create | `packages/features/bookings/lib/` webhook/notification handlers | If OQ-4 (suppress notifications) needs implementation |
| Migration history | `packages/prisma/migrations/20260430000000_drop_instant_meeting_webhook_trigger/` | Context on why `INSTANT_MEETING` webhook trigger was removed |

### Documentation

| Doc | Path | When useful |
|-----|------|-------------|
| CONTRIBUTING.md | `caldiy-masterclass/CONTRIBUTING.md` | File naming conventions (`Prisma*Repository`, `*Service`), PR workflow |
| App-store CONTRIBUTING.md | `packages/app-store/CONTRIBUTING.md` | Only if modifying Daily Video app integration patterns |
| OpenAPI spec | `docs/api-reference/v2/openapi.json` | Only relevant if API v2 exposure is added (currently out of scope) |
| DataTable guide | `packages/features/data-table/GUIDE.md` | Only if bookings list UI requires DataTable integration |

### Past PRs / Related Code

| Topic | What to look for |
|-------|------------------|
| `connectAndJoin` test | `connectAndJoin.handler.test.ts` — should remain green; do not break |
| Booking creation tests | Existing tests near `packages/trpc/server/routers/loggedInViewer/` — pattern for new test files |
| Cal Video booking flow | Trace a normal Cal Video booking end-to-end through `RegularBookingService` → `EventManager.create` to understand reference shape |
| Hidden event types | Search for `hidden: true` in event type creation patterns to find prior art for D-1 |

### Monitoring / Logs

Not directly relevant pre-implementation. After deployment:
- Check `BOOKING_CREATED` webhook firing behavior (OQ-4)
- Monitor Daily room creation success/failure rates
- Track `instantMeeting` rate-limit bucket if wired

### Existing ADRs

| ADR | Location | Relevance |
|-----|----------|-----------|
| ADR-001: Cancellation reason storage | `caldiy-masterclass/specs/cancellation-reason-requirement/decisions.md` | Unrelated to instant meeting but shows ADR format used in the repo |

---

## Resources to Ignore

| Resource | Reason |
|----------|--------|
| `*.generated.ts` files | Auto-generated by app-store-cli; never modify directly |
| `packages/prisma/migrations/` (historical) | Past migrations are read-only context; no schema migration expected for this feature |
| `apps/docs/` (Nextra documentation site) | Public-facing docs; not related to this feature |
| `packages/app-store/` (non-Daily integrations) | Only `dailyvideo` is in scope; other app-store packages are irrelevant |
| Legacy instant-meeting schema fields (`isInstantEvent`, `instantMeetingScheduleId`, `instantMeetingExpiryTimeOffsetInSeconds`, `instantMeetingParameters`) | Spec explicitly excludes cleanup of these fields |
| `InstantMeetingToken` model and all related code | Team-scoped token-race model; explicitly out of scope |
| Team/org routing code | Feature is single-host only; no org gates |
| `apps/api/v2/` | API v2 exposure is out of scope |
| `packages/embeds/` | Embed SDK; not relevant |
| `packages/platform/` | Platform/OAuth clients; not relevant |

---

## Open Questions Requiring Resolution Before/During Implementation

| # | Question | Impact on context |
|---|----------|-------------------|
| OQ-1 | Where should the "Start meeting" entry-point live? (bookings toolbar vs shell vs Kbar) | Determines which frontend files are hot context |
| OQ-3 | Booking title and attendee rows? | Affects i18n keys needed and booking list display |
| OQ-4 | Should confirmation emails/SMS fire? | Determines whether notification suppression logic is needed in the service |
| OQ-5 | Which timezone defines "now"? | Affects `startTime`/`endTime` computation in the service |
