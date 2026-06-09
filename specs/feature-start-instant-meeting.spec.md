# Feature Start Instant Meeting Spec

## Business Goal

Give solo hosts a fast way to start an ad-hoc video meeting from inside Cal.diy: one action creates a real 30-minute booking, blocks their calendar, and produces a link they can share—without scheduling friction or external conferencing tools.

## User / System Problem

Hosts who need to meet immediately cannot start a meeting from the app. They must either share a normal booking link and hope a slot works, or leave Cal.diy for Zoom/Meet. The removed “instant meeting” feature targeted teams (guest rings, org member joins via token) and is unavailable in Cal.diy. There is no host-initiated equivalent.

## Current Behavior

- **Product positioning:** Cal.diy docs list Instant Meeting as unavailable (❌ vs Cal.com).
- **Removed flow:** `InstantBookingCreateService`, `/api/book/instant-event`, booker `InstantBooking` / “Connect now” UI, event-type Instant tab, and team instant-meeting pages were removed in the Cal.diy refactor. Creation logic is not in the tree.
- **Orphaned pieces:**
  - `InstantMeetingToken` model still exists and requires `teamId`; tied to team bookings.
  - `connectAndJoin` tRPC handler and `connect-and-join-view.tsx` remain but require org membership; UI copy states instant bookings are for team event types only.
  - Event type fields remain: `isInstantEvent`, `instantMeetingScheduleId`, `instantMeetingParameters`, etc.
  - `getPublicEvent.ts` still computes `showInstantEventConnectNowModal` and `isCurrentlyAvailable` for instant schedules, but no booker UI consumes this for instant creation.
- **Video:** `createInstantMeetingWithCalVideo` in `packages/features/conferencing/lib/videoClient.ts` can create a Daily room; it depends on `getDailyAppKeys()` from the global Cal Video app (`daily-video` slug).
- **Cal Video configuration:** Instance operator sets `DAILY_API_KEY` in environment; `scripts/seed-app-store.ts` registers the global app. Cal Video metadata marks `isGlobal: true` and `installed` when the env key is present. Hosts do not enter a Daily API key in settings; they may install Cal Video from the app store for normal event types.
- **Existing video routes:** Guests/hosts can join via `/video/[uid]`; booking metadata can store `videoCallUrl` (pattern used by deleted instant flow).
- **Normal bookings:** `RegularBookingService` creates bookings, calendar events, and video references through the standard pipeline; availability and busy checks live under `packages/features/availability/`.

## Expected Behavior

- A logged-in host has a single prominent action (e.g. **Start meeting**) in the authenticated app shell or bookings area.
- On success:
  - A booking is created with **accepted** status, **30-minute** duration starting at or immediately after the action time, host as organizer.
  - Location is **Cal Video**; a Daily room is created and linked on the booking (metadata / references consistent with existing video bookings).
  - The host’s connected calendar(s) show a **busy block** for that window, equivalent to a normal booking.
  - The host sees a success state with:
    - **Join now** — opens the host into the Cal Video meeting for that booking.
    - **Copy link** — one shareable URL for guests (video join URL for the booking).
- On failure:
  - If the host has any **overlap** in `[now, now + 30 minutes]` from existing Cal.diy bookings or connected calendar busy times, the action is rejected with a clear message; **no** booking, room, or calendar event is created.
  - If Cal Video / Daily is not configured on the instance, the action fails with a clear message aimed at the host (contact administrator / video unavailable); **no** partial booking.
- Only **one** instant-style meeting can be started when the host is not free; overlapping attempts are blocked by the same availability rule.
- **No** guest-facing instant event page, **no** team token, **no** `connectAndJoin` race, **no** duration picker.

## Technical Scope

In scope:

| Area | Scope |
|------|--------|
| API | New authenticated mutation (tRPC) to start an instant meeting for the current user |
| Availability | Pre-flight check for conflicts in a fixed 30-minute window from “now” (bookings + calendar busy) |
| Booking creation | Create accepted booking with Cal Video location and standard calendar write path |
| Video | Use existing `createInstantMeetingWithCalVideo` (or equivalent existing video client path) |
| Event type | Internal/hidden event type per user (or equivalent) — 30 min, Cal Video only, not promoted on public profile |
| UI | Start action + success modal/sheet (join + copy link) + error states |
| i18n | User-visible strings in `packages/i18n/locales/en/common.json` |
| Tests | Unit/integration tests for availability rejection and successful creation paths |

Out of scope for implementation details in this spec (left to planning agent): exact file placement, reuse vs fork of `RegularBookingService`, schema migrations.

Explicitly **not** in scope:

- Schema changes to `InstantMeetingToken` or team instant flows
- Public booker changes
- New dependencies
- Configurable duration or provider picker

## Non-Goals

- Restoring deleted Cal.com team instant meeting (guest “Connect now”, `InstantMeetingToken`, team push notifications, `connectAndJoin` first-host-wins).
- Guest-initiated instant meetings on public booking pages.
- User-configurable instant meeting length (fixed 30 minutes only).
- Choosing Zoom/Meet/Jitsi for instant meetings (Cal Video only).
- Dedicated instant-meeting analytics, insights, or webhook trigger type.
- Re-enabling the event-type **Instant** settings tab for hosts to configure instant event types manually.
- Email/SMS customization beyond what normal booking creation already sends (if any).

## Constraints

- Cal.diy monorepo: TypeScript, tRPC, Prisma, Next.js App Router patterns in `apps/web`.
- Use `select` in Prisma queries; `ErrorWithCode` in services, `TRPCError` in routers.
- No new npm dependencies without explicit approval.
- Cal Video requires instance-level `DAILY_API_KEY`; feature must degrade gracefully when keys are missing.
- PR size guidelines: prefer a focused change set; split UI and service layers if needed.
- Translations required for all new UI strings (English `common.json` minimum).
- Must not expose `credential.key` or other sensitive fields in API responses.
- Permission: only the authenticated host can start a meeting for themselves (no team/org instant in v1).

## Acceptance Criteria

| ID | Criterion | Verifiable by |
|----|-----------|---------------|
| AC-1 | Logged-in host with no conflicts in the next 30 minutes can start a meeting and receives join URL and copyable guest link | Manual + automated |
| AC-2 | Created booking has accepted status, 30-minute length, host as organizer, Cal Video location | API/DB test |
| AC-3 | Host connected calendar shows a busy event for the meeting window | Manual (or integration with calendar mock) |
| AC-4 | If host has an overlapping accepted/upcoming booking in the window, start is rejected and no booking/room/calendar row is created | Automated |
| AC-5 | If host has overlapping busy time from a connected calendar in the window, start is rejected and no booking/room/calendar row is created | Automated |
| AC-6 | If Daily/Cal Video is not available on the instance, start fails with a user-visible error and no partial booking | Automated + manual on env without key |
| AC-7 | Guest opening the shared video link can reach the Cal Video meeting for that booking (when meeting is active) | Manual |
| AC-8 | Unauthenticated users cannot call the start-instant-meeting API | Automated |
| AC-9 | No UI or API path creates `InstantMeetingToken` or uses `connectAndJoin` for this feature | Code review |

## Verification Expectations

### Automated tests

| Scenario | Expected assertion |
|----------|-------------------|
| Host free, Daily configured | Mutation succeeds; booking exists with correct times and video reference |
| Host has booking overlap in window | Mutation errors; booking count unchanged |
| Host has calendar busy overlap in window | Mutation errors; booking count unchanged |
| Unauthenticated call | Unauthorized / forbidden |
| Daily keys missing | Error response; no booking persisted |

Likely areas: new handler/service test file; may extend booking or availability test patterns under `packages/features/bookings` or `packages/trpc`.

### Regression

- Existing booking creation and video booking tests continue to pass.
- `connectAndJoin.handler.test.ts` unchanged in behavior (orphan path unaffected).
- Run `yarn type-check:ci --force` and relevant unit tests for touched packages before merge.

### Manual verification

- [ ] Start meeting when calendar is clear → join works, link shareable to second browser/guest.
- [ ] Start meeting when a calendar event overlaps → blocked with clear message.
- [ ] Start meeting when an upcoming Cal.diy booking overlaps → blocked.
- [ ] Instance without `DAILY_API_KEY` → friendly failure.
- [ ] Calendar provider (e.g. Google) shows new event for the 30-minute block.

## Open Questions

| # | Question | Options | Recommendation |
|---|----------|---------|----------------|
| OQ-1 | Exact share URL: `/video/{uid}` only, or also offer `/booking/{uid}`? | Video only vs both | **Video URL only** for v1 (simplest); label copy clearly “Meeting link” |
| OQ-2 | Start time: literal `now` vs round up to next minute? | Truncate seconds vs ceil minute | **Ceil to next full minute** if sub-minute alignment matters for calendar providers (planning agent to confirm against existing booking rounding) |
| OQ-3 | Where to place the **Start meeting** entry point? | Shell nav, bookings header, dashboard | **Bookings area + optional shell shortcut** — planning agent to pick one primary surface for v1 |
| OQ-4 | Hidden event type: auto-create on first use vs one-time migration/seed per user? | Lazy create vs onboarding | **Lazy create** on first instant start to avoid migration |
| OQ-5 | Send standard booking confirmation emails on instant create? | Yes / no / host-only | **Defer** — default to whatever normal booking creation emits unless product says otherwise; note in plan |

## Assumptions

- Host timezone for the 30-minute window uses the user’s profile timezone (same as other booking flows).
- “Busy” includes accepted and pending bookings that block the slot, consistent with existing availability semantics (planning agent to align with `getUserAvailability` / busy aggregation).
- Instant meetings use the same Cal Video room lifecycle as other Daily bookings (`/video/[uid]`).
- Rate limiting similar to the removed `instantMeeting` rate limit type may be desirable but is not required for v1 unless planning finds an existing hook.
- Generic booking webhooks may fire if the implementation reuses standard booking creation hooks; no new instant-specific webhook trigger is assumed.
- Host does not need a separate Cal Video app-store install if the instance global Daily key is present (creation uses server keys).

## Initial Context Hints

| Area | File | Key detail |
|------|------|------------|
| Removed instant booking service | Git history `ab21c7f805` | Deleted `InstantBookingCreateService.ts`, `instant-event.ts` API — reference for metadata/video patterns, not for team/token logic |
| Instant video room | `packages/features/conferencing/lib/videoClient.ts` | `createInstantMeetingWithCalVideo(endTime)` |
| Daily keys | `packages/app-store/dailyvideo/lib/getDailyAppKeys.ts` | Reads global app keys from `daily-video` slug |
| Cal Video app metadata | `packages/app-store/dailyvideo/_metadata.ts` | `isGlobal: true`, `installed: !!DAILY_API_KEY` |
| Orphan team join | `packages/trpc/server/routers/loggedInViewer/connectAndJoin.handler.ts` | Org-only; not used for this feature |
| Orphan UI | `apps/web/modules/connect-and-join/connect-and-join-view.tsx` | Team/uprade copy; unrelated to host-initiated flow |
| Video join page | `apps/web/app/(use-page-wrapper)/video/[uid]/page.tsx` | Guest/host join surface |
| Availability | `packages/features/availability/lib/getUserAvailability.ts` | Busy times and schedule logic |
| Booking creation | `packages/features/bookings/lib/service/RegularBookingService.ts` | Standard booking pipeline reference |
| Schema (unused token) | `packages/prisma/schema.prisma` | `InstantMeetingToken` requires `teamId` — avoid for v1 |
| Event type instant fields | `packages/prisma/schema.prisma` | `isInstantEvent`, `instantMeetingScheduleId` — not required for host-initiated v1 |
| i18n (legacy instant) | `packages/i18n/locales/en/common.json` | Keys like `instant_tab_title`, `connect_now` exist for old flow; new keys needed for “Start meeting” |
| Docs matrix | `apps/docs/content/index.mdx` | Instant Meeting listed ❌ for Cal.diy — update docs when feature ships (outside this spec’s implementation) |
