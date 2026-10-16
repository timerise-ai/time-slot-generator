# Adaptation

How the module lands in a host app: what the host must already have, the seams it fills, the names it
renames, the order of work, and the non-negotiables that do not bend.

## The host must already have

- A **Next.js App Router** app with route handlers on the Node.js runtime (`slot-id.ts` uses `node:crypto`).
  The engine files import nothing from Next.js, so the same files also serve any other Node server.
- **Firestore** through `firebase-admin`, or **Supabase/Postgres** through `@supabase/supabase-js`.
- A **service** record of its own: the bookable thing, with a time zone, its hosts, an optional location and
  shared resources.
- **Staff** who are allowed to write strategies, behind the host's own auth.

## Seam contract

| Seam | This skill ships | The host supplies |
|---|---|---|
| Domain words | Service, Strategy, Booking, Host, Location, Resource, BusyBlock, project, and the rename table below | Its own word for each |
| Tenant scope | `projectId`, read from the stored service | Its org, workspace or account |
| Auth guard | None inside the engine; the booking route marks where it goes | Its guard before `saveStrategy` and, when guests sign in, before booking |
| Data access | `SlotStore` with Firestore and Supabase implementations | Its project, and the schema or indexes applied |
| Routes | `app/api/slots` and `app/api/bookings` on Supabase ([nextjs-routes.md](nextjs-routes.md)) | Its URL scheme, and a Firestore swap if it is on Firestore |
| Clock | `now` injected everywhere | `new Date()` at the edge, fixed in tests |
| Slot IDs | md5 via `node:crypto` behind `SlotIdHasher` | A pure md5 on the Edge runtime |
| Validation | `validateStrategy` with `code: detail` errors | A zod or valibot schema mapping to it, if it has one |
| Background work | None required; an optional one-off normalization | Its cron or migration runner |
| Strings | Error codes only | Its i18n for the messages |
| UI | None; the listing returns UTC and local-offset times | Its slot picker, styled its own way |
| Tests | A `node:test` suite that also runs under Vitest | Its runner and its `npm test` script |

## Rename table

The canonical vocabulary is the left column, and every reference uses it. A host that calls things
differently renames the domain words in its own code and keeps the identifiers below.

| Canonical | Common host names |
|---|---|
| Service | Event type, offering, class, activity, product |
| Strategy | Schedule, availability rule, opening hours |
| Booking | Reservation, appointment, order |
| Host | Staff, provider, instructor, employee |
| Location | Venue, branch, site |
| Resource | Room, court, lane, device, equipment |
| BusyBlock | Blackout, time off, calendar event |
| project | Organization, workspace, account, tenant |

**The authoring contract stays as written**: `generateSlots`, `buildAvailability`, `listServiceSlots`,
`prepareBookingSlots`, `resolveSlotsForBooking`, `toSlotOutput`, `SlotStore`, `BookableService`,
`validateStrategy`, `normalizeStrategy`, `saveStrategy`, `assertCapacityInTransaction`, `md5SlotId`,
`slotIdInput`, the strategy type names, the error codes and the SQL function `book_slots`. They are the
names the references, the tests and the ledger share.

## Integration points

- **Storage.** Add the columns the engine reads to the host's existing `services`, `locations` and
  `bookings` rather than creating new tables ([supabase-backend.md](supabase-backend.md),
  [firestore-backend.md](firestore-backend.md)). Bookings are written only through `book_slots` or
  `assertCapacityInTransaction`.
- **Auth.** Strategy writes need staff. Public listings and bookings go through the routes with a
  server-side client; the anon role reads nothing directly.
- **Tenancy.** Every query is scoped by a service the server read from storage, never by an ID from the
  request body.
- **Styling.** The module ships no UI. The listing answer carries `dateTimeISOFrom` with the service's
  offset, so the host's picker renders local times without converting in the browser.
- **i18n.** Errors are stable codes (`slot-no-longer-available`, `strategy-bounds-invalid`, and the rest in
  [strategy-model.md](strategy-model.md)); the host maps them to its messages.

## Order of work

1. Read the host: data store, auth, how services, locations and bookings are stored, test runner. Fill the
   right-hand column of the seam contract and the rename table before writing code.
2. Copy `types.ts` and `validate.ts` ([strategy-model.md](strategy-model.md)), `timezone.ts`, `duration.ts`
   and `normalize.ts` ([timezones.md](timezones.md)), then `slot-id.ts` and `generators.ts`
   ([generators.md](generators.md)).
3. Copy `availability.ts` and `service.ts` ([availability.md](availability.md)).
4. Copy `engine.test.ts` ([testing.md](testing.md)), wire it to `npm test` under four time zones, and run
   it. Nothing so far touches a database.
5. Apply the schema and copy `supabase-store.ts`, or set up the Firestore indexes and copy
   `firestore-store.ts`.
6. Copy the routes ([nextjs-routes.md](nextjs-routes.md)) and put the host's guard where they mark it.
7. Build the host's slot picker and strategy editor on the listing answer and `saveStrategy`.

## Non-negotiables

The same five rules as `SKILL.md` and the README, in the same order.

1. **Never change the slot ID input.** `md5("<utcStartISO>;<duration>;<strategyId>")` is the identity every
   client and every booking holds; a different input orphans them all.
2. **Never infer "overnight" from end before start.** A reversed range is far more often a typo than a night
   shift, so an overnight window needs `overnight: true`.
3. **Never generate without a horizon.** `maxHorizonDays` and `maxSlots` cap every strategy, since one with
   no end would otherwise generate until 2038.
4. **Never filter availability in the listing only.** The listing and the booking path read one
   `buildAvailability` map, so an ID cannot book what the listing hides.
5. **Never count a host's booking of this service as host busy time.** It already subtracts seats as a
   `BOOKING`; blocking on it as well hides a slot that still has room.

Everything else, the words, the screens, the guard and the store, is the host app's.
