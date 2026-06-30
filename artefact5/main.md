# Artifact 5 Main

**Feature slug:** `instant-meeting`
**Pull request:**: https://github.com/killroy192/caldiy-masterclass/pull/2

---

### Tests / checks evidence

| Check | Command | Result |
|-------|---------|--------|
| **New unit/component tests (16)** | `TZ=UTC npx vitest run packages/features/bookings/lib/service/StartInstantMeetingService.test.ts packages/trpc/server/routers/loggedInViewer/startInstantMeeting.handler.test.ts apps/web/modules/bookings/components/StartInstantMeetingButton.test.tsx` | **16/16 passed** |
| **Regression: `connectAndJoin`** | `TZ=UTC npx vitest run packages/trpc/server/routers/loggedInViewer/connectAndJoin.handler.test.ts` | **1/1 passed** |
| **Biome (changed files)** | `npx biome check` on service, handler, button | **0 errors** (nursery `useExplicitType` infos + existing `as any` / function-length warnings) |

**AC coverage (from verification plan):**

| AC | Status |
|----|--------|
| AC-1 — Toolbar action | Automated (component + `BookingListContainer` upcoming-only) |
| AC-2 — ACCEPTED 30-min booking | Automated (service happy path) |
| AC-3 — Cal Video location + join URL | Automated (service) |
| AC-4 — Calendar block | **Manual only** (`EventManager` mocked in unit tests) |
| AC-5 — Join + Copy success UI | Automated (component) |
| AC-6 — Host busy | Automated (service + handler + component toast) |
| AC-7 — Cal Video unavailable | Automated (service) |
| AC-8 — No destination calendar | Automated (service + handler) |
| AC-9 — Unauthorized | Implicit via `authedProcedure` |
| AC-10 — Upcoming list + hidden event type | **Partial** (service asserts hidden slug; list UI manual) |
| AC-11 — No `InstantMeetingToken` | Automated (explicit assertion) |

**Not run in this session (recommended before merge):** `yarn type-check:ci --force`, full `TZ=UTC yarn test`, Playwright E2E (planned in artefact3 but not in this commit).

**Verification score (artefact3):** 11/12 — strong spec compliance; maintainability minor deductions (`as any`, 157-line service method).

---

### Review notes

**Architecture (matches D-1, D-2, D-3):**

- **`StartInstantMeetingService`** — pre-flight → find/create hidden `instant-meeting` event type → `ensureAvailableUsers` → `prisma.booking.create` → `EventManager.create` → persist references/metadata; rolls back booking on integration failure.
- **tRPC** — `loggedInViewer.startInstantMeeting` (`authedProcedure`, `{ timeZone }` input); `ErrorWithCode` → `TRPCError` mapping (`CONFLICT` / `PRECONDITION_FAILED`).
- **UI** — `StartInstantMeetingButton` in `BookingListContainer` toolbar, **upcoming tab only**; inline success panel with Join + Copy; invalidates bookings query on success.

**Scope control (no creep):**

- No API v2, schema migration, team/org flows, `InstantMeetingToken`, configurable duration, dedicated `INSTANT_MEETING` webhook, or rate-limit wiring.
- Notifications suppressed by not calling `RegularBookingService` post-create email/SMS paths (OQ-4).

**Reviewer focus areas:**

1. **`as any` bridges** in service (`ensureAvailableUsers`, `getAllCredentialsIncludeServiceAccountKey`) — same pattern as `connectAndJoin.handler.ts`; consider follow-up typing.
2. **`execute()` length** (157 lines) — biome suggests split; acceptable for v1 but worth a refactor PR.
3. **Credential `select` includes `key`** — required for `EventManager`; ensure nothing sensitive is returned to the client (only server-side service).
4. **Manual AC-4** — confirm a real Google/Outlook block after merge in staging.
5. **PR size** — 11 code files (slightly above 10-file guideline; 3 are tests).

---

### Release notes

**feat(bookings): Start instant meeting for logged-in hosts**

Hosts with a connected destination calendar and Cal Video configured can start a 30-minute video meeting from **Bookings → Upcoming**:

- Creates an accepted booking tied to a per-host hidden `instant-meeting` event type.
- Provisions a Daily (Cal Video) room and blocks the host's destination calendar.
- Shows **Join meeting** and **Copy meeting link** after success.

**Clear errors when:**

- No destination calendar is connected.
- Cal Video is disabled or misconfigured.
- The host is busy for the next 30 minutes.

**Not included in v1:** team instant meetings, guest "connect now", configurable duration, API v2, confirmation emails/SMS, E2E Playwright specs.

---

### Rollback plan

| Scenario | Action |
|----------|--------|
| **Post-merge revert** | `git revert 37cf23a54c` (or merge revert PR). No DB migration to roll back. |
| **Runtime disable (no deploy)** | Disable `daily-video` app → feature fails with `instant_meeting_video_unavailable` (AC-7). |
| **UI-only hotfix** | Remove `<StartInstantMeetingButton />` from `BookingListContainer.tsx`; backend mutation remains unused. |
| **Partial failure data** | Orphan Daily rooms may exist if rollback runs after room creation; rooms auto-expire (end + 1h). Existing bookings from successful runs are normal accepted bookings — cancel via standard booking flows. |
| **Hidden event types** | Per-host `instant-meeting` rows may remain after revert; harmless, no user-facing impact. |

No feature flag — rollback is revert deploy or UI removal.

---

### Known risks / limitations

| Risk | Severity | Notes |
|------|----------|-------|
| **Double-click / race** | Medium | Button disabled while pending; `instantMeeting` rate limit (1/10 min) **not wired** — rapid clicks could create two meetings before conflict check sees the first. |
| **Orphan Daily rooms on rollback** | Low | Accepted — rooms expire automatically; same trade-off as other booking flows. |
| **AC-4 not automated** | Low | Calendar write is external; relies on production-proven `EventManager.create` path. |
| **No E2E in this PR** | Low | Unit/component coverage is strong; E2E deferred per plan (steps 8–12). |
| **Generic `BOOKING_CREATED` webhooks** | Low | May fire (D-3 path); dedicated `INSTANT_MEETING` trigger intentionally out of scope. |
| **No feature flag** | Low | Auth-only + upcoming tab; hidden event type limits public exposure. |
| **`as any` type bridges** | Low | Maintainability / future refactor risk, not runtime. |
| **Timezone** | Info | Client sends browser timezone (OQ-5); stored as UTC on booking. |

**Manual verification still recommended:**

1. Happy path: start → join + copy → guest link in incognito.
2. Busy host (existing booking + external calendar block).
3. No calendar / disabled Daily.
4. Booking appears under Upcoming with join affordance.

---

### Final merge / deployability decision

**Recommendation: merge-ready after CI green + manual AC-4 spot-check.**

| Gate | Status |
|------|--------|
| Spec compliance (11 ACs) | Pass — 9 automated, 2 manual |
| Scope control | Pass |
| Test quality (16 new tests) | Pass |
| Regression (`connectAndJoin`) | Pass |
| Lint (Biome) | Pass (warnings only) |
| Type-check (`yarn type-check:ci --force`) | **Pending** — run before merge |
| E2E | Deferred (non-blocking for v1 per plan) |
| Manual staging (AC-4, AC-10) | **Pending** — recommended before production |

**Decision:** Approve for merge once `yarn type-check:ci --force` passes in CI and a quick staging run confirms calendar blocking (AC-4) and upcoming-list visibility (AC-10). No migration or infra changes — standard app deploy only.
