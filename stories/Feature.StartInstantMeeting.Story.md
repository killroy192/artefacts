# Feature: Start Instant Meeting (Host-Initiated)

## User story

As a **logged-in host**, I want to **start a 30-minute meeting immediately** from the Cal.diy app so that I get a **shareable video link** I can send to anyone, without going through the public booking flow.

## Problem

Hosts who need an ad-hoc call today must create a normal booking (pick a slot, share a booking page) or use an external tool. Cal.diy previously had a **team/org “instant meeting”** flow (guest clicks “Connect now”, team members race to join via token). That was removed in the Cal.diy refactor and does not fit the solo/self-hosted product. There is no host-initiated “start meeting now” path.

## Requirements

| Area | Decision |
|------|----------|
| Initiator | Host only (logged-in); no public guest “connect now” page |
| Duration | Fixed **30 minutes** |
| Calendar | **Block** host calendar for the slot (same as a normal booking) |
| Concurrency | **Not allowed** — cannot start if the host is busy **now → now + 30 min** (existing bookings or calendar conflicts) |
| Video | **Cal Video** (`integrations:daily`) via instance Daily configuration |
| Share link | **Simplest** — one copyable join URL for guests (video meeting link) |
| UX | Minimal — one primary action, success state with join + copy |

## Out of scope

- Restoring Cal.com team instant meeting (`InstantMeetingToken`, `connectAndJoin`, push to team)
- Guest-initiated instant booking on public event types
- Configurable meeting duration
- Custom video provider selection per instant meeting
- Instant meeting webhooks as a dedicated trigger (generic booking webhooks may apply if existing booking creation fires them)

## Success looks like

1. Host clicks **Start meeting** when free for the next 30 minutes.
2. System creates an accepted booking, Cal Video room, and calendar block.
3. Host sees **Join now** and **Copy link** to share with attendees.
4. If busy or video unavailable, host sees a clear error and nothing is partially created.
