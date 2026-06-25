# Start Instant Meeting — Implementation Plan

**Feature slug:** `instant-meeting` 
**Spec:** [`artefacts/artefact3-example/instant-meeting.spec.md`](./instant-meeting.spec.md)  
**Context map:** [`artefacts/artefact3-example/instant-meeting.contextmap.md`](./instant-meeting.contextmap.md)

---

## Resolved Open Questions

- **OQ-1 (UI placement):** Bookings page toolbar (right side of `BookingListContainer` flex row)
- **OQ-3 (Title/attendees):** Localized template with host name (e.g. "Instant meeting — {host}")
- **OQ-4 (Notifications):** Suppress — no emails/SMS for instant meetings
- **OQ-5 (Timezone):** Browser/client timezone sent as input; stored as UTC

## Confirmed Facts

- All 18 referenced codebase files exist and are functional
- `EventManager.create(evt)` handles Cal Video room creation + calendar event + returns `{ results, referencesToCreate }` — this is the D-3 entry point
- `createBooking()` in `handleNewBooking/createBooking.ts` inserts the booking row; references are persisted in a separate `booking.update()`
- `ensureAvailableUsers` throws `ErrorCode.NoAvailableUsersFound` when no users pass conflict checks
- Hidden event types are created with `hidden: true` via the standard `EventType` create path (onboarding `secret_meeting` pattern)
- Destination calendar resolved via: event-type destination calendar → user destination calendar → null
- No "Start meeting" or "New booking" action exists in the bookings toolbar today
- `BookingListContainer` toolbar (line ~160-208) is the insertion point — after the `grow` spacer div
- tRPC mutations in `loggedInViewer` follow: `*.schema.ts` + lazy-imported `*.handler.ts` + `authedProcedure.input(...).mutation(...)`
- Cal Video credentials fall back to `FAKE_DAILY_CREDENTIAL` / `getDailyAppKeys()` inside `EventManager`
- `useCopy` hook provides `{ isCopied, copyToClipboard }` with Safari-safe clipboard
- `JoinMeetingButton` / `useJoinableLocation` resolve joinable URLs from `booking.location` + `metadata.videoCallUrl`
- `instantMeeting` rate-limit bucket exists (1 req / 10 min) but is unwired
- i18n keys `instant_meeting`, `instant_meeting_with_title`, `join_meeting` already exist

## Assumptions

- `EventManager.create()` can be called outside `RegularBookingService` with a manually built `CalendarEvent` and organizer credentials — no hidden coupling to the full booker pipeline
- The host's organizer credentials (calendar + video) can be loaded via `getAllCredentialsIncludeServiceAccountKey` with a user object and event type, same as `RegularBookingService` does at line ~1158
- `ensureAvailableUsers` can work with a single-user event type (array of one `IsFixedAwareUser`) for the host-only case
- `createBooking()` from `handleNewBooking/createBooking.ts` can be called directly with a minimal `reqBody` shape, or the service can use a direct `prisma.booking.create()` with the known required fields (`uid`, `title`, `startTime`, `endTime`, `status`, `location`, `user`, `eventType`, `iCalUID`)
- Suppressing notifications means the service does not call the email/SMS scheduling functions that `RegularBookingService` invokes post-create
- The `CalendarEvent` built by the service (with `destinationCalendar`, `organizer`, `location: "integrations:daily"`) is sufficient for `EventManager.create()` to produce a Daily room + calendar block

## Files and Modules Involved

### PR 1 — Backend (service + tRPC mutation + i18n)

| File | Action | Purpose |
|------|--------|---------|
| `packages/features/bookings/lib/service/StartInstantMeetingService.ts` | **Create** | Core service: pre-flight checks, find-or-create hidden event type, conflict check, create booking, `EventManager.create()`, persist references, rollback on failure |
| `packages/trpc/server/routers/loggedInViewer/startInstantMeeting.schema.ts` | **Create** | Zod input schema (timezone string) and output type |
| `packages/trpc/server/routers/loggedInViewer/startInstantMeeting.handler.ts` | **Create** | Handler: extract user from ctx, call service, map result to tRPC response |
| `packages/trpc/server/routers/loggedInViewer/_router.tsx` | **Edit** | Wire new `startInstantMeeting` mutation |
| `packages/i18n/locales/en/common.json` | **Edit** | Add new keys: `start_meeting`, `instant_meeting_started`, `instant_meeting_host_busy`, `instant_meeting_no_calendar`, `instant_meeting_video_unavailable` |

### PR 2 — Frontend (button + success state)

| File | Action | Purpose |
|------|--------|---------|
| `apps/web/modules/bookings/components/StartInstantMeetingButton.tsx` | **Create** | Button component with loading/error/success states; includes inline success panel with Join + Copy |
| `apps/web/modules/bookings/components/BookingListContainer.tsx` | **Edit** | Insert `StartInstantMeetingButton` in toolbar row |

## Implementation Steps

### Test-First Policy (mandatory)

For every implementation step below where behavior is clear, use strict red → green:

1. Write/adjust the test first.
2. Run only the new/affected tests and confirm they fail (red).
3. Implement the minimum code to satisfy the test.
4. Re-run the same tests and confirm pass (green).
5. Run related regression tests before moving to the next step.

### Suggested test files to add

- `packages/features/bookings/lib/service/StartInstantMeetingService.test.ts`
- `packages/trpc/server/routers/loggedInViewer/startInstantMeeting.handler.test.ts`
- `apps/web/modules/bookings/components/StartInstantMeetingButton.test.tsx`
- `apps/web/playwright/start-instant-meeting.e2e.ts` (select ACs only; see feasibility section)

### PR 1: Backend — Service + tRPC Mutation

**Step 1: Create tRPC schema** (`startInstantMeeting.schema.ts`)
- Input: `{ timeZone: z.string() }` (browser timezone from client)
- Output type: `{ bookingUid: string; bookingId: number; meetingUrl: string; meetingPassword?: string }`
- **Test-first:** add schema tests for valid timezone, missing timezone, and invalid type in `startInstantMeeting.handler.test.ts` (or dedicated schema test) before wiring the mutation.

**Step 2: Create `StartInstantMeetingService`**

The service orchestrates these sub-steps in order:

1. **Load user with destination calendar** — query `prisma.user.findUnique` with `select` for `id`, `email`, `name`, `locale`, `timeZone`, `destinationCalendar`, `username`. If `destinationCalendar` is null, throw `ErrorWithCode("instant_meeting_no_calendar")`
2. **Check Cal Video availability** — query `prisma.app.findUnique({ where: { slug: "daily-video" }, select: { enabled, keys } })`. Validate keys via `dailyAppKeysSchema.safeParse`. If disabled or invalid, throw `ErrorWithCode("instant_meeting_video_unavailable")`
3. **Compute time window** — `startTime = new Date()` (UTC), `endTime = startTime + 30 minutes`. Format as ISO strings
4. **Find or create hidden event type** — query `prisma.eventType.findFirst({ where: { userId, slug: "instant-meeting", hidden: true } })`. If not found, create one with `{ title: "Instant meeting", slug: "instant-meeting", length: 30, hidden: true, locations: [{ type: "integrations:daily" }], userId, owner: { connect: { id: userId } } }`
5. **Conflict check** — call `ensureAvailableUsers` with the event type (wrapping host as single `IsFixedAwareUser` with `isFixed: true`), `{ dateFrom: startTime, dateTo: endTime, timeZone }`. Catch `NoAvailableUsersFound` and rethrow as `ErrorWithCode("instant_meeting_host_busy")`
6. **Build CalendarEvent** — construct the `CalendarEvent` object:
   - `type`: event type title
   - `title`: localized `instant_meeting_with_title` with host name
   - `startTime` / `endTime`: ISO strings
   - `organizer`: `{ email, name, timeZone, language, id }`
   - `attendees`: `[]` (no guests at creation time)
   - `location`: `"integrations:daily"`
   - `destinationCalendar`: `[user.destinationCalendar]`
   - `uid`: generate via `short-uuid`
7. **Create booking row** — `prisma.booking.create()` with `uid`, `title`, `startTime`, `endTime`, `status: "ACCEPTED"`, `location: "integrations:daily"`, `eventType: { connect }`, `user: { connect }`, `destinationCalendar: { connect }`, `iCalUID`, `creationSource: "WEBAPP"`
8. **Load credentials + create EventManager** — `getAllCredentialsIncludeServiceAccountKey(user, eventType)` → `refreshCredentials` → `new EventManager({ ...user, credentials })`
9. **Run `EventManager.create(evt)`** — returns `{ results, referencesToCreate }`. Extract `videoCallUrl` from `evt.videoCallData?.url` or results
10. **Persist references + metadata** — `prisma.booking.update({ where: { uid }, data: { location: evt.location, metadata: { videoCallUrl }, references: { createMany: { data: referencesToCreate } } } })`
11. **Rollback on failure** — wrap steps 8-10 in try/catch. If `EventManager.create()` or reference persistence fails, delete the booking row (`prisma.booking.delete({ where: { id } })`) and rethrow
12. **Return** — `{ bookingUid, bookingId, meetingUrl: videoCallUrl, meetingPassword }`

Note: by not calling the email/SMS scheduling functions that `RegularBookingService` invokes after step 10, notifications are implicitly suppressed (OQ-4).

- **Test-first (service):** build tests in `StartInstantMeetingService.test.ts` in this order:
  1. host has no destination calendar → throws `instant_meeting_no_calendar`, no booking write
  2. Daily app disabled/invalid keys → throws `instant_meeting_video_unavailable`, no booking write
  3. host busy overlap (`ensureAvailableUsers`/conflict path) → throws `instant_meeting_host_busy`, no booking write
  4. happy path → booking `ACCEPTED`, 30-minute window, Daily location, references/metadata persisted
  5. rollback path (`EventManager.create` throws) → booking deleted, error rethrown
  6. no `InstantMeetingToken` side effects (assert no create calls)

**Step 3: Create tRPC handler** (`startInstantMeeting.handler.ts`)
- Extract `ctx.user` (id, email, name, locale, timeZone)
- Instantiate and call `StartInstantMeetingService`
- Catch `ErrorWithCode` and map to `TRPCError` (`host_busy` → `CONFLICT`, `no_calendar` → `PRECONDITION_FAILED`, `video_unavailable` → `PRECONDITION_FAILED`)
- **Test-first:** in `startInstantMeeting.handler.test.ts`, add cases for success mapping + each error mapping + unauthorized caller.

**Step 4: Wire into router** (`_router.tsx`)
- Import `ZStartInstantMeetingInputSchema`
- Add `startInstantMeeting: authedProcedure.input(ZStartInstantMeetingInputSchema).mutation(...)` with lazy-imported handler
- Add to handler cache type
- **Test-first:** add a router-level smoke test that confirms the procedure is present and rejects unauthenticated calls.

**Step 5: Add i18n strings** (`common.json`)
- `"start_meeting": "Start meeting"`
- `"instant_meeting_started": "Meeting started"`
- `"instant_meeting_host_busy": "You have a conflicting event in the next 30 minutes"`
- `"instant_meeting_no_calendar": "Connect a calendar to start instant meetings"`
- `"instant_meeting_video_unavailable": "Video conferencing is not available"`
- `"copy_meeting_link": "Copy meeting link"`
- **Test-first (where possible):** assert surfaced error keys/messages in handler/component tests instead of relying on manual checks.

### PR 2: Frontend — Button + Success State

**Step 6: Create `StartInstantMeetingButton` component**
- Uses `trpc.viewer.loggedInViewerRouter.startInstantMeeting.useMutation()`
- Passes `{ timeZone: Intl.DateTimeFormat().resolvedOptions().timeZone }` as input (OQ-5)
- Three visual states:
  - **Idle:** Primary button labeled `t("start_meeting")` with video icon
  - **Loading:** Button disabled with spinner
  - **Success:** Inline panel or dialog showing:
    - `JoinMeetingButton` pointing to `meetingUrl` (reuse existing component pattern — pass `location` and `metadata: { videoCallUrl }` with `bookingStatus: "ACCEPTED"`)
    - "Copy link" button using `useCopy` + `showToast(t("link_copied"), "success")`
  - **Error:** `showToast(err.message, "error")` — messages come from i18n via tRPC error mapping
- On success, invalidate `utils.viewer.bookings` to refresh the list
- **Test-first:** in `StartInstantMeetingButton.test.tsx`, add cases for idle render, loading disabled state, success panel with join/copy controls, and error toast display.

**Step 7: Add button to bookings toolbar** (`BookingListContainer.tsx`)
- Insert `<StartInstantMeetingButton />` in the toolbar flex row at line ~190, after the `<div className="hidden grow md:block" />` spacer, before `<DataTableSegment.Select />`
- Only render when `status === "upcoming"` (no point starting a meeting from past/cancelled views)
- **Test-first:** add/extend container test to assert button visible only in upcoming status and hidden in past/cancelled tabs.

### PR 3: E2E Evidence for "Yes" Feasibility ACs (unit-covered)

Add Playwright specs in `apps/web/playwright/start-instant-meeting.e2e.ts` as a separate implementation phase after unit/component tests are green.

**Step 8: E2E for AC-1 (action visibility)**
- Precondition: AC-1 already covered by component/unit tests (`BookingListContainer` / button render tests).
- E2E scenario: logged-in host opens `/bookings/upcoming` and sees `Start meeting` action.

**Step 9: E2E for AC-5 (success UI affordances)**
- Precondition: AC-5 already covered by component tests (`StartInstantMeetingButton.test.tsx`).
- E2E scenario: host starts meeting successfully; success state shows Join and Copy controls and copy feedback.

**Step 10: E2E for AC-6 (busy conflict error path)**
- Precondition: AC-6 already covered by service/handler unit tests.
- E2E scenario: seed overlapping booking via fixtures/prisma; attempt start meeting; assert user-visible conflict error and no success state.

**Step 11: E2E for AC-8 (no destination calendar error path)**
- Precondition: AC-8 already covered by service/handler unit tests.
- E2E scenario: seed user without destination calendar; attempt start meeting; assert actionable error to connect calendar.

**Step 12: E2E for AC-9 (unauthorized protection)**
- Precondition: AC-9 already covered by router/handler unit tests.
- E2E scenario: unauthenticated user cannot use host flow (redirect/login or unauthorized response when invoking the endpoint).

**Execution order (red → green)**
- For each step 8-12: write the E2E first, run targeted Playwright spec and confirm red, implement/fix behavior, rerun and confirm green.
- Keep each E2E mapped to a single primary AC for traceability.

### Red/Green run sequence

- Step 1-5 (backend): run targeted Vitest files after each test addition, confirm red, implement, confirm green.
- Step 6-7 (frontend): run targeted component tests after each assertion change, confirm red→green.
- Step 8-12 (e2e): run targeted Playwright file after each new scenario, confirm red→green.
- End of each PR: run `TZ=UTC yarn test` for affected packages, then `yarn type-check:ci --force`, then `yarn biome check --write` on changed paths.

## E2E Feasibility Inspection (for AC evidence)

E2E coverage is feasible. The repo already has Playwright booking flows under `apps/web/playwright`, authenticated fixtures, booking fixtures, and clipboard permissions enabled in Playwright config.

### AC-by-AC E2E suitability

| AC | E2E feasibility | Notes |
|----|-----------------|-------|
| AC-1 | **Yes** | Assert start button visible on `/bookings/upcoming` for logged-in host |
| AC-2 | **Partial** | Trigger action in UI, then verify booking row appears; exact DB fields still best in unit/integration |
| AC-3 | **Partial** | UI can assert join action enabled; precise reference persistence is better as backend test |
| AC-4 | **Limited** | External calendar side effect is hard for deterministic E2E; keep manual + backend integration evidence |
| AC-5 | **Yes** | Assert success state, Join button, Copy link success feedback |
| AC-6 | **Yes** | Seed conflicting booking via fixtures/prisma, assert mutation fails and UI error shown |
| AC-7 | **Possible but brittle** | Can toggle Daily app config via prisma in test setup; keep unit test as primary evidence |
| AC-8 | **Yes** | Seed user without destination calendar, assert clear UI error |
| AC-9 | **Yes** | Unauthenticated navigation/API request should fail (already standard in app) |
| AC-10 | **Partial** | UI list presence is testable; hidden event-type linkage best asserted by backend tests |
| AC-11 | **No (UI), Yes (backend tests)** | Token non-creation should be verified by unit/integration DB assertions |

### Recommended E2E scope for this feature

- AC-1 visibility E2E on upcoming bookings page.
- AC-5 happy-path E2E for Join + Copy success state.
- AC-6 conflict-path E2E with pre-seeded overlap.
- AC-8 no-calendar E2E with pre-seeded user state.
- AC-9 unauthorized E2E (login/unauthorized guard).
- Keep AC-7 Cal Video misconfiguration primarily in unit tests to avoid flaky environment-dependent E2E.

## Risks

| Risk | Mitigation |
|------|------------|
| `ensureAvailableUsers` expects a full event-type shape with `users[]` array of `IsFixedAwareUser` — mismatch with minimal hidden event type | Verify the minimum required fields; construct the host as `{ ...user, isFixed: true }` and build only the fields `ensureAvailableUsers` actually reads |
| `EventManager.create()` may have hidden dependencies on `RegularBookingService` state (e.g. `CalendarEventBuilder` side effects) | Test with a manually constructed `CalendarEvent` early; the method's public signature accepts a plain `CalendarEvent` object |
| Race condition: two rapid clicks could create two instant meetings before conflict check catches the first | Wire the existing `instantMeeting` rate-limit bucket (1 req / 10 min) in the handler; disable button during mutation |
| Rollback may leave orphan Daily room if room was created but booking delete fails | Accept this risk — Daily rooms auto-expire (endTime + 1h); same trade-off as existing booking cancellation flows |
| `getAllCredentialsIncludeServiceAccountKey` may need more user/event-type fields than the service loads | Check the function's required input shape and ensure the select query covers it |

## Non-Goals (explicit)

- No API v2 endpoint
- No team/org flows or `InstantMeetingToken`
- No configurable duration (fixed 30 min)
- No custom video provider selection
- No dedicated `INSTANT_MEETING` webhook trigger
- No cleanup of legacy instant-meeting schema fields
- No Kbar/global shell shortcut (defer to future)
- No mobile app support

## Verification (mapped to Acceptance Criteria)

| AC | Criterion | Verified by |
|----|-----------|-------------|
| AC-1 | Host sees "Start meeting" in bookings toolbar | Unit/component test (toolbar/button render) + E2E (`start-instant-meeting.e2e.ts`) + manual spot-check |
| AC-2 | Creates ACCEPTED booking with 30-min window | Unit test: assert booking status + times; manual: DB check |
| AC-3 | Sets location to Cal Video + stores video join URL | Unit test: assert `location` + `metadata.videoCallUrl` + `references` |
| AC-4 | Blocks host's calendar for 30 min | Manual: check external calendar after creation |
| AC-5 | Success UI shows Join + Copy link | Component test (success state controls) + E2E happy path + manual spot-check |
| AC-6 | Fails with error when host is busy | Unit test (busy conflict) + E2E conflict scenario + manual spot-check |
| AC-7 | Fails when Cal Video disabled/misconfigured | Unit test: mock disabled app → assert error + no records |
| AC-8 | Fails when no destination calendar | Unit test (no destination calendar) + E2E no-calendar scenario + manual spot-check |
| AC-9 | Unauthenticated users get unauthorized | Unit/router test (`authedProcedure`) + E2E unauthorized guard scenario |
| AC-10 | Booking appears in upcoming list with hidden event type | Manual: check bookings list + DB `eventTypeId` |
| AC-11 | No `InstantMeetingToken` created | Unit test: assert no `InstantMeetingToken` rows after creation |

## Smallest Safe First Step

**PR 1** is the smallest safe first step — it delivers the complete backend (service + mutation) that can be tested via tRPC client or API call before any UI exists. It touches ~5 files and should stay under 300 lines of new code.
