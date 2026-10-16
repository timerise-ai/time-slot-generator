# Availability

Generated slots say what *could* be booked. Availability subtracts what already is.

## What reduces a slot

| Source | Kind | Effect | Notes |
|---|---|---|---|
| Active booking of this service | `BOOKING` | `quantity -= seats` (default 1) | Overlap, not slot ID: a 09:00–10:00 booking also reduces an overlapping 09:30 slot of another strategy |
| Blackout / exception on the service, its location or its hosts | `BLOCK` | Slot removed | Holidays, maintenance, sick days |
| External calendar busy time of a host | `BLOCK` | Slot removed | Read the stored UTC instants; see below |
| A host's booking of a **different** service | `BLOCK` | Slot removed | The host is elsewhere |
| A host's booking of **this** service | — | Already a `BOOKING` | Adding it as `BLOCK` too hides slots with free seats |
| Shared resource (room, court, device) | cap | `quantity = min(quantity, capacity − overlapping bookings)` | Counts bookings, not seats: one booking occupies one unit |
| Working hours of the location | filter | Slot removed unless it starts and ends inside that local day's hours | A day missing from the map is closed |

Inactive bookings (`CANCELED`, `REJECTED`, or the host's equivalents) are ignored everywhere. Full slots
stay in the result with `quantity: 0` so a UI can show "sold out"; blocked slots disappear.

Overlap is half-open: a slot ending at 10:00 does not collide with a booking starting at 10:00.

## External calendars

Store each busy event as a block with **UTC `from`/`to`**. A calendar's own zone (the user's Google
calendar, an Outlook mailbox) is irrelevant once the event is an instant. Two mistakes to avoid:

- Storing the event's local wall clock and converting it later with the *project's* zone — correct only
  when the calendar and project zones agree.
- Storing UTC and then converting it again as if it were wall clock — shifts every event by the offset.

All-day events become blocks from local midnight to the next local midnight in the calendar's zone,
via `zonedToUtc(date, '00:00', calendarZone)`.

## Working hours

`WorkingHours` maps `MONDAY`…`SUNDAY` to `{ timeFrom, timeTo }` in the location's zone. Missing
`timeFrom` is 00:00, missing `timeTo` is 24:00, and `timeTo <= timeFrom` closes on the next day (a bar
open 18:00–02:00). The check compares instants on the slot's local start date, so a 23:00–00:00 slot is
outside 08:00–17:00 — comparing only "HH:mm" strings lets it through because "00:00" < "17:00".

Working hours are a filter on top of strategies. Prefer encoding hours in the strategies themselves;
keep working hours for location-wide limits that apply to many services.

## The engine side

```ts
// slots/availability.ts — generated slots minus what is already taken. Pure.
import { addDays, formatWithOffset, timeToSeconds, wallClock, zonedToUtc } from './timezone';
import { DAY_ORDER, type DayOfWeek, type GeneratedSlot } from './types';

export interface Interval {
  from: Date;
  to: Date;
}

/**
 * Something occupying time on the bookable unit.
 * - BOOKING: an active booking of this service; subtracts `seats` (default 1).
 * - BLOCK:   a blackout, a host's busy time, an external calendar event, or a
 *            host's booking of a *different* service; removes the slot.
 */
export interface BusyInterval extends Interval {
  kind: 'BOOKING' | 'BLOCK';
  seats?: number | null;
}

/** A shared resource (room, court, device) with N units; each overlapping booking takes one. */
export interface ResourceCap {
  resourceId: string;
  capacity: number;
  bookings: Interval[];
}

/** Missing day = closed. Missing timeFrom = 00:00, missing timeTo = 24:00. timeTo <= timeFrom = closes next day. */
export type WorkingHours = Partial<Record<DayOfWeek, { timeFrom?: string | null; timeTo?: string | null }>>;

export interface AvailabilityInput {
  generated: GeneratedSlot[];
  /** One-off slots created by hand, outside any strategy. */
  manualSlots?: GeneratedSlot[];
  busy: BusyInterval[];
  resources?: ResourceCap[];
  workingHours?: WorkingHours | null;
  /** Zone the working hours are expressed in (the location's or the project's). */
  timeZone: string;
}

/** Half-open: a slot ending at 10:00 does not overlap a booking starting at 10:00. */
export function overlaps(a: Interval, b: Interval): boolean {
  return a.from.getTime() < b.to.getTime() && b.from.getTime() < a.to.getTime();
}

/**
 * The bookable map, keyed by slotId. Blocked slots and slots outside working
 * hours are absent; full slots are present with quantity <= 0.
 *
 * The booking path must look slots up in THIS map, built by the same function
 * the listing uses. Filtering in the listing only lets anyone who knows an ID
 * book what the listing hides.
 */
export function buildAvailability(input: AvailabilityInput): Map<string, GeneratedSlot> {
  const map = new Map<string, GeneratedSlot>();
  for (const original of [...input.generated, ...(input.manualSlots ?? [])]) {
    const slot = { ...original };
    if (input.workingHours && !isWithinWorkingHours(slot, input.timeZone, input.workingHours)) continue;

    let blocked = false;
    for (const busy of input.busy) {
      if (!overlaps(slot, busy)) continue;
      if (busy.kind === 'BLOCK') {
        blocked = true;
        break;
      }
      slot.quantity -= seatsOf(busy.seats);
    }
    if (blocked) continue;

    for (const resource of input.resources ?? []) {
      const taken = resource.bookings.filter(booking => overlaps(slot, booking)).length;
      slot.quantity = Math.min(slot.quantity, resource.capacity - taken);
    }
    map.set(slot.slotId, slot);
  }
  return map;
}

/** Compared as instants on the slot's local start date, so a slot ending after midnight is outside 08:00–17:00. */
export function isWithinWorkingHours(slot: Interval, timeZone: string, hours: WorkingHours): boolean {
  const { date, weekday } = wallClock(slot.from, timeZone);
  const day = hours[DAY_ORDER[weekday] as DayOfWeek];
  if (!day) return false;
  const timeFrom = day.timeFrom ?? '00:00';
  const timeTo = day.timeTo ?? '24:00';
  const closesNextDay = timeToSeconds(timeTo) <= timeToSeconds(timeFrom);
  const open = zonedToUtc(date, timeFrom, timeZone);
  const close = zonedToUtc(closesNextDay ? addDays(date, 1) : date, timeTo, timeZone);
  return slot.from.getTime() >= open.getTime() && slot.to.getTime() <= close.getTime();
}

export interface SlotOutput {
  slotId: string;
  strategyId: string;
  /** UTC, "Z". Clients send this back; it is the canonical value. */
  dateTimeFrom: string;
  dateTimeTo: string;
  /** Same instant with the local offset, "2026-11-09T09:00:00+01:00". For display. */
  dateTimeISOFrom: string;
  dateTimeISOTo: string;
  duration: string;
  /** Free seats, never negative. */
  quantity: number;
  capacity: number;
  labels: string[];
}

export function toSlotOutput(slots: Iterable<GeneratedSlot>, timeZone: string): SlotOutput[] {
  return [...slots]
    .sort((a, b) => a.from.getTime() - b.from.getTime())
    .map(slot => ({
      slotId: slot.slotId,
      strategyId: slot.strategyId,
      dateTimeFrom: slot.from.toISOString(),
      dateTimeTo: slot.to.toISOString(),
      dateTimeISOFrom: formatWithOffset(slot.from, timeZone),
      dateTimeISOTo: formatWithOffset(slot.to, timeZone),
      duration: slot.duration,
      quantity: Math.max(0, slot.quantity),
      capacity: slot.capacity,
      labels: slot.labels,
    }));
}

/**
 * The booking path: resolve requested IDs against ONE availability map built
 * from now to the horizon. Never regenerate per requested slot — each
 * generation walks every strategy to the horizon, and an 18-slot booking that
 * did so 18 times ran past a 100s edge timeout.
 */
export function resolveSlotsForBooking(
  availability: Map<string, GeneratedSlot>,
  slotIds: string[],
  seats: number
): GeneratedSlot[] {
  return slotIds.map(slotId => {
    const slot = availability.get(slotId);
    if (!slot) throw new Error(`slot-no-longer-available: ${slotId}`);
    if (slot.quantity < seats) throw new Error(`slot-has-not-enough-capacity: ${slotId}`);
    return slot;
  });
}

function seatsOf(seats: number | null | undefined): number {
  return typeof seats === 'number' && Number.isFinite(seats) && seats > 0 ? seats : 1;
}
```

## The service entry point

`listServiceSlots` serves the listing; `prepareBookingSlots` serves the booking path. Both build the
same availability map, so nothing the listing hides can be booked by ID.

```ts
// slots/service.ts — the one entry point the API layer calls. The store is the seam.
import {
  buildAvailability,
  resolveSlotsForBooking,
  toSlotOutput,
  type BusyInterval,
  type Interval,
  type ResourceCap,
  type SlotOutput,
  type WorkingHours,
} from './availability';
import { generateSlots, type GenerateOptions } from './generators';
import { DEFAULT_LIMITS, type GeneratedSlot, type SlotStrategy } from './types';

const MS_PER_DAY = 86_400_000;

/** What the engine needs to know about the bookable thing. Rename to the host's vocabulary. */
export interface BookableService {
  serviceId: string;
  /** Default zone for strategies without a pin, for working hours and for display. */
  timeZone: string;
  hostIds: string[];
  /** The location whose working hours apply, if any. */
  locationId: string | null;
  resourceIds: string[];
}

export interface SlotStore {
  listStrategies(serviceId: string): Promise<SlotStrategy[]>;
  listManualSlots(serviceId: string): Promise<GeneratedSlot[]>;
  /**
   * Everything overlapping `range`: this service's active bookings (BOOKING),
   * plus blackouts, external-calendar busy time, and the hosts' bookings of
   * OTHER services (BLOCK). A host's booking of this same service is already a
   * BOOKING; adding it again as BLOCK would hide a slot that still has seats.
   */
  listBusy(service: BookableService, range: Interval): Promise<BusyInterval[]>;
  listResources(resourceIds: string[], range: Interval): Promise<ResourceCap[]>;
  getWorkingHours(locationId: string): Promise<WorkingHours | null>;
}

export interface ListSlotsResult {
  slots: SlotOutput[];
  truncated: boolean;
  failed: { strategyId: string; reason: string }[];
}

/** The listing: slots in a viewport, with what is left of each. */
export async function listServiceSlots(
  store: SlotStore,
  service: BookableService,
  window: { now: Date; from?: Date; to?: Date },
  options: GenerateOptions = {}
): Promise<ListSlotsResult> {
  const { availability, truncated, failed } = await computeAvailability(store, service, window, options);
  const fromMs = (window.from ?? window.now).getTime();
  const visible = [...availability.values()].filter(
    slot => slot.from.getTime() >= fromMs && (!window.to || slot.from.getTime() <= window.to.getTime())
  );
  return { slots: toSlotOutput(visible, service.timeZone), truncated, failed };
}

/**
 * The booking path. One generation from now to the horizon, then N lookups.
 * Run it before the write transaction; inside the transaction, re-count seats
 * from the bookings table itself (see the capacity check in availability.md).
 */
export async function prepareBookingSlots(
  store: SlotStore,
  service: BookableService,
  slotIds: string[],
  seats: number,
  now: Date,
  options: GenerateOptions = {}
): Promise<GeneratedSlot[]> {
  const { availability } = await computeAvailability(store, service, { now }, options);
  return resolveSlotsForBooking(availability, slotIds, seats);
}

async function computeAvailability(
  store: SlotStore,
  service: BookableService,
  window: { now: Date; from?: Date; to?: Date },
  options: GenerateOptions
): Promise<{ availability: Map<string, GeneratedSlot>; truncated: boolean; failed: ListSlotsResult['failed'] }> {
  const limits = options.limits ?? DEFAULT_LIMITS;
  const horizon = new Date(window.now.getTime() + limits.maxHorizonDays * MS_PER_DAY);

  const [strategies, storedManualSlots, workingHours] = await Promise.all([
    store.listStrategies(service.serviceId),
    store.listManualSlots(service.serviceId),
    service.locationId ? store.getWorkingHours(service.locationId) : Promise.resolve(null),
  ]);

  // Viewport-free generation would be correct but wasteful; the viewport is
  // safe to pass because limits are counted from the strategy start, not from it.
  const generation = generateSlots(
    strategies,
    { now: window.now, from: window.from, to: window.to, timeZone: service.timeZone },
    options
  );
  const generated = generation.slots;
  const manualSlots = storedManualSlots.filter(slot => slot.from.getTime() >= window.now.getTime());
  const all = [...generated, ...manualSlots];
  if (all.length === 0) {
    return { availability: new Map(), truncated: generation.truncated, failed: generation.failed };
  }

  // Only fetch busy time for the span the slots actually cover. A loop, not
  // Math.min(...spread): tens of thousands of arguments overflow the stack.
  let fromMs = Infinity;
  let toMs = -Infinity;
  for (const slot of all) {
    fromMs = Math.min(fromMs, slot.from.getTime());
    toMs = Math.max(toMs, slot.to.getTime());
  }
  const range: Interval = { from: new Date(fromMs), to: new Date(Math.min(toMs, horizon.getTime() + MS_PER_DAY)) };
  const [busy, resources] = await Promise.all([
    store.listBusy(service, range),
    service.resourceIds.length ? store.listResources(service.resourceIds, range) : Promise.resolve([]),
  ]);

  const availability = buildAvailability({
    generated,
    manualSlots,
    busy,
    resources,
    workingHours,
    timeZone: service.timeZone,
  });
  return { availability, truncated: generation.truncated, failed: generation.failed };
}
```

## The booking path

1. Authenticate and resolve the service **from storage** (tenant comes from the stored row).
2. `prepareBookingSlots(store, service, slotIds, seats, now)` — one generation, N lookups. Errors:
   `slot-no-longer-available` (ID unknown: wrong service, past, blocked, outside hours, strategy edited),
   `slot-has-not-enough-capacity`.
3. Open the write transaction and **re-count seats from the bookings themselves** — Firestore:
   `assertCapacityInTransaction` in [firestore-backend.md](firestore-backend.md); Postgres: the
   `book_slots` function in [supabase-backend.md](supabase-backend.md). Generation ran outside the
   transaction and two guests can both have seen the last seat.
4. Write the booking with a **copy** of each slot's `from`, `to`, `duration`, `slotId` and `strategyId`,
   plus `seats`, `hostIds`, `resourceIds` and an overall `from`/`to` span for range queries.
5. Return the booked slots with both the UTC and the local-offset times.

Rescheduling is the same path for the new slots, with the old booking cancelled in the same transaction.

## Output

| Field | Example | Notes |
|---|---|---|
| `slotId` | `9b2f…` | Send back to book |
| `dateTimeFrom` / `dateTimeTo` | `2026-11-09T08:00:00.000Z` | Canonical UTC |
| `dateTimeISOFrom` / `dateTimeISOTo` | `2026-11-09T09:00:00+01:00` | Display; same instant |
| `duration` | `PT1H` | |
| `quantity` / `capacity` | `2` / `4` | Free seats / seats at creation |
| `labels`, `strategyId` | | Filtering in the UI |

Clients must render with the offset field or format the UTC value in the service's zone — never in the
browser's zone unless that is what the product wants.

## Checklist

- [ ] Listing and booking both go through `buildAvailability`
- [ ] Host bookings of the same service excluded from `BLOCK`
- [ ] External calendar events stored as UTC instants
- [ ] Capacity re-checked in the write transaction, in seats
- [ ] Booking stores its own copy of slot instants
