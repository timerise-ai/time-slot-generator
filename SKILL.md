---
name: time-slot-generator
description: >
  Build a bookable time-slot generator driven by strategies: daily ranges,
  Nth-weekday-of-month rules, one-off slots and whole-day slots, authored in
  local wall-clock time and stored as UTC instants, correct across DST, far-from-UTC
  zones and any server TZ, with availability (seats, bookings, blackouts, host
  busy time, shared resources, working hours) on Firestore or Supabase. Use when:
  (1) an app needs appointment, class, rental or reservation slots generated from
  rules rather than stored one by one, (2) existing slot code shows wrong days
  around DST, wrong weekdays far from UTC, missing slots in a week view, or times
  that change with the server's TZ, (3) strategies must be validated, pinned to a
  zone, or migrated from wall-clock-as-UTC storage, (4) the user mentions: time
  slots, slot generator, availability, booking slots, opening hours, slot
  strategy, RANGE, PERIODICITY, DAY_RANGE, ONCE, slotInterval, slotDuration,
  slotsLimit, first Monday of the month, last Friday, overnight slots, lead time,
  booking window, timeOffset, IANA time zone, DST, fromZonedTime, storageVersion,
  slot ID, overbooking, capacity check. Carries a dependency-free engine on Intl
  with a 23-test suite run under four process time zones, slot IDs compatible with
  existing data, and a capacity re-check inside the booking transaction for both
  backends. TypeScript on any Node server (Next.js route handlers, Express,
  GraphQL); the data layer is a seam. Not a calendar UI, not RRULE/iCalendar
  recurrence, not payment or booking-flow screens.
---

# Time-Slot Generator

Slots are never stored. Strategies are stored, and slots are regenerated from them on every read, then
looked up again by a deterministic ID when someone books. Everything in this skill follows from that: the
generator must be a pure function of (strategies, now, viewport) whose output does not depend on the
viewport, the server's time zone, or the day of the year — or a slot that was on screen is gone when the
guest presses "book".

## When to use

- Generating bookable slots from rules: "Mon–Fri 09:00–17:00 every 30 min", "first and last Friday of
  each month 18:00–20:00", "whole days, weekends only", "this one Saturday 10:00–14:00".
- Showing availability: capacity per slot, minus bookings, blackouts, a host's other commitments,
  shared rooms or equipment, and outside working hours.
- Fixing slot code with time-zone bugs, or moving one off wall-clock-as-UTC storage.
- Porting a slot engine between Firestore and Supabase/Postgres.

## When NOT to use

- **The booking screens or kiosk flow** → `booking-kiosk` (it consumes the slots this skill produces).
- **Calendar recurrence (RRULE, iCalendar, Google/Outlook event series)** → use `rrule` or the provider's
  API. Strategies describe *bookable capacity*, not events.
- **Syncing external calendars** → host integration. This skill only consumes their busy time as `BLOCK`.
- **Payments, invoices, reminders** → separate modules.

## Architecture

```
strategies (DB) ──normalize──▶ SlotStrategy[] ──generateSlots(now, viewport, limits)──▶ GeneratedSlot[]
                                                                                          │
bookings, busy blocks, host bookings, resources, working hours ──SlotStore──▶ buildAvailability
                                                                                          │
                        listing: toSlotOutput (UTC + local offset)  ◀──────────────────── ┤
                        booking: resolveSlotsForBooking(ids) → transaction capacity check ◀┘
```

## Critical facts

1. **Store instants in UTC, author in wall clock, pin the zone.** `start.time: "09:00"` means 09:00 in
   the strategy's `timeZone`, fixed at create time. A later change to the project's zone must not move
   existing slots.
2. **Iterate local calendar dates, never `+24h`.** A DST day is 23 or 25 hours. Adding 24h to 23:30 the
   evening before a spring-forward day lands two days later and that day silently disappears.
3. **Emit chronologically before you stop early.** Generators stop at the first slot past a cutoff. If
   candidates are produced weekday by weekday (all Sundays, then all Mondays…), the first Sunday past the
   cutoff ends generation and a week view shows one day.
4. **Limits count from the strategy start, never from the viewport.** Otherwise "the first 10 slots"
   depends on which week the client asks for, and a slot visible in one view is unbookable.
5. **The server's TZ must not matter.** `new Date('2026-03-29T09:00')` is parsed in the process zone, and
   `toZonedTime(d).getUTCDay()` is only right when the process runs in UTC. Use the Intl helpers.
6. **One generation per booking, then lookups.** Regenerating per requested slot multiplies the work by
   the number of slots; an 18-slot booking did that and ran past a 100 s edge timeout.
7. **Generation is stale by the time you write.** Re-count seats inside the booking transaction.

## Hard rules

> **Never change the slot ID input** — `md5("<utcStartISO>;<duration>;<strategyId>")`. Every ID already in
> a client or on a booking stops resolving.

> **Never infer "overnight" from end < start.** 17:00–09:00 is usually a typo; require `overnight: true`.

> **Never generate without a horizon.** Unbounded strategies run to 2038 unless `maxHorizonDays` caps them.

> **Never filter availability only in the listing.** Build the booking path from the same map, or anyone
> holding an ID can book what the listing hides.

> **Never treat a booking of this service by a host as host busy time.** It already subtracts seats;
> blocking on it as well hides a slot that still has room.

## Adaptation Contract

| Seam | The skill ships | The host supplies |
|---|---|---|
| Domain entities | Service, Strategy, Booking, Host, Location, Resource, BusyBlock | Its names (Event type, Staff, Room…) |
| Tenant scope | `projectId`, derived from the stored service | org / workspace / account |
| Auth guard | None inside the engine; strategy writes need staff | Its guard before `saveStrategy` |
| Data access | `SlotStore` + Firestore and Supabase implementations | Its ORM or SDK |
| Clock | `now` injected everywhere | `new Date()` at the edge, fixed in tests |
| Slot IDs | md5 via `node:crypto` | A pure md5 on edge runtimes |
| Validation | `validateStrategy` with `code: detail` errors | zod/valibot schema mapping to it |
| Background work | None required; optional one-off normalization | Its cron or migration runner |
| Strings | Error codes only | Its i18n for the messages |
| Tests | `node:test` suite | Its runner (vitest/jest work unchanged) |

## Quick start

1. Copy the types and time helpers — [strategy-model.md](references/strategy-model.md),
   [timezones.md](references/timezones.md).
2. Copy the generators — [generators.md](references/generators.md).
3. Copy availability and the service entry point — [availability.md](references/availability.md).
4. Pick a backend — [firestore-backend.md](references/firestore-backend.md) or
   [supabase-backend.md](references/supabase-backend.md).
5. Copy the test suite and run it under at least two `TZ` values — [testing.md](references/testing.md).
6. Porting an existing engine? Read [provenance.md](references/provenance.md) for the defect list and
   fix order.

## Reference directory

| Scenario | Trigger keywords | Reference |
|---|---|---|
| Strategy shape, fields, validation | RANGE, PERIODICITY, ONCE, DAY_RANGE, tendency, timeOffset, slotsLimit, overnight, strategy-bounds-invalid | [strategy-model.md](references/strategy-model.md) |
| Time zones and DST | IANA, wall clock, UTC, DST gap, overlap, fromZonedTime, offset, storageVersion, legacy | [timezones.md](references/timezones.md) |
| Producing slots | generateSlots, horizon, maxSlots, truncated, slot ID, md5, chronological, viewport | [generators.md](references/generators.md) |
| What is still free | availability, seats, BOOKING, BLOCK, working hours, resource, capacity, overbooking, prepareBookingSlots | [availability.md](references/availability.md) |
| Firestore | firebase-admin, Timestamp, composite index, array-contains-any, onSnapshot, runTransaction | [firestore-backend.md](references/firestore-backend.md) |
| Supabase / Postgres | timestamptz, jsonb, RLS, with check, gin index, book_slots, advisory lock | [supabase-backend.md](references/supabase-backend.md) |
| Proving it | node:test, TZ=, DST regression, Lord Howe, Auckland, differential check | [testing.md](references/testing.md) |
| Why it is built this way | provenance, defects, kept deliberately, port order | [provenance.md](references/provenance.md) |
