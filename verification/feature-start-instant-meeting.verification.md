# Verification Plan: Start Instant Meeting

**Spec:** `artefacts/specs/feature-start-instant-meeting.spec.md`
**Plan:** `artefacts/plans/feature-start-instant-meeting.plan.md`
**Context map:** `artefacts/context-maps/feature-start-instant-meeting.contextmap.md`

---

## 1. Acceptance Criteria to Verification Method Mapping

| AC | Criterion | Verification method | Automated? | Existing asset to reuse |
|----|-----------|--------------------:|:----------:|-------------------------|
| AC-1 | Host with no conflicts gets join URL + share URL | Unit test: happy-path service test asserts returned `{ joinUrl, shareUrl, bookingUid }` match expected URL patterns | Yes | `bookingScenario` mock patterns from `fresh-booking.test.ts` |
| AC-2 | Booking shape: ACCEPTED, 30 min, Cal Video, organizer | Unit test: assert `prisma.booking.create` was called with `status: "ACCEPTED"`, 30-min interval, `location: "integrations:daily"`, correct `userId` | Yes | `createBookingScenario` field assertions from `handleNewBooking/test/` |
| AC-3 | Calendar busy block created | Mock assertion: verify `CalendarManager.createEvent` invoked with matching start/end times | Partial | `mockCalendar` pattern from `bookingScenario`; manual verification needed for real provider |
| AC-4 | Booking overlap → rejected, no side effects | Unit test: configure availability mock to return busy entry, assert `CONFLICT` error + no `prisma.booking.create` call | Yes | `checkForConflicts.test.ts` overlap scenarios |
| AC-5 | Calendar busy overlap → rejected, no side effects | Unit test: mock `getUserAvailability` to return external calendar busy, assert `CONFLICT` | Yes | `getUserAvailabilityIncludingBusyTimesFromLimits.test.ts` mock patterns |
| AC-6 | Daily keys missing → clean failure, no partial booking | Unit test: mock `getDailyAppKeys` to throw, assert `VIDEO_UNAVAILABLE` before any DB write | Yes | `mockVideoAppToCrashOnCreateMeeting` pattern |
| AC-7 | Guest opens shared link → reaches video room | Manual test with running Daily room; share URL opens `/video/{uid}` page | No | `get-cal-video-reference.test.ts` for reference selection logic |
| AC-8 | Unauthenticated user → 401 | Unit test: invoke handler without session ctx, assert `UNAUTHORIZED` | Yes | `connectAndJoin.handler.test.ts` auth setup pattern |
| AC-9 | No `InstantMeetingToken` / `connectAndJoin` in new code | Automated grep across all new files | Yes | N/A — grep command |

---

## 2. Existing Tests and Checks to Reuse

| Existing asset | Location | How it helps |
|----------------|----------|--------------|
| `checkForConflicts.test.ts` | `packages/features/bookings/lib/conflictChecker/` | Proves `checkForConflicts` handles empty arrays, boundary overlaps, contained intervals — do NOT rewrite this; CALL it from service |
| `bookingScenario` helpers | `@calcom/testing/lib/bookingScenario/bookingScenario` | `mockSuccessfulVideoMeetingCreation`, `mockVideoAppToCrashOnCreateMeeting`, credential helpers for video mock setup |
| `connectAndJoin.handler.test.ts` | `packages/trpc/server/routers/loggedInViewer/` | MUST remain unchanged (AC-9 regression); demonstrates Prisma mock pattern (`vi.mock("@calcom/prisma")`) for handler tests |
| `getUserAvailabilityIncludingBusyTimesFromLimits.test.ts` | `packages/features/availability/lib/` | Shows how to mock `UserAvailabilityService` dependencies (`vi.mock` BusyTimes DI, calendar, etc.) |
| `get-cal-video-reference.test.ts` | `packages/features/` | Verifies Daily video reference selection from booking references |
| `unlinkConnectedAccount.handler.test.ts` | `packages/trpc/server/routers/loggedInViewer/` | Pattern for simple `authedProcedure` handler test with Prisma mock |
| Biome + TypeCheck CI | `.github/workflows/lint.yml`, `check-types.yml` | Existing CI pipelines validate type safety and formatting without extra setup |

**Rule: Do not duplicate `checkForConflicts` logic in new tests.** The service test should mock `checkForConflicts` return values (conflict = true/false) or mock `getUserAvailability` response and let the real `checkForConflicts` run.

---

## 3. Smallest New Failing Tests (Coverage Gaps)

The codebase has NO test for:
- Instant meeting service orchestration (availability check + video create + booking create)
- `createInstantMeetingWithCalVideo` being called correctly
- Lazy event type creation/reuse logic

**Proposed new test file:** `packages/features/instant-meeting/service/StartInstantMeetingService.test.ts`

### Test 1: Happy path (smallest proving test)

```typescript
describe("StartInstantMeetingService", () => {
  it("creates booking with video reference when host is free and Daily is configured", async () => {
    // Arrange: mock getDailyAppKeys (resolve), getUserAvailability (empty busy), 
    //          createInstantMeetingWithCalVideo (return { url, id, password })
    // Act: service.startMeeting(userId, timezone)
    // Assert: prisma.booking.create called with ACCEPTED + 30min + "integrations:daily"
    //         prisma.bookingReference.create called with type "daily_video"
    //         returns { joinUrl, shareUrl, bookingUid }
  });
});
```

### Test 2: Conflict rejection (smallest negative test)

```typescript
it("rejects with CONFLICT when host has overlapping booking and creates nothing", async () => {
  // Arrange: mock getUserAvailability to return busy entry overlapping [now, now+30min]
  // Act + Assert: expect(service.startMeeting(...)).rejects.toMatchObject({ code: "CONFLICT" })
  //              prisma.booking.create NOT called
  //              createInstantMeetingWithCalVideo NOT called
});
```

### Test 3: Video unavailable (earliest-exit failure)

```typescript
it("rejects with VIDEO_UNAVAILABLE when Daily keys are missing and creates nothing", async () => {
  // Arrange: mock getDailyAppKeys to throw
  // Act + Assert: expect(service.startMeeting(...)).rejects.toMatchObject({ code: "VIDEO_UNAVAILABLE" })
  //              getUserAvailability NOT called (fail fast)
  //              prisma.booking.create NOT called
});
```

### Test 4: Lazy event type (create then reuse)

```typescript
it("creates hidden event type on first call and reuses it on second", async () => {
  // First call: prisma.eventType.findFirst returns null → prisma.eventType.create called
  // Second call: prisma.eventType.findFirst returns existing → prisma.eventType.create NOT called
});
```

### Test 5: Auth required

```typescript
it("rejects unauthenticated calls with UNAUTHORIZED", async () => {
  // Call handler/mutation without ctx.user → TRPCError code UNAUTHORIZED
});
```

---

## 4. Repository Commands (Typecheck, Lint, Build, Smoke)

| Purpose | Command | When to run | Gate served |
|---------|---------|-------------|-------------|
| **Type check** | `yarn type-check:ci --force` | After every file change, before commit | G1, G4, G5, G6 |
| **Lint + format** | `yarn biome check --write .` | Before commit | G5 |
| **Unit tests (all)** | `TZ=UTC yarn test` | Before push | G1, G3 |
| **Unit tests (scoped)** | `TZ=UTC yarn test packages/features/instant-meeting/` | During development | G3 |
| **Build** | `yarn build` | CI only (slow); optional locally | G4 |
| **Smoke: regression** | `TZ=UTC yarn test packages/trpc/server/routers/loggedInViewer/connectAndJoin.handler.test.ts` | After router modification | G1 (AC-9), G2 |
| **Scope audit** | `git diff --name-only \| sort` | Before PR | G2, G6 |
| **Security grep** | `rg "credential\.key\|encryptedKey\|as any\|InstantMeetingToken\|connectAndJoin" <new-files>` | Before PR | G4, G2 |
| **Import style** | `rg "from ['\"]@calcom/(ui\|features\|lib)['\"]$" <new-files>` | Before PR | G5 |
| **Diff size** | `git diff --stat` | Before PR | G2, G6 |

### CI Pipeline (automatic on PR)

From `.github/workflows/all-checks.yml`:
1. `yarn type-check:ci` (check-types.yml)
2. `yarn lint` (lint.yml)
3. `TZ=UTC yarn test -- --no-isolate` (unit-tests.yml)
4. Timezone pass: `TZ=America/Los_Angeles VITEST_MODE=timezone yarn test -- --no-isolate`
5. Integration tests: `VITEST_MODE=integration yarn test -- --no-isolate`

---

## 5. Acceptance Criteria Not Directly Verifiable Yet

| AC | Gap | Reason | Workaround |
|----|-----|--------|------------|
| AC-3 | Calendar busy block cannot be verified in automated test without real calendar credential | External API (Google Calendar / Office 365) not available in unit test environment | Mock assertion proves `createEvent` was called with correct params; manual test with real credential documents actual behavior |
| AC-7 | Guest video link cannot be tested without running Daily room | Requires network access to Daily.co API and an active room | Manual verification with screenshots; trust that `/video/[uid]` page + `BookingReference` with `meetingUrl` works (covered by existing `get-cal-video-reference.test.ts`) |
| AC-9 (partial) | Code review judgment call | Grep catches explicit references but cannot detect semantic equivalents (e.g., reimplementing token logic without naming it `InstantMeetingToken`) | Pair grep with human code review of the service file |

**All other ACs (1, 2, 4, 5, 6, 8) are fully verifiable via automated unit tests.**

---

## 6. Action List: How Each Gate Will Be Passed

### G1: Spec Compliance

| Action | Proves | Tool/Command |
|--------|--------|--------------|
| Write 5 unit tests (Section 3 above) | AC-1, AC-2, AC-4, AC-5, AC-6 | `TZ=UTC yarn test packages/features/instant-meeting/` |
| Add auth-rejection test | AC-8 | Same test file, unauthenticated ctx |
| Run grep on new files | AC-9 | `rg "InstantMeetingToken\|connectAndJoin" <files>` |
| Mock assertion for `CalendarManager.createEvent` | AC-3 | Within happy-path test |
| Manual test with screenshots | AC-7 | Start meeting → open share URL in incognito |
| Fill AC traceability table in PR description | All | PR body |

### G2: Scope Control

| Action | Proves | Tool/Command |
|--------|--------|--------------|
| Confirm no schema.prisma changes | No schema drift | `git diff -- packages/prisma/schema.prisma` |
| Confirm RegularBookingService untouched | No coupling to complex pipeline | `git diff -- packages/features/bookings/lib/service/RegularBookingService.ts` |
| Confirm connectAndJoin untouched | AC-9, no legacy interference | `git diff -- **/connectAndJoin*` |
| Confirm onboarding untouched | No side effects on user setup | `git diff -- apps/web/modules/onboarding/` |
| Confirm no new dependencies | No unapproved packages | `git diff -- **/package.json yarn.lock` filtered for `dependencies` |
| Verify file list matches plan | Only planned files touched | `git diff --name-only` vs plan table |
| Verify line count under 500 | PR size compliance | `git diff --stat` |

### G3: Test Quality

| Action | Proves | Tool/Command |
|--------|--------|--------------|
| Each test asserts a specific error CODE (not message string) | Typed errors, not brittle matching | Code review of test file |
| Each failure-path test has negative assertion (side effect did NOT happen) | Proves atomicity | Check for `expect(mock).not.toHaveBeenCalled()` |
| Test names describe behavior in plain English | Readability | Code review |
| Tests mock at boundary (deps), not internal implementation | Refactor-safe | No mocking of private methods or internal utility calls |
| Lazy event type test covers BOTH create and reuse paths | Full branch coverage for critical logic | Test 4 above |
| Run `connectAndJoin.handler.test.ts` to confirm unchanged behavior | Regression proof | `TZ=UTC yarn test packages/trpc/server/routers/loggedInViewer/connectAndJoin.handler.test.ts` |

### G4: Risk

| Action | Proves | Tool/Command |
|--------|--------|--------------|
| Grep for credential exposure | No secrets in responses | `rg "credential\.key\|\.key\|encryptedKey" <new-files>` |
| Grep for `as any` | Type safety | `rg "as any" <new-files>` |
| Verify `authedProcedure` used (not `publicProcedure`) | Auth enforced | Read `_router.tsx` diff |
| Count DB calls in service (max 4-5) | No N+1 | Code review of service method |
| Verify video room created BEFORE booking (ordering) | Orphan direction is safe (room auto-expires) | Code review of service step order |
| Verify no `upsert` for event type (use `findFirst` + `create`) | No accidental overwrite of user data | Code review |
| Confirm output schema has only safe fields | No data leakage | Read `startInstantMeeting.schema.ts` |
| Run full type check | No unsafe casts | `yarn type-check:ci --force` |

### G5: Maintainability

| Action | Proves | Tool/Command |
|--------|--------|--------------|
| Service has exactly ONE public method (`startMeeting`) | Single responsibility | Code review |
| Handler is thin (extract ctx, call service, map errors) | Separation of concerns | Code review — handler < 30 lines |
| No barrel imports | Direct path imports per CLAUDE.md | `rg "from ['\"]@calcom/(ui\|features\|lib)['\"]$" <new-files>` |
| `ErrorWithCode` used in service, `TRPCError` only in handler | Error convention | Code review |
| Zod schemas have explicit types (no `z.any()`) | Type safety | Read schema file |
| i18n keys are snake_case with `instant_meeting_` prefix | Naming consistency | Read `common.json` diff |
| No comments that restate what code does | Clean code | Code review |
| File names match conventions | Discoverability | `startInstantMeeting.schema.ts`, `.handler.ts`, `StartInstantMeetingService.ts` |
| Run Biome | Format + lint compliance | `yarn biome check --write .` |

### G6: Evidence

| Action | Proves | Artifact location |
|--------|--------|-------------------|
| Capture test run output (all pass) | Tests work | PR comment or CI link |
| Capture type-check pass | Type safety | CI status badge |
| Run scope audit commands, paste output | Scope clean | PR description |
| Run security grep, paste output | No credential leaks | PR comment |
| Paste `git diff --stat` | Size limit met | PR description footer |
| Fill AC traceability table | Complete coverage map | PR body |
| Screenshots of manual test (PR 2) | UI works end-to-end | PR comment |

---

## Scoring Rubric

After implementation, each gate receives a score:

| Score | Meaning |
|-------|---------|
| 0 | Gate failed — blocking issue found |
| 1 | Gate partially passed — non-blocking concerns or gaps documented |
| 2 | Gate fully passed — all checks clean with evidence |

### Scoring Table (to be filled post-implementation)

| # | Gate | Actions required | Score (0-2) | Notes |
|---|------|-----------------|:-----------:|-------|
| G1 | Spec compliance | 5 automated tests pass + grep + manual test log | ___ | All 9 ACs traced |
| G2 | Scope control | 7 scope audit commands return clean | ___ | Only planned files in diff |
| G3 | Test quality | 6 quality criteria met per test | ___ | Negative assertions present |
| G4 | Risk | 8 risk checks pass (grep + review + type-check) | ___ | Zero security/perf issues |
| G5 | Maintainability | 9 code quality criteria met + Biome passes | ___ | New dev can read in 10 min |
| G6 | Evidence | 7 artifacts attached to PR | ___ | Reviewer can verify without reading code |

**Merge threshold:** All gates >= 1. Total >= 9/12.
**Ideal:** All gates = 2. Total = 12/12.

---

## Gate Failure Responses

| Gate | If score = 0 | Fix action |
|------|--------------|------------|
| G1 | Missing AC coverage | Write the missing test or document why it's not automatable |
| G2 | Unplanned file modified | Remove the change or update the plan with justification |
| G3 | Tests don't prove behavior | Rewrite tests with proper assertions (not just "runs without error") |
| G4 | Security/perf concern found | Fix immediately — no merge with G4 = 0 |
| G5 | Code is hard to follow | Refactor: extract, rename, or split until clear |
| G6 | Missing evidence | Produce the artifact before requesting re-review |

---

## QA Verification Steps

### Prerequisites

- Running Cal.diy instance with `DAILY_API_KEY` configured
- At least one connected calendar (Google or Outlook) for the test user
- A second browser/incognito window for guest verification
- Access to remove `DAILY_API_KEY` from env for negative testing

### QA-1: Happy Path — Start Meeting When Free

| Step | Action | Expected result |
|------|--------|-----------------|
| 1 | Log in as host with no upcoming bookings in the next 30 minutes | Bookings page loads; toolbar visible |
| 2 | Click "Start meeting" button in the bookings toolbar | Button shows loading state |
| 3 | Wait for response | Success sheet/modal appears with "Join now" and "Copy meeting link" actions |
| 4 | Verify displayed time range | Shows current time (rounded to minute) through +30 minutes |
| 5 | Click "Join now" | New tab opens with Cal Video (Daily) meeting room; host enters meeting |
| 6 | Click "Copy meeting link" | Toast confirms "Meeting link copied"; clipboard contains URL matching `/video/{uid}` pattern |
| 7 | Check bookings list | New booking appears with "Instant Meeting" title, ACCEPTED status, 30-min duration |
| 8 | Check connected calendar (Google/Outlook) | Busy event appears for the 30-minute window with "Instant Meeting" title |

**Pass:** All 8 steps succeed. **Covers:** AC-1, AC-2, AC-3

### QA-2: Guest Joins via Shared Link

| Step | Action | Expected result |
|------|--------|-----------------|
| 1 | Complete QA-1 steps 1-6 | Host has meeting running, link copied |
| 2 | Open incognito/second browser | Clean session, no auth |
| 3 | Paste the copied meeting link into address bar | `/video/{uid}` page loads |
| 4 | Enter name (if Daily prejoin screen shown) and join | Guest enters the same Daily room as the host |
| 5 | Verify host sees guest in the meeting | Both participants visible in the room |

**Pass:** Guest reaches the video room without authentication. **Covers:** AC-7

### QA-3: Rejection — Overlapping Cal.diy Booking

| Step | Action | Expected result |
|------|--------|-----------------|
| 1 | Create a normal booking for the host that starts within the next 30 minutes | Booking confirmed and visible in list |
| 2 | Click "Start meeting" | Button shows loading briefly |
| 3 | Observe result | Error toast appears: "You have a conflicting event in the next 30 minutes" (or equivalent) |
| 4 | Verify no new booking created | Bookings list has no new "Instant Meeting" entry |
| 5 | Check connected calendar | No new event added |
| 6 | Check Daily dashboard (optional) | No new room created |

**Pass:** Clear error message; no side effects. **Covers:** AC-4

### QA-4: Rejection — Overlapping Calendar Busy Time

| Step | Action | Expected result |
|------|--------|-----------------|
| 1 | Create an event directly in connected calendar (Google/Outlook) within the next 30 minutes (NOT through Cal.diy) | External event visible in calendar provider |
| 2 | Wait 1-2 minutes for calendar sync (or trigger manual refresh if available) | Cal.diy recognizes external busy time |
| 3 | Click "Start meeting" | Button shows loading briefly |
| 4 | Observe result | Error toast with conflict message |
| 5 | Verify no new booking or Daily room created | No "Instant Meeting" in bookings list |

**Pass:** External calendar busy blocks instant meeting creation. **Covers:** AC-5

### QA-5: Rejection — Video Not Configured

| Step | Action | Expected result |
|------|--------|-----------------|
| 1 | Remove `DAILY_API_KEY` from environment (or use a test instance without it) | App running without video config |
| 2 | Log in as host | Bookings page loads normally |
| 3 | Click "Start meeting" | Button shows loading briefly |
| 4 | Observe result | Error toast: "Video conferencing is not configured on this instance" (or equivalent) |
| 5 | Verify no booking created | No "Instant Meeting" in bookings list |
| 6 | Restore `DAILY_API_KEY` and verify feature works again | QA-1 happy path succeeds |

**Pass:** Graceful degradation with clear message; no partial state. **Covers:** AC-6

### QA-6: Authentication — Unauthenticated Access

| Step | Action | Expected result |
|------|--------|-----------------|
| 1 | Open browser with no active session (incognito) | Not logged in |
| 2 | Attempt to call the mutation endpoint directly (via browser devtools/curl) | API returns 401 Unauthorized |
| 3 | Verify no booking created | No new bookings in the system for any user |

**Pass:** Unauthenticated users cannot trigger meeting creation. **Covers:** AC-8

### QA-7: Consecutive Meeting Attempts

| Step | Action | Expected result |
|------|--------|-----------------|
| 1 | Host has no conflicts; click "Start meeting" | Meeting created successfully (QA-1) |
| 2 | Immediately click "Start meeting" again (within 30-min window of first meeting) | Error: conflict with the just-created meeting |
| 3 | Wait until the first meeting's 30-min window has passed | Slot is now free |
| 4 | Click "Start meeting" again | New meeting created successfully |

**Pass:** Availability check prevents double-booking; system recovers after time passes. **Covers:** AC-4 (self-overlap)

### QA-8: UI State Consistency

| Step | Action | Expected result |
|------|--------|-----------------|
| 1 | Click "Start meeting" and receive success sheet | Sheet is open |
| 2 | Close/dismiss the success sheet | Sheet closes; bookings list is visible |
| 3 | Refresh the page | New "Instant Meeting" booking persists in the list |
| 4 | Navigate away and return to bookings | Booking still present; no orphan state |

**Pass:** UI state is consistent with DB state after all interactions. **Covers:** AC-1, AC-2

### QA-9: Legacy Instant Meeting Isolation

| Step | Action | Expected result |
|------|--------|-----------------|
| 1 | Navigate to any page that previously referenced instant meetings (if accessible) | No broken UI or errors |
| 2 | Verify "Start meeting" button does NOT appear on public booking pages | Button only visible in authenticated bookings area |
| 3 | Verify no `InstantMeetingToken` rows created in DB after using the feature | `SELECT * FROM "InstantMeetingToken"` returns no new rows |
| 4 | Verify `connectAndJoin` endpoint still works as before (returns org-required error) | Calling the legacy mutation returns the same error as pre-feature |

**Pass:** New feature is fully isolated from legacy team instant meeting code. **Covers:** AC-9

### QA Summary Matrix

| QA Test | ACs Covered | Automated equivalent | Priority |
|---------|-------------|---------------------|----------|
| QA-1 | AC-1, AC-2, AC-3 | Unit tests 1-2 + mock assertion | P0 (must pass) |
| QA-2 | AC-7 | None (manual only) | P0 |
| QA-3 | AC-4 | Unit test 2 | P0 |
| QA-4 | AC-5 | Unit test 2 (calendar variant) | P1 (depends on calendar sync timing) |
| QA-5 | AC-6 | Unit test 3 | P0 |
| QA-6 | AC-8 | Unit test 5 | P1 |
| QA-7 | AC-4 (self) | Not directly covered by unit tests | P1 |
| QA-8 | AC-1, AC-2 | None (UI persistence) | P2 |
| QA-9 | AC-9 | Grep + unit regression | P1 |

### QA Sign-Off Checklist

- [ ] QA-1: Happy path completed with evidence (screenshot of success sheet + booking in list)
- [ ] QA-2: Guest join verified (screenshot of both participants in room)
- [ ] QA-3: Booking overlap rejection verified (screenshot of error toast)
- [ ] QA-4: Calendar busy rejection verified (screenshot of error toast + external event)
- [ ] QA-5: No-video-config rejection verified (screenshot of error message)
- [ ] QA-6: Auth rejection confirmed (401 response captured)
- [ ] QA-7: Consecutive attempt blocked then succeeds after window passes
- [ ] QA-8: UI state persists after refresh and navigation
- [ ] QA-9: No legacy artifacts created; isolation confirmed

**QA Pass Criteria:** All P0 tests pass. No more than 1 P1 test has documented, non-blocking issue. P2 tests are informational.

---

## Execution Timeline

```
PR 1 (Backend):
  1. Implement service + handler + schema + router
  2. Write 5 unit tests (Section 3)
  3. Run: yarn type-check:ci --force
  4. Run: TZ=UTC yarn test packages/features/instant-meeting/
  5. Run: TZ=UTC yarn test packages/trpc/server/routers/loggedInViewer/connectAndJoin.handler.test.ts
  6. Run: scope audit commands (Section 4)
  7. Run: security greps (Section 4)
  8. Run: yarn biome check --write .
  9. Fill scoring table → all G1-G6 checks
  10. Compose PR description with evidence

PR 2 (Frontend):
  1. Implement button + sheet + toolbar integration + i18n
  2. Run: yarn type-check:ci --force
  3. Run: yarn biome check --write .
  4. Manual test: 4 scenarios (free, busy, no Daily, guest link)
  5. Capture screenshots
  6. Run: scope audit (no unexpected files)
  7. Fill scoring table
  8. Compose PR description with evidence
```
