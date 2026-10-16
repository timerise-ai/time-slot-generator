# Generators

`generateSlots(strategies, window, options)` turns strategies into concrete slots. It is pure: the clock
is `window.now`, nothing is read or written, and the result depends only on its arguments.

## Contract

| Input | Meaning |
|---|---|
| `window.now` | Injected clock. Slots starting before it are never produced |
| `window.from` / `window.to` | Viewport, inclusive on slot start. Defaults: `now` / the horizon |
| `window.timeZone` | Default zone for strategies without a pinned `timeZone` |
| `options.limits.maxHorizonDays` | Nothing past `now + maxHorizonDays`, whatever a strategy says (default 731) |
| `options.limits.maxSlots` | Per-strategy cap; hitting it sets `truncated` (default 50 000) |
| `options.hashSlotId` | Slot ID hash; default md5 via `node:crypto` |

| Output | Meaning |
|---|---|
| `slots` | Sorted by start, unique by `slotId` |
| `truncated` | A strategy hit `maxSlots`; log it and show a "showing first N" hint |
| `failed` | Strategies that threw (bad duration, unknown type) with the reason. The others still generate |

## Invariants

| Invariant | Why |
|---|---|
| Each generator yields in chronological order | Early exit at the first slot past a cutoff is only correct then |
| Local dates are iterated as `YYYY-MM-DD` | `+24h` skips or repeats DST days |
| PERIODICITY merges and sorts a month's dates before emitting | Weekday-by-weekday emission stops after the first weekday |
| Limits count from `max(now + lead, start.dateTime)` | Viewport-relative counting makes visible slots unbookable |
| Generation stops at `min(end.dateTime, now + end.timeOffset, viewport end, horizon)` | One cutoff, computed once |
| A day that has already begun is not offered as a DAY_RANGE slot | Whole-day products start at midnight |
| ONCE ignores lead time and booking window | It is an explicit instant chosen by an operator |
| RANGE starts iterating one day early | Yesterday's overnight window may still have slots ahead |

## Slot identity

```
slotId = md5("<from as UTC ISO>;<slotDuration>;<strategyId>")   // DAY_RANGE uses "PT24H"
```

Slots are not stored, so the ID is how a booking finds its slot again: the booking path regenerates and
looks the ID up. Editing a strategy's times therefore changes the IDs of its future slots, which is
correct, since those slots no longer exist, but bookings already made keep their own copy of the slot's
instants and must never be re-resolved from the strategy.

On an edge runtime without `node:crypto`, pass a pure-JS md5 as `hashSlotId`. Switching to SHA-256 is
fine for a new system and breaks every existing ID for an old one.

```ts
// lib/slots/slot-id.ts: deterministic slot identity.
//
// A slot is never stored; it is regenerated on every read and found again by
// ID. The ID therefore has to be a pure function of what defines the slot:
// its UTC start, its duration and the strategy that made it. Change any input
// to this string and every ID already handed to a client or stored on a
// booking stops resolving.
import { createHash } from 'node:crypto';

export type SlotIdHasher = (raw: string) => string;

export const md5SlotId: SlotIdHasher = raw => createHash('md5').update(raw).digest('hex');

export function slotIdInput(from: Date, duration: string, strategyId: string): string {
  return `${from.toISOString()};${duration};${strategyId}`;
}
```

## The engine

```ts
// lib/slots/generators.ts: strategies -> concrete slots. Pure: no I/O, no clock.
import { durationToMs } from './duration';
import { md5SlotId, slotIdInput, type SlotIdHasher } from './slot-id';
import { addDays, daysInMonth, wallClock, weekdayOf, zonedToUtc } from './timezone';
import {
  DAY_ORDER,
  DEFAULT_LIMITS,
  MONTH_ORDER,
  type DayOfWeek,
  type DayRangeStrategy,
  type GeneratedSlot,
  type GenerationLimits,
  type GenerationWindow,
  type Month,
  type OnceStrategy,
  type PeriodicityStrategy,
  type RangeStrategy,
  type SlotStrategy,
  type StrategyBound,
  type Tendency,
} from './types';

const MS_PER_DAY = 86_400_000;
const DAY_RANGE_DURATION = 'PT24H';

export interface GenerateOptions {
  limits?: GenerationLimits;
  hashSlotId?: SlotIdHasher;
}

export interface GenerationResult {
  /** Sorted by start, unique by slotId. */
  slots: GeneratedSlot[];
  /** True when a strategy hit `maxSlots`; the list is incomplete. */
  truncated: boolean;
  /** Strategies that could not be generated, with the reason. The rest still are. */
  failed: { strategyId: string; reason: string }[];
}

interface Context {
  nowMs: number;
  viewFromMs: number;
  viewToMs: number;
  /** now + maxHorizonDays: nothing is generated past it. */
  horizonMs: number;
  defaultTimeZone: string;
  hash: SlotIdHasher;
}

export function generateSlots(
  strategies: SlotStrategy[],
  window: GenerationWindow,
  options: GenerateOptions = {}
): GenerationResult {
  const limits = options.limits ?? DEFAULT_LIMITS;
  const nowMs = window.now.getTime();
  const horizonMs = nowMs + limits.maxHorizonDays * MS_PER_DAY;
  const ctx: Context = {
    nowMs,
    viewFromMs: (window.from ?? window.now).getTime(),
    viewToMs: Math.min(window.to?.getTime() ?? horizonMs, horizonMs),
    horizonMs,
    defaultTimeZone: window.timeZone,
    hash: options.hashSlotId ?? md5SlotId,
  };

  const byId = new Map<string, GeneratedSlot>();
  const failed: GenerationResult['failed'] = [];
  let truncated = false;

  for (const strategy of strategies) {
    try {
      let count = 0;
      for (const slot of generateOne(strategy, ctx)) {
        // Cap per strategy, so one runaway strategy cannot starve the others.
        if (++count > limits.maxSlots) {
          truncated = true;
          break;
        }
        byId.set(slot.slotId, slot);
      }
    } catch (error) {
      // One malformed strategy must not take down availability for the service.
      failed.push({ strategyId: strategy.strategyId, reason: (error as Error).message });
    }
  }

  const slots = [...byId.values()].sort((a, b) => a.from.getTime() - b.from.getTime());
  return { slots, truncated, failed };
}

function generateOne(strategy: SlotStrategy, ctx: Context): Iterable<GeneratedSlot> {
  switch (strategy.strategyType) {
    case 'RANGE':
      return generateRange(strategy, ctx);
    case 'PERIODICITY':
      return generatePeriodicity(strategy, ctx);
    case 'ONCE':
      return generateOnce(strategy, ctx);
    case 'DAY_RANGE':
      return generateDayRange(strategy, ctx);
    default:
      throw new Error(`strategy-type-invalid: ${(strategy as { strategyType: string }).strategyType}`);
  }
}

// ---------------------------------------------------------------- bounds

interface Bounds {
  /** First instant a slot may start at; limits count from here. */
  countFromMs: number;
  /** Generation starts here: countFrom when a limit applies, else the later of countFrom and the viewport. */
  iterateFromMs: number;
  /** Last instant a slot may start at (inclusive). */
  stopAtMs: number;
}

function resolveBounds(
  start: StrategyBound,
  end: StrategyBound,
  hasLimit: boolean,
  ctx: Context
): Bounds | undefined {
  // A zero offset means "not set", as it always has: PT0S is not a zero-length window.
  const leadMs = start.timeOffset ? durationToMs(start.timeOffset) : 0;
  const windowMs = end.timeOffset ? durationToMs(end.timeOffset) : 0;

  const countFromMs = Math.max(
    ctx.nowMs + leadMs,
    start.dateTime ? Date.parse(start.dateTime) : -Infinity
  );
  const stopAtMs = Math.min(
    end.dateTime ? Date.parse(end.dateTime) : Infinity,
    windowMs > 0 ? ctx.nowMs + windowMs : Infinity,
    ctx.viewToMs
  );
  if (Number.isNaN(countFromMs) || Number.isNaN(stopAtMs)) {
    throw new Error('strategy-bounds-invalid: start.dateTime or end.dateTime is not an ISO instant');
  }
  if (stopAtMs < countFromMs) return undefined;

  // Limits are counted from the strategy's first bookable slot, never from the
  // viewport. Otherwise "the first 10 slots" depends on which week a client is
  // looking at, and a slot shown in one view is missing from another.
  const iterateFromMs = hasLimit ? countFromMs : Math.max(countFromMs, ctx.viewFromMs);
  return { countFromMs, iterateFromMs, stopAtMs };
}

// ---------------------------------------------------------------- RANGE

function* generateRange(s: RangeStrategy, ctx: Context): Generator<GeneratedSlot> {
  const timeZone = s.timeZone ?? ctx.defaultTimeZone;
  const limit = s.slotsLimit ?? undefined;
  const bounds = resolveBounds(s.start, s.end, limit !== undefined, ctx);
  if (!bounds || limit === 0) return;
  const durationMs = durationToMs(s.slotDuration);
  const intervalMs = durationToMs(s.slotInterval);

  // Start a day early: yesterday's overnight window may still have slots ahead.
  let date = addDays(wallClock(new Date(bounds.iterateFromMs), timeZone).date, -1);
  const lastDate = addDays(wallClock(new Date(bounds.stopAtMs), timeZone).date, 1);
  let emitted = 0;

  // Step through local calendar dates, never through +24h. A DST day is 23 or 25
  // hours long, and adding 24h to "23:30 the evening before" skips it entirely.
  for (; date <= lastDate; date = addDays(date, 1)) {
    if (!matchesDay(date, s.daysOfWeek, s.months)) continue;
    for (const from of dailyWindow(date, s, timeZone, durationMs, intervalMs)) {
      const fromMs = from.getTime();
      if (fromMs < bounds.countFromMs) continue;
      if (fromMs > bounds.stopAtMs) return;
      if (fromMs >= ctx.viewFromMs) yield makeSlot(from, durationMs, s.slotDuration, s, ctx);
      if (limit !== undefined && ++emitted >= limit) return;
    }
  }
}

// ---------------------------------------------------------------- PERIODICITY

const TENDENCY_PICK: Record<Tendency, (days: string[]) => (string | undefined)[]> = {
  FIRST: days => [days[0]],
  SECOND: days => [days[1]],
  THIRD: days => [days[2]],
  FOURTH: days => [days[3]],
  // Only months that have a fifth such weekday.
  FIFTH: days => [days[4]],
  PENULTIMATE: days => [days[days.length - 2]],
  LAST: days => [days[days.length - 1]],
  ALL: days => days,
};

function* generatePeriodicity(s: PeriodicityStrategy, ctx: Context): Generator<GeneratedSlot> {
  const timeZone = s.timeZone ?? ctx.defaultTimeZone;
  const limit = s.slotsLimit ?? undefined;
  const bounds = resolveBounds(s.start, s.end, limit !== undefined, ctx);
  if (!bounds || limit === 0) return;
  const durationMs = durationToMs(s.slotDuration);
  const intervalMs = durationToMs(s.slotInterval);
  const weekdays = s.daysOfWeek?.length ? s.daysOfWeek : DAY_ORDER;
  const tendencies: readonly Tendency[] = s.tendency?.length ? s.tendency : ['ALL'];

  const first = wallClock(new Date(bounds.iterateFromMs - MS_PER_DAY), timeZone);
  const last = wallClock(new Date(bounds.stopAtMs), timeZone);
  let year = first.year;
  let month = first.month;
  let emitted = 0;

  // Months in the local calendar, so a +13:00 zone does not file the 1st under
  // the previous month.
  while (year < last.year || (year === last.year && month <= last.month)) {
    if (matchesMonth(month, s.months)) {
      // Every candidate date of the month, merged and sorted BEFORE emitting.
      // Walking one weekday at a time (all Sundays, then all Mondays, and so on) and
      // stopping at the first date past the cutoff drops every other weekday.
      for (const date of monthCandidates(year, month, weekdays, tendencies)) {
        for (const from of dailyWindow(date, s, timeZone, durationMs, intervalMs)) {
          const fromMs = from.getTime();
          if (fromMs < bounds.countFromMs) continue;
          if (fromMs > bounds.stopAtMs) return;
          if (fromMs >= ctx.viewFromMs) yield makeSlot(from, durationMs, s.slotDuration, s, ctx);
          if (limit !== undefined && ++emitted >= limit) return;
        }
      }
    }
    month += 1;
    if (month === 13) {
      month = 1;
      year += 1;
    }
  }
}

function monthCandidates(
  year: number,
  month: number,
  weekdays: readonly DayOfWeek[],
  tendencies: readonly Tendency[]
): string[] {
  const picked = new Set<string>();
  for (const weekday of weekdays) {
    const target = DAY_ORDER.indexOf(weekday);
    const days: string[] = [];
    for (let day = 1; day <= daysInMonth(year, month); day++) {
      const date = `${year}-${String(month).padStart(2, '0')}-${String(day).padStart(2, '0')}`;
      if (weekdayOf(date) === target) days.push(date);
    }
    for (const tendency of tendencies) {
      for (const date of TENDENCY_PICK[tendency](days)) if (date) picked.add(date);
    }
  }
  return [...picked].sort();
}

// ---------------------------------------------------------------- DAY_RANGE

function* generateDayRange(s: DayRangeStrategy, ctx: Context): Generator<GeneratedSlot> {
  const timeZone = s.timeZone ?? ctx.defaultTimeZone;
  const limit = s.daysLimit ?? undefined;
  const bounds = resolveBounds(s.start, s.end, limit !== undefined, ctx);
  if (!bounds || limit === 0) return;

  let date = wallClock(new Date(bounds.iterateFromMs), timeZone).date;
  const lastDate = addDays(wallClock(new Date(bounds.stopAtMs), timeZone).date, 1);
  let emitted = 0;

  for (; date <= lastDate; date = addDays(date, 1)) {
    if (!matchesDay(date, s.daysOfWeek, s.months)) continue;
    // Midnight to the next local midnight: 23h or 25h on DST days, not 24h.
    const from = zonedToUtc(date, '00:00', timeZone);
    const to = zonedToUtc(addDays(date, 1), '00:00', timeZone);
    const fromMs = from.getTime();
    // A day that has already begun is not offered.
    if (fromMs < bounds.countFromMs) continue;
    if (fromMs > bounds.stopAtMs) return;
    if (fromMs >= ctx.viewFromMs) {
      yield {
        slotId: ctx.hash(slotIdInput(from, DAY_RANGE_DURATION, s.strategyId)),
        strategyId: s.strategyId,
        from,
        to,
        duration: DAY_RANGE_DURATION,
        capacity: s.slotQuantity,
        quantity: s.slotQuantity,
        labels: s.labels ?? [],
      };
    }
    if (limit !== undefined && ++emitted >= limit) return;
  }
}

// ---------------------------------------------------------------- ONCE

function* generateOnce(s: OnceStrategy, ctx: Context): Generator<GeneratedSlot> {
  const from = new Date(s.start);
  const fromMs = from.getTime();
  if (Number.isNaN(fromMs)) throw new Error('strategy-bounds-invalid: ONCE start is not an ISO instant');
  if (fromMs < ctx.nowMs || fromMs < ctx.viewFromMs || fromMs > ctx.viewToMs) return;
  const to = s.end ? new Date(s.end) : new Date(fromMs + durationToMs(s.slotDuration));
  yield {
    slotId: ctx.hash(slotIdInput(from, s.slotDuration, s.strategyId)),
    strategyId: s.strategyId,
    from,
    to,
    duration: s.slotDuration,
    capacity: s.slotQuantity,
    quantity: s.slotQuantity,
    labels: s.labels ?? [],
  };
}

// ---------------------------------------------------------------- shared

/** Slot starts inside one day's start.time to end.time window, in order. */
function* dailyWindow(
  date: string,
  s: RangeStrategy | PeriodicityStrategy,
  timeZone: string,
  durationMs: number,
  intervalMs: number
): Generator<Date> {
  // A zero interval would never advance; validation rejects it, this keeps a bad row harmless.
  if (durationMs <= 0 || intervalMs <= 0) throw new Error('strategy-interval-invalid: zero duration or interval');
  const windowStart = zonedToUtc(date, s.start.time ?? '00:00', timeZone).getTime();
  const endDate = s.overnight ? addDays(date, 1) : date;
  const windowEnd = zonedToUtc(endDate, s.end.time ?? '24:00', timeZone).getTime();
  for (let t = windowStart; t + durationMs <= windowEnd; t += intervalMs) yield new Date(t);
}

function makeSlot(
  from: Date,
  durationMs: number,
  duration: string,
  s: RangeStrategy | PeriodicityStrategy,
  ctx: Context
): GeneratedSlot {
  return {
    slotId: ctx.hash(slotIdInput(from, duration, s.strategyId)),
    strategyId: s.strategyId,
    from,
    to: new Date(from.getTime() + durationMs),
    duration,
    capacity: s.slotQuantity,
    quantity: s.slotQuantity,
    labels: s.labels ?? [],
  };
}

function matchesDay(date: string, daysOfWeek?: DayOfWeek[] | null, months?: Month[] | null): boolean {
  const weekdayOk = !daysOfWeek?.length || daysOfWeek.includes(DAY_ORDER[weekdayOf(date)] as DayOfWeek);
  return weekdayOk && matchesMonth(Number(date.slice(5, 7)), months);
}

function matchesMonth(month: number, months?: Month[] | null): boolean {
  return !months?.length || months.includes(MONTH_ORDER[month - 1] as Month);
}
```

## Cost

Work is linear in days times slots per day up to the cutoff. A 5-minute RANGE strategy with no end and no
viewport is 288 slots a day; the default 731-day horizon makes that about 210 000 iterations, which is
why `maxSlots` exists and why the listing should always pass a viewport. The booking path cannot pass
one (it must see every slot to find an ID), so keep the horizon realistic for the product (90 or 365
days is typical) and set `end.timeOffset` on strategies.

## Checklist

- [ ] `now` injected; no `new Date()` in the engine
- [ ] Listing passes a viewport; booking path does not
- [ ] `truncated` and `failed` logged with the service ID
- [ ] Horizon set for the product, and strategies given a booking window
- [ ] Slot ID input unchanged from any existing data
