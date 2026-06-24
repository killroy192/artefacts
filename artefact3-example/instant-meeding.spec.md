# Spec: Start Instant Meeting (Host-Initiated)

**Feature slug:** `instant-meeting`  
**Story:** [`Feature.StartInstantMeeting.Story.md`](../stories/Feature.StartInstantMeeting.Story.md)  

---

## Business Goal

Give logged-in hosts a fast, in-app way to start an ad-hoc 30-minute video call and share a join link, without creating a public booking page or picking a future slot. This restores a practical “meet now” capability for solo/self-hosted Cal.diy users after the legacy team/org instant-meeting flow was removed.

## User / System Problem

Hosts who need an immediate call today must either book through the normal public flow (pick a time, share a booking page) or leave Cal.diy for an external tool. The product has no host-initiated “start meeting now” path. Remaining instant-meeting code targets the old guest-initiated, team-token race model and does not serve solo hosts.

## Current Behavior

### Product / UX

- No “Start meeting” (or equivalent) action exists in the logged-in host experience (bookings list, shell navigation, event types).
- The only live instant-meeting UI remnant is the org-only **Connect and join** page (`apps/web/modules/connect-and-join/connect-and-join-view.tsx`), which calls `loggedInViewerRouter.connectAndJoin` with a token query param. Non-org users see an upgrade message, not a host-initiated flow.
- Public event pages still expose legacy instant-event concepts (`isInstantEvent`, “Connect now” modal gating via `isCurrentlyAvailable` in `getPublicEvent.ts`), but that is guest-initiated and tied to event-type instant-meeting schedules—not host-initiated.

### API / backend

- `connectAndJoin.handler.ts` accepts a team `InstantMeetingToken`, assigns the booking to a racing org member, and returns a meeting URL. It requires organization membership and does not create bookings from scratch.
- No tRPC mutation or REST endpoint exists to let a logged-in host start an instant meeting without a pre-existing token.
- `createInstantMeetingWithCalVideo` in `packages/features/conferencing/lib/videoClient.ts` can create a Daily room from an end time, but it is **not called anywhere** in the codebase today.
- A dedicated `instantMeeting` rate-limit bucket exists in `packages/lib/rateLimit.ts` but has no active caller for a new host flow.
- `INSTANT_MEETING` was removed from `WebhookTriggerEvents` (migration `20260430000000_drop_instant_meeting_webhook_trigger`).

### Data model

- `InstantMeetingToken` (team-scoped, optional `bookingId`) remains in Prisma schema; no `create` usage found in application code.
- `EventType` still has legacy fields: `isInstantEvent`, `instantMeetingScheduleId`, `instantMeetingExpiryTimeOffsetInSeconds`, `instantMeetingParameters`.
- `Booking.eventTypeId` is optional; bookings normally reference an event type. `CreationSource` enum values are `WEBAPP`, `API_V1`, `API_V2` only.

### Video and calendar integrations

- Cal Video uses location type `integrations:daily` (`DailyLocationType`). Rooms are created via `VideoApiAdapter` (`packages/app-store/dailyvideo/lib/VideoApiAdapter.ts`); `createInstantCalVideoRoom` builds a public Daily room with expiry derived from meeting end time (+1 hour buffer).
- Normal bookings use `EventManager.create` to create dedicated video meetings and calendar events on the host’s destination calendar when location is a dedicated integration.
- Daily availability depends on instance `daily-video` app keys (`getDailyAppKeys` reads slug `daily-video`); `EventManager` also checks `app.enabled` and valid keys before falling back to Cal Video.
- Busy/conflict detection for bookings uses `checkForConflicts` (`packages/features/bookings/lib/conflictChecker/checkForConflicts.ts`) against buffered busy times from calendars and existing bookings (`ensureAvailableUsers` in the regular booking path).

### Existing join / copy patterns

- `JoinMeetingButton` + `useJoinableLocation` resolve a joinable URL from `booking.location` and/or `metadata.videoCallUrl`.
- Copy-to-clipboard is available via `useCopy` (`packages/lib/hooks/useCopy.ts`) and used in event-type permalink flows.

## Expected Behavior

- A **logged-in host** can trigger a single primary action (e.g. “Start meeting”) from the Cal.diy app.
- On success, the system:
  - Creates an **accepted** booking for **now → now + 30 minutes** (fixed duration).
  - Sets location to **Cal Video** (`integrations:daily`) using instance Daily configuration.
  - Creates a **Daily video room** and persists meeting reference data on the booking (consistent with normal Cal Video bookings).
  - **Blocks the host’s calendar** for the 30-minute slot (same outcome as a normal accepted booking with a connected destination calendar).
  - Returns a **shareable guest join URL** (the video meeting link—simplest form, not a booking-page URL).
- The host sees a minimal **success state** with:
  - **Join now** — opens the video meeting (host may use owner token behavior as today’s Cal Video bookings do).
  - **Copy link** — copies the guest join URL to clipboard.
- If the host is **busy** for the interval **now through now + 30 minutes** (overlapping accepted bookings and/or external calendar busy times), the action **fails with a clear error** and **creates nothing** (no orphan booking, room, or calendar event).
- If **Cal Video is unavailable** (app disabled, missing/invalid `DAILY_API_KEY`, or room creation failure), the action **fails with a clear error** and **creates nothing**.
- If the host has **no connected destination calendar**, the action **fails with a clear error** (e.g. prompt to connect a calendar) and **creates nothing**.
- Each instant meeting booking is tied to a **hidden internal event type** owned by the host (dedicated slug, not shown on the public booking page).
- Orchestration uses the **standard booking pipeline** (D-3): create booking row, then `EventManager.create` for Cal Video room + calendar block + references — same shape as normal Cal Video bookings.
- The flow is **host-only**; there is no public guest “connect now” page or team token race.
- Generic **booking-created** side effects (e.g. standard `BOOKING_CREATED` webhooks, confirmation emails) may occur unless explicitly gated (see OQ-4); dedicated instant-meeting webhooks remain out of scope.

## Technical Scope

| Layer | In scope |
|-------|----------|
| **Frontend (apps/web)** | Minimal UI: one entry-point control, loading/error states, success state with Join + Copy. Likely near bookings (`bookings-view` / `BookingListContainer`) or shell, but exact placement is an open question. New UI strings in `packages/i18n/locales/en/common.json`. Reuse `JoinMeetingButton` / `useCopy` where appropriate. |
| **API (packages/trpc)** | New authenticated mutation under `loggedInViewerRouter` (or equivalent viewer router), following `authedProcedure` patterns. Input: none or minimal; server derives host from session. Output: booking identifier, meeting/join URL, and fields needed for success UI. |
| **Services / booking** | Dedicated **`StartInstantMeetingService`** (name illustrative) that reuses booking building blocks without the public booker handler: pre-flight checks → conflict check via `ensureAvailableUsers` / `getUserAvailability` → `createBooking` → `EventManager.create` → persist references/location (D-3). Does **not** use `createInstantMeetingWithCalVideo` or manual reference/calendar wiring. Roll back booking if integration step fails. |
| **Video** | Cal Video only (`integrations:daily`); instance-level Daily keys via `daily-video` app. |
| **Data** | Use a **hidden `EventType` per host** (see Decisions). Prefer **no schema migration**; reuse existing `EventType.hidden` and Cal Video location fields. Do **not** revive `InstantMeetingToken` or `connectAndJoin` for this feature. |

**Out of this spec’s implementation scope (see Non-Goals):** API v2 public endpoints, mobile apps, legacy instant-event public booker UI, team/org flows, configurable duration, per-meeting video provider choice.

## Non-Goals

- Restoring Cal.com-style team instant meeting (`InstantMeetingToken`, guest “Connect now”, `connectAndJoin`, push notifications to team members).
- Guest-initiated instant booking on public event types (`isInstantEvent` booker flow).
- Configurable meeting duration (fixed 30 minutes only).
- Custom video provider selection per instant meeting.
- Dedicated `INSTANT_MEETING` webhook trigger (removed from product; generic `BOOKING_CREATED` may fire if standard creation path is used).
- Cleaning up all legacy instant-meeting schema fields, translations, or dead code unless required for the new flow.
- API v2 exposure of host instant meeting (unless explicitly added later).

## Constraints

- **Auth:** Host must be logged in; permission checks belong in `page.tsx` for any new route, not `layout.tsx` (repo convention).
- **Monorepo patterns:** Direct imports (no barrel files), `import type` for types, `TRPCError` in tRPC handlers, `ErrorWithCode` in services, `select` in Prisma queries, Biome formatting.
- **i18n:** All new UI strings in `packages/i18n/locales/en/common.json`.
- **Video:** `integrations:daily` only; requires `daily-video` app enabled with valid keys on the instance.
- **Concurrency:** Must reject when host has any conflict overlapping `[now, now + 30 min]`—same classes of busy data as normal booking availability (existing bookings + connected calendar busy times).
- **Atomicity:** Story requires no partial artifacts on failure (booking without room, room without booking, etc.).
- **Destination calendar:** Host must have a connected destination calendar before starting; otherwise fail early with a user-visible error (no booking, room, or calendar side effects).
- **Event type:** Each host uses a hidden internal event type (30-minute length, `integrations:daily` location); not exposed on the public profile.
- **Orchestration (D-3):** Reuse `EventManager.create` and post-create reference persistence from the normal Cal Video booking path. Do **not** call the full `RegularBookingService` HTTP/booker entrypoint; extract only the creation slice. Do **not** use `createInstantMeetingWithCalVideo`.
- **Cal.diy context:** No teams/orgs for multi-host assignment; single host is the booking organizer.
- **PR size guidance:** Implementation should stay small and reviewable per repo rules (<500 LOC, <10 code files)—may imply reusing existing booking/video services rather than a large new subsystem.

## Acceptance Criteria

| ID | Criterion | Verifiable by |
|----|-----------|---------------|
| AC-1 | Logged-in host sees a primary “Start meeting” (or equivalent) action in the app. | Manual |
| AC-2 | When the host is free for the next 30 minutes and Cal Video is configured, clicking the action creates an **accepted** booking with `startTime ≈ now` and `endTime ≈ now + 30 min`. | API/DB check, manual |
| AC-3 | Successful creation sets booking location to Cal Video (`integrations:daily`) and stores a resolvable video join URL (in `location` and/or booking reference / metadata consistent with existing Cal Video bookings). | API/DB check, manual |
| AC-4 | Successful creation blocks the host’s connected destination calendar for the 30-minute slot (visible in external calendar or via existing calendar-sync inspection). | Manual |
| AC-5 | Success UI shows **Join now** (opens video) and **Copy link** (copies guest join URL). | Manual |
| AC-6 | When the host has a conflicting booking or calendar busy block overlapping `[now, now + 30 min]`, the action fails with a user-visible error and **no** new booking, Daily room, or calendar event is created. | Test, manual |
| AC-7 | When Cal Video is disabled or misconfigured, the action fails with a user-visible error and **no** partial records are created. | Manual (config toggle) |
| AC-8 | When the host has no connected destination calendar, the action fails with a user-visible error directing them to connect a calendar; **no** partial records are created. | Manual, test |
| AC-9 | Unauthenticated users cannot start an instant meeting (API returns unauthorized). | API check |
| AC-10 | The created booking appears in the host’s upcoming bookings list like other accepted bookings and references the host’s hidden internal event type. | Manual, DB check |
| AC-11 | No guest-facing “connect now” page or `InstantMeetingToken` is created as part of this flow. | DB check, manual |

## Verification Expectations

### Automated tests

| Scenario | Assertion |
|----------|-----------|
| Host free + Cal Video available | Mutation succeeds; booking `ACCEPTED`; 30-minute window; location/references contain Daily URL. |
| Host busy (existing booking overlap) | Mutation fails; no new `Booking` row; no new `BookingReference` for Daily. |
| Host busy (calendar busy overlap) | Same as above (if calendar fixtures used in test harness). |
| Cal Video unavailable | Mutation fails with clear error; no booking persisted. |
| No destination calendar | Mutation fails with clear error; no booking persisted. |
| Unauthenticated call | `UNAUTHORIZED` (or equivalent). |

**Likely test locations:** new handler unit/integration tests alongside `packages/trpc/server/routers/loggedInViewer/`; service-level tests near booking conflict tests (`ensureAvailableUsers`, `checkForConflicts`).

### Regression

- `yarn type-check:ci --force`
- `yarn biome check` on touched files
- `TZ=UTC yarn test` for affected packages (booking, trpc loggedInViewer, any new service tests)
- Existing `connectAndJoin.handler.test.ts` and booking creation tests remain green

### Manual verification

1. Log in as host with Cal Video configured and an empty next 30 minutes → start meeting → join and copy link work.
2. Create a booking occupying “now” → start meeting → error, no new booking.
3. Block calendar externally for current slot → start meeting → error, no new booking.
4. Disable `daily-video` app or remove API key → start meeting → error, no new booking.
5. Disconnect destination calendar → start meeting → error prompting calendar connection, no new booking.
6. Confirm booking shows under **Bookings → Upcoming** with join affordance and hidden event type link.
7. Open copied link in incognito → guest can reach Cal Video prejoin (public room).

**Repo commands:** `yarn type-check:ci --force`, `TZ=UTC yarn test`, `yarn biome check --write .` (on changed paths).

## Decisions

| # | Topic | Decision |
|---|-------|----------|
| D-1 | Event type attachment (**OQ-2**) | **Hidden internal event type per host.** Each host gets (or reuses) a dedicated `EventType` with `hidden: true`, fixed 30-minute length, location `integrations:daily`, and a stable internal slug (e.g. `instant-meeting`). Not listed on the public booking page. Bookings created through this flow reference that event type’s `eventTypeId`. |
| D-2 | No destination calendar (**OQ-7**) | **Fail early.** If the host has no connected destination calendar, reject the action with a clear user-visible error (e.g. link/prompt to connect a calendar). Do not create a booking, Daily room, or calendar event. |
| D-3 | Orchestration (**OQ-6**) | **Path B — standard pipeline.** A focused `StartInstantMeetingService` creates the booking row, builds a `CalendarEvent`, runs conflict checks via existing availability helpers, then calls `EventManager.create` for Cal Video + calendar block + `BookingReference` persistence — matching normal Cal Video bookings. Does not invoke the full public booker/`RegularBookingService` handler; does not use `createInstantMeetingWithCalVideo`. Roll back the booking if `EventManager.create` fails. |

## Open Questions

| # | Question | Options | Notes |
|---|----------|---------|-------|
| OQ-1 | **Where should the entry-point live?** | (a) Bookings page header/toolbar (b) Global shell / Kbar (c) Dedicated lightweight modal from multiple places | Story asks for minimal UX with one primary action; bookings context is strongest fit. **Recommendation:** bookings upcoming view unless product prefers global visibility. |
| OQ-3 | **What booking title and attendee rows should be created?** | (a) Fixed title e.g. “Instant meeting” (b) Localized template with host name (c) Host as sole attendee vs no attendees | Legacy i18n keys exist (`instant_meeting_with_title`). Attendee list affects emails and booking detail UI. |
| OQ-4 | **Should confirmation emails / SMS fire?** | (a) Same as normal accepted booking (b) Suppress notifications for instant meetings | **More relevant with D-3** — Path B may trigger the same post-create notification path as regular bookings unless explicitly gated. |
| OQ-5 | **Which timezone defines “now”?** | (a) Host profile timezone (b) Browser timezone (c) UTC | Affects `startTime`/`endTime` storage and calendar block edges. **Recommendation:** host timezone to match existing booking behavior. |

### D-3 — Orchestration reference (Path B, chosen)

The implementation follows the standard Cal Video booking integration path:

1. Pre-flight: auth, destination calendar (D-2), Cal Video enabled, busy check `[now, now+30m]`.
2. Load or create hidden event type (D-1); load organizer credentials.
3. Build `CalendarEvent` (now → +30m, `integrations:daily`, organizer, destination calendar).
4. Conflict check via `ensureAvailableUsers` / `getUserAvailability` (single host).
5. Create accepted booking via `createBooking`.
6. `EventManager.create(evt)` → Daily room (`createMeeting`, tied to booking `uid`) + calendar event + references.
7. Persist references and update `location` / metadata per `RegularBookingService` post-create patterns.
8. On integration failure after booking insert: roll back booking (and clean up any created integration artifacts per existing patterns).

**Rejected alternative (Path A):** `createInstantMeetingWithCalVideo` + manual reference/calendar wiring — not used; higher partial-creation risk and drift from normal booking shape.

<details>
<summary>Path A vs Path B comparison (archived)</summary>

| Concern | Path A (room-only helper) | Path B (`EventManager.create`) ✓ |
|---------|---------------------------|----------------------------------|
| Calendar block | Manual / separate step | Built-in |
| `BookingReference` shape | Manual | Matches existing bookings |
| Atomic rollback | Fully custom | Partially existing patterns |
| Code reuse | Low | High |
| Risk of partial creation | Higher | Lower |
| Suppressing notifications | Easier to skip | Needs explicit gate (OQ-4) |

</details>

## Assumptions

- Duration is **exactly 30 minutes**, not configurable in v1.
- “Busy” means any overlap with buffered busy times used in normal booking availability (existing Cal.diy bookings + connected calendar busy), not the legacy `instantMeetingSchedule` working-hours check.
- Share link is the **Daily room URL** guests can open directly (public room with knocking/prejoin as configured today), not a Cal.diy booking detail page.
- Host is the booking organizer (`userId`); no team routing or round-robin.
- Single-host Cal.diy instance: no org gate (unlike `connectAndJoin`).
- Standard `BOOKING_CREATED` webhooks are expected with D-3 unless explicitly suppressed in the service layer.
- Success state can be inline (modal or panel) rather than a new route.
- Rate limiting may be applied but is not required by the story; existing `instantMeeting` bucket could be wired if abuse is a concern.
- Each host has (or receives on first use) one hidden internal event type for instant meetings (D-1).
- Host must have a destination calendar connected before instant meeting creation succeeds (D-2).
- Video + calendar + references are created via `EventManager.create`, not `createInstantMeetingWithCalVideo` (D-3).

## Initial Context Hints

| Area | File | Key detail |
|------|------|------------|
| Legacy team join | `packages/trpc/server/routers/loggedInViewer/connectAndJoin.handler.ts` | Org-only; consumes `InstantMeetingToken`; not the new flow |
| Legacy UI | `apps/web/modules/connect-and-join/connect-and-join-view.tsx` | Token-based; shows upgrade for non-org users |
| Unused video helper | `packages/features/conferencing/lib/videoClient.ts` | `createInstantMeetingWithCalVideo(endTime)` — no callers |
| Daily instant room | `packages/app-store/dailyvideo/lib/VideoApiAdapter.ts` | `createInstantCalVideoRoom` / `createInstantMeeting`; public room, `exp` from end time + 1h |
| Cal Video location | `packages/app-store/constants.ts` | `DailyLocationType = "integrations:daily"` |
| Daily keys | `packages/app-store/dailyvideo/lib/getDailyAppKeys.ts` | Reads `daily-video` app keys; throws if missing |
| Booking video + calendar | `packages/features/bookings/lib/EventManager.ts` | `create()` — dedicated video + `createAllCalendarEvents` (D-3 entry point) |
| Post-create reference wiring | `packages/features/bookings/lib/service/RegularBookingService.ts` | Post-`createBooking` → `EventManager.create` → reference/location persistence pattern to extract |
| Conflict check | `packages/features/bookings/lib/conflictChecker/checkForConflicts.ts` | Interval overlap on buffered busy times |
| Availability gate | `packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts` | Uses `checkForConflicts`; throws `NoAvailableUsersFound` |
| Join button | `apps/web/modules/bookings/components/JoinMeetingButton.tsx` | Join via `useJoinableLocation` |
| Copy pattern | `packages/lib/hooks/useCopy.ts` | Clipboard + copied state |
| Bookings page | `apps/web/app/(use-page-wrapper)/(main-nav)/bookings/[status]/page.tsx` | Auth + `BookingsList` shell |
| Bookings view | `apps/web/modules/bookings/views/bookings-view.tsx` | List/calendar container |
| tRPC router | `packages/trpc/server/routers/loggedInViewer/_router.tsx` | Pattern for new `authedProcedure` mutation |
| Prisma legacy | `packages/prisma/schema.prisma` | `InstantMeetingToken`, `EventType.isInstantEvent`, instant meeting fields |
| Webhook removal | `packages/prisma/migrations/20260430000000_drop_instant_meeting_webhook_trigger/migration.sql` | `INSTANT_MEETING` trigger removed |
| Rate limit bucket | `packages/lib/rateLimit.ts` | `instantMeeting` type defined, unused |
| Public instant gating | `packages/features/eventtypes/lib/getPublicEvent.ts` | `isCurrentlyAvailable` for legacy guest instant events |
| i18n (reuse) | `packages/i18n/locales/en/common.json` | `join_meeting`, `instant_meeting`, `instant_meeting_with_title` exist |
| Creation source enum | `packages/prisma/schema.prisma` | `CreationSource`: WEBAPP, API_V1, API_V2 |

---

## Spec self-check (planning readiness)

**Blocking questions:** OQ-1 (UI placement), OQ-3 (title/attendees), OQ-4 (notifications — elevated priority with D-3), OQ-5 (timezone). D-1, D-2, and D-3 are resolved.

**Scope creep risks:** Re-enabling public `isInstantEvent` flows, team tokens, instant-meeting schedule UI, pulling in all of `RegularBookingService`, or cleaning all legacy instant-meeting schema/i18n.

**Risky assumptions:** Invoking the full `RegularBookingService` handler instead of an extracted creation slice; skipping rollback when `EventManager.create` fails after booking insert.

**Acceptance criteria gaps addressed:** AC-6/AC-7/AC-8 cover atomic failure; AC-10 covers hidden event type; AC-11 excludes legacy token flow.

**Suggested spec improvements for a follow-up pass:** Once OQ-1 is decided, add a single sentence to Expected Behavior naming the surface (e.g. “Bookings → Upcoming toolbar”). Once OQ-3/OQ-4 are decided, add explicit email/attendee acceptance criteria.
