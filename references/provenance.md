# Provenance

This module is the slot engine of a booking platform, written down by the engineer who builds it: strategy
storage, slot generation, availability and the booking path, audited against the earlier implementation
and rebuilt as plain TypeScript served from Next.js route handlers, on Firestore or Supabase/Postgres. The
earlier implementation was a long-running API server on a document database, and it had already been through
one hardening pass: true-UTC storage with a read-time normalizer, local-calendar weekday and month bucketing,
bounded listener caches, and a single generation per booking. The audit ran that engine against DST,
far-from-UTC and viewport cases. On ordinary windows the templates produce the same slot IDs and instants as
the earlier implementation, which the differential check in [testing.md](testing.md) confirms.

This file is the ledger: what the audit changed and how the templates verify it, what was kept deliberately
and why it is safe, what was designed here and has never run in production, and what the agent evals found
in the designed parts. 32 entries. Each audit fix is pinned by a test in [testing.md](testing.md).

## Fixed in the templates

### 1. PERIODICITY dropped every weekday but the first when cut short

Candidates were produced one weekday at a time, all Sundays of the month and then all Mondays, and
generation returned at the first date past the cutoff. A one-week view returned only the Sunday. The same
early exit fired on `end.dateTime`, on the booking window and on `slotsLimit`, so it also reached the
booking path. **Shipped:** each month's dates are merged and sorted before emitting
([generators.md](generators.md)).

### 2. A local day was skipped on DST transitions

Days were advanced by adding 24 hours to an instant. When "now" was late in the evening before a
23-hour day, the next step landed two local days later; on a 25-hour day the same date came up twice and
inflated `slotsLimit` counts. **Shipped:** iteration over `YYYY-MM-DD` local dates.

### 3. DAY_RANGE slots were 24 hours on 23- and 25-hour days

The day slot ended at start + 24h, that is at 01:00 or 23:00 local on transition days, leaving an hour
uncovered or double-covered. **Shipped:** slots run to the next local midnight.

### 4. Output depended on the server's time zone

Local strings were parsed with `new Date('...T09:00')` (process zone) and weekdays read with
`getUTCDay()` on a "zoned" Date (correct only in a UTC process). The earlier implementation ran in a UTC
process, so it was latent; any other deployment or a developer laptop got different slots. **Shipped:**
Intl-only helpers ([timezones.md](timezones.md)) and a suite run under four process zones.

### 5. `slotsLimit` and `daysLimit` counted from the viewport

The count began at the viewport start, so "the first 10 slots" differed per requested week, and a slot
shown in a later week was absent from the booking-path generation. **Shipped:** limits count from the
strategy's first bookable slot.

### 6. The pinned strategy zone was never read

Strategies stored their authoring `timeZone`, but generation used the project's current zone, so
changing it reinterpreted every strategy. **Shipped:** `strategy.timeZone ?? service.timeZone`.

### 7. External busy time was converted twice or with the wrong zone

Blocks stored both a wall-clock field and a UTC field; generation converted the wall-clock field with the
*project* zone. Events stored as true UTC were shifted by the project offset; events from a calendar in
another zone were off by the difference. **Shipped:** blocks are UTC instants only
([availability.md](availability.md)).

### 8. Working hours compared clock strings

"00:00" sorts before "17:00", so a slot ending at midnight passed 08:00 to 17:00 hours. **Shipped:**
instants on the slot's local start date, with next-day closing.

### 9. Working hours applied to the listing only

The booking path looked IDs up in a map built without the working-hours filter, so a slot hidden from the
listing could be booked by ID. **Shipped:** one `buildAvailability` for both paths.

### 10. Malformed time strings became midnight

An unparseable `time` either fell back to 00:00 or produced an invalid date and no slots, with no
error either way. **Shipped:** `strategy-time-invalid` at write time.

### 11. A limit of zero produced one slot

The limit was checked only after a slot had been yielded, so `slotsLimit: 0` gave one slot while the
documentation promised none; a stored `null` likewise stopped the strategy after its first slot.
**Shipped:** `0` yields nothing, `null` means no cap.

## Kept deliberately

- **Slot ID = md5 of `utcStart;duration;strategyId`.** Not a security token, only an identity; changing
  it orphans every existing booking reference.
- **Capacity by overlap, not by slot ID.** A booking reduces every overlapping slot, so two strategies
  covering the same hours share one capacity. This is what a single bookable unit means.
- **A host's other commitments block the slot entirely.** A person cannot be in two places.
- **Resources count bookings, not seats.** One booking occupies one room.
- **`slotInterval >= slotDuration`.** Overlapping slots from one strategy would share no capacity and
  double-sell.
- **Full slots are returned with `quantity: 0`; blocked slots are removed.** "Sold out" and "not offered"
  are different messages.
- **A DAY_RANGE day already begun is not offered.**
- **`PT0S` offsets mean "not set".**
- **One generation per booking, then lookups.** The earlier implementation once regenerated per requested
  slot, and an 18-slot booking ran past a 100-second edge timeout. `prepareBookingSlots` generates once.

## Designed here, not yet run in production

- **Overnight windows** (`overnight: true`), explicit so a reversed range stays a validation error.
- **Generation horizon** (`maxHorizonDays`, `maxSlots`, `truncated`), so no strategy generates to 2038.
- **Per-strategy time zone** honoured by the generators.
- **Failure isolation**: one malformed strategy is reported in `failed` instead of failing the service.
- **Supabase/Postgres backend** with the `book_slots` capacity function.
- **Validation codes** for zone, quantity, durations, times, limits and horizon.
- **Next.js route handlers** for the listing and the booking ([nextjs-routes.md](nextjs-routes.md)), with
  credentials read at request time and errors answered by code.

## Found by the agent evals

Defects in the parts designed here, found when agents installing the skill edited a template to close them,
and reproduced on a scratch Postgres with Supabase's default grants before the template changed.

- **The public key could book.** Postgres lets PUBLIC execute a new function and Supabase grants anon on
  every table in `public`, so the anon key could call `book_slots` with any capacity and read the host
  tables the schema creates. **Shipped (0.1.1):** RLS on every table in the schema, and `book_slots`
  executable by `service_role` only. Apps on 0.1.0 apply both to their database.
- **The transaction re-counted by slot ID.** The listing counts capacity by overlap, but `book_slots` and
  `assertCapacityInTransaction` counted only bookings holding the same ID, so overlapping slots of two
  strategies could both sell the last seat at once. **Shipped (0.1.1):** one lock per service and a seat sum
  over overlapping bookings, in both stores.
- **Resource blackouts were never read.** The schema allows `owner_type = 'resource'`, but neither store
  queried those blocks, so a closed room still offered its slots. **Shipped (0.1.1):** both stores include
  the service's resources among the block owners.
- **The public listing had no viewport by default.** `MAX_VIEWPORT_DAYS` applied only when both `from` and
  `to` were sent; with neither, one weekday strategy listed 8337 slots up to the two-year horizon.
  **Shipped (0.1.2):** `from` defaults to now and `to` to `MAX_VIEWPORT_DAYS` after it.
- **Supabase bookings kept no slot copy.** The booking path promises a copy of each slot, which the
  Firestore write kept and `book_slots` did not, so an edited strategy left a booking with IDs alone.
  **Shipped (0.1.2):** a `slots` jsonb column filled by `book_slots` from what the route passes.

The first two and the last are checked on that scratch Postgres before each release; the engine suite does
not reach the stores or the routes, so the third and the fourth rest on the code alone.

## Fix order when rebuilding an existing engine

Most damaging first:

1. PERIODICITY chronological emission (#1): visible in every bounded view and in the booking path.
2. External busy time from UTC fields (#7): double bookings against Outlook and Google events.
3. Limits counted from the strategy start (#5): failed bookings of visible slots.
4. Local-date iteration and DAY_RANGE midnight (#2, #3): twice a year, in every zone with DST.
5. Pinned zone (#6), working hours (#8, #9), process-TZ independence (#4).

Run the differential check from [testing.md](testing.md) before and after: ordinary windows must not
change.
