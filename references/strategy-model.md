# Strategy model

A strategy is a rule that produces slots. A service has any number of them; their slots are merged. To
offer 09:00 to 12:00 and 13:00 to 17:00 (a lunch break), write two RANGE strategies, since one
strategy has one daily window.

## The four types

| Type | Produces | Typical use |
|---|---|---|
| `RANGE` | On every matching day, slots of `slotDuration` every `slotInterval` inside `start.time` to `end.time` | Opening hours, appointments |
| `PERIODICITY` | Same, but only on the Nth weekday(s) of the month (`tendency`) | "First Monday", "last Friday", monthly classes |
| `ONCE` | One slot at fixed instants `start` to `end` | A single event or workshop |
| `DAY_RANGE` | One slot per matching local day, midnight to midnight | Day rentals, full-day passes |

`tendency` values: `FIRST`, `SECOND`, `THIRD`, `FOURTH`, `FIFTH` (only months that have a fifth such
weekday, and most do not), `PENULTIMATE`, `LAST`, `ALL`. Several combine (`['FIRST', 'LAST']`); empty means
`ALL`. With `daysOfWeek: ['MONDAY', 'THURSDAY']` and `tendency: ['FIRST']` you get the first Monday *and*
the first Thursday.

## Bounds: `start` and `end`

Every non-ONCE strategy has `start` and `end`, each with three optional parts that combine:

| Field | On `start` | On `end` |
|---|---|---|
| `dateTime` (UTC instant) | No slot starts before it | No slot starts after it |
| `time` (wall clock, strategy zone) | Daily window opens (default `00:00`) | Daily window closes (default `24:00`) |
| `timeOffset` (ISO duration) | Lead time: no slot starts before `now + offset` | Booking window: no slot starts after `now + offset` |

A zero offset (`PT0S`) means "not set". A slot must fit inside the daily window entirely: with
`09:00` to `11:00` and `slotDuration: PT45M`, `slotInterval: PT45M`, you get 09:00 and 09:45 (10:30 would
end at 11:15).

`overnight: true` moves the window's end to the next local day: `start.time: '22:00'`,
`end.time: '02:00'` gives 22:00, 23:00, 00:00, 01:00. The day filters (`daysOfWeek`, `months`, `tendency`)
apply to the day the window opens on. It must be explicit; see the hard rules in `SKILL.md`.

## Limits

`slotsLimit` (RANGE, PERIODICITY) and `daysLimit` (DAY_RANGE) cap the total a strategy produces, counted
from its first bookable slot, `max(now + lead time, start.dateTime)`, in chronological order. `0`
produces nothing; absent means unlimited. The count never depends on the viewport a client requests.

## Examples

```json
{ "strategyType": "RANGE", "timeZone": "Europe/Warsaw", "slotQuantity": 1,
  "slotDuration": "PT30M", "slotInterval": "PT30M",
  "daysOfWeek": ["MONDAY", "TUESDAY", "WEDNESDAY", "THURSDAY", "FRIDAY"],
  "start": { "time": "09:00", "timeOffset": "PT2H" },
  "end":   { "time": "17:00", "timeOffset": "P60D" } }
```

Weekday appointments every 30 minutes, bookable from two hours to sixty days ahead.

```json
{ "strategyType": "PERIODICITY", "timeZone": "America/New_York", "slotQuantity": 20,
  "slotDuration": "PT2H", "slotInterval": "PT2H",
  "daysOfWeek": ["FRIDAY"], "tendency": ["FIRST", "LAST"],
  "start": { "time": "18:00", "dateTime": "2027-01-01T05:00:00.000Z" },
  "end":   { "time": "20:00" } }
```

A 20-seat evening on the first and last Friday of each month, from 2027.

```json
{ "strategyType": "DAY_RANGE", "timeZone": "Pacific/Auckland", "slotQuantity": 3,
  "daysOfWeek": ["SATURDAY", "SUNDAY"], "start": { "timeOffset": "P1D" }, "end": { "timeOffset": "P90D" },
  "daysLimit": 20 }
```

Three whole-day rentals per weekend day, the next 20 weekend days.

## Types

```ts
// lib/slots/types.ts: backend-neutral shapes. No SDK types cross this line:
// instants are ISO 8601 UTC strings ("2026-11-09T08:00:00.000Z"), durations
// are ISO 8601 durations ("PT45M"), wall-clock times are "HH:mm[:ss]".

export type DayOfWeek =
  | 'SUNDAY'
  | 'MONDAY'
  | 'TUESDAY'
  | 'WEDNESDAY'
  | 'THURSDAY'
  | 'FRIDAY'
  | 'SATURDAY';

export type Month =
  | 'JANUARY'
  | 'FEBRUARY'
  | 'MARCH'
  | 'APRIL'
  | 'MAY'
  | 'JUNE'
  | 'JULY'
  | 'AUGUST'
  | 'SEPTEMBER'
  | 'OCTOBER'
  | 'NOVEMBER'
  | 'DECEMBER';

export type Tendency =
  | 'FIRST'
  | 'SECOND'
  | 'THIRD'
  | 'FOURTH'
  | 'FIFTH'
  | 'PENULTIMATE'
  | 'LAST'
  | 'ALL';

export type StrategyType = 'RANGE' | 'PERIODICITY' | 'ONCE' | 'DAY_RANGE';

/** Index = JavaScript weekday number (0 = Sunday). */
export const DAY_ORDER: readonly DayOfWeek[] = [
  'SUNDAY',
  'MONDAY',
  'TUESDAY',
  'WEDNESDAY',
  'THURSDAY',
  'FRIDAY',
  'SATURDAY',
];

/** Index = month number - 1. */
export const MONTH_ORDER: readonly Month[] = [
  'JANUARY',
  'FEBRUARY',
  'MARCH',
  'APRIL',
  'MAY',
  'JUNE',
  'JULY',
  'AUGUST',
  'SEPTEMBER',
  'OCTOBER',
  'NOVEMBER',
  'DECEMBER',
];

/**
 * One side of a strategy's validity. All three parts are optional and combine:
 * - dateTime:   absolute bound, a UTC instant.
 * - time:       daily wall-clock bound in the strategy's time zone.
 * - timeOffset: bound relative to "now": on `start` the minimum lead time,
 *               on `end` the booking window.
 */
export interface StrategyBound {
  dateTime?: string | null;
  time?: string | null;
  timeOffset?: string | null;
}

interface StrategyBase {
  strategyId: string;
  strategyType: StrategyType;
  /** Seats per slot. */
  slotQuantity: number;
  /** IANA zone the wall-clock fields were authored in. Pin it at create time. */
  timeZone?: string | null;
  labels?: string[] | null;
}

/** Every matching day: slots of `slotDuration`, every `slotInterval`, inside start.time to end.time. */
export interface RangeStrategy extends StrategyBase {
  strategyType: 'RANGE';
  start: StrategyBound;
  end: StrategyBound;
  slotDuration: string;
  slotInterval: string;
  daysOfWeek?: DayOfWeek[] | null;
  months?: Month[] | null;
  /** The daily window ends on the next local day (22:00 to 02:00). Must be explicit. */
  overnight?: boolean | null;
  /** Total slots this strategy produces, counted from its first bookable slot. */
  slotsLimit?: number | null;
}

/** Like RANGE, but only on the Nth weekday(s) of each month: "first and last Friday". */
export interface PeriodicityStrategy extends StrategyBase {
  strategyType: 'PERIODICITY';
  start: StrategyBound;
  end: StrategyBound;
  slotDuration: string;
  slotInterval: string;
  /** Empty or missing = ALL. */
  tendency?: Tendency[] | null;
  daysOfWeek?: DayOfWeek[] | null;
  months?: Month[] | null;
  overnight?: boolean | null;
  slotsLimit?: number | null;
}

/** A single slot at fixed instants. */
export interface OnceStrategy extends StrategyBase {
  strategyType: 'ONCE';
  start: string;
  /** Defaults to start + slotDuration. */
  end?: string | null;
  slotDuration: string;
}

/** One slot per matching local day, midnight to midnight (23h or 25h on DST days). */
export interface DayRangeStrategy extends StrategyBase {
  strategyType: 'DAY_RANGE';
  start: StrategyBound;
  end: StrategyBound;
  daysOfWeek?: DayOfWeek[] | null;
  months?: Month[] | null;
  daysLimit?: number | null;
}

export type SlotStrategy = RangeStrategy | PeriodicityStrategy | OnceStrategy | DayRangeStrategy;

export interface GeneratedSlot {
  slotId: string;
  /** '' for a manually created slot that belongs to no strategy. */
  strategyId: string;
  from: Date;
  to: Date;
  duration: string;
  /** Seats the slot was created with. */
  capacity: number;
  /** Seats still free; availability lowers it. */
  quantity: number;
  labels: string[];
}

export interface GenerationWindow {
  /** Injected clock. Never call `new Date()` inside the engine. */
  now: Date;
  /** Viewport start; defaults to `now`. */
  from?: Date;
  /** Viewport end (inclusive on slot start); defaults to the horizon. */
  to?: Date;
  /** Default zone for strategies without a pinned `timeZone`. */
  timeZone: string;
}

export interface GenerationLimits {
  /** Nothing is generated past now + this many days, whatever the strategy says. */
  maxHorizonDays: number;
  /** Hard stop for one generation; the result reports `truncated`. */
  maxSlots: number;
}

export const DEFAULT_LIMITS: GenerationLimits = { maxHorizonDays: 731, maxSlots: 50_000 };
```

## Validation

Run it on create **and** update, before writing. Codes are stable; map them to HTTP 400 and translated
messages.

| Code | Cause |
|---|---|
| `strategy-timezone-invalid` | `timeZone` is not an IANA zone (`Europe/Warsw`, `CET+1`) |
| `strategy-quantity-invalid` | `slotQuantity` negative or fractional |
| `strategy-duration-invalid` | Not `PnW`/`PnD`/`PTnHnMnS`, or zero. Years and months are refused: their length depends on the date |
| `strategy-time-invalid` | `time` not `HH:mm[:ss]`; `24:00` only as an end |
| `strategy-bounds-invalid` | `start.dateTime >= end.dateTime`; an instant without `Z`/offset; `end.time <= start.time` without `overnight`; window shorter than one slot; lead time >= booking window |
| `strategy-interval-invalid` | `slotInterval < slotDuration`: overlapping slots that share no capacity |
| `strategy-horizon-invalid` | `end.timeOffset` beyond `maxHorizonDays` |
| `strategy-limit-invalid` | `slotsLimit`/`daysLimit` negative or fractional |
| `strategy-end-conflict` | Legacy `end.timePeriod` and `end.timeOffset` both set and different (raised by the normalizer) |

An instant without a zone is refused rather than assumed UTC: `"2026-11-09T09:00:00"` is almost always a
wall-clock time from a form, and treating it as UTC is the bug this whole model exists to prevent.

```ts
// lib/slots/validate.ts: run on every create AND update, before writing.
//
// Errors are `code: detail` strings so an API layer can map the code to a
// status and a translated message without parsing prose.
import { durationToMs } from './duration';
import { isValidTimeZone, timeToSeconds } from './timezone';
import { DEFAULT_LIMITS, type GenerationLimits, type SlotStrategy, type StrategyBound } from './types';

const MS_PER_DAY = 86_400_000;

export function validateStrategy(s: SlotStrategy, limits: GenerationLimits = DEFAULT_LIMITS): void {
  if (s.timeZone != null && !isValidTimeZone(s.timeZone)) {
    fail('strategy-timezone-invalid', `"${s.timeZone}" is not an IANA zone`);
  }
  if (!Number.isInteger(s.slotQuantity) || s.slotQuantity < 0) {
    fail('strategy-quantity-invalid', 'slotQuantity must be a whole number >= 0');
  }

  if (s.strategyType === 'ONCE') {
    const start = instant(s.start, 'start');
    const durationMs = duration(s.slotDuration, 'slotDuration');
    if (durationMs <= 0) fail('strategy-duration-invalid', 'slotDuration must be > 0');
    if (s.end != null && instant(s.end, 'end') <= start) {
      fail('strategy-bounds-invalid', 'ONCE start must be before end');
    }
    return;
  }

  validateBounds(s.start, s.end, limits);

  if (s.strategyType === 'DAY_RANGE') {
    limit(s.daysLimit, 'daysLimit');
    return;
  }

  const durationMs = duration(s.slotDuration, 'slotDuration');
  const intervalMs = duration(s.slotInterval, 'slotInterval');
  if (durationMs <= 0) fail('strategy-duration-invalid', 'slotDuration must be > 0');
  // A shorter interval yields overlapping slots that share no capacity.
  if (intervalMs < durationMs) {
    fail('strategy-interval-invalid', `slotInterval (${s.slotInterval}) must be >= slotDuration (${s.slotDuration})`);
  }
  limit(s.slotsLimit, 'slotsLimit');

  const from = timeToSecondsOrFail(s.start.time ?? '00:00', 'start.time');
  const to = timeToSecondsOrFail(s.end.time ?? '24:00', 'end.time');
  const windowSeconds = s.overnight ? to + 86_400 - from : to - from;
  if (!s.overnight && to <= from) {
    // Never infer "overnight" from end < start: 17:00 to 09:00 is far more often a typo.
    fail('strategy-bounds-invalid', 'end.time must be after start.time (set overnight for windows past midnight)');
  }
  if (s.overnight && (to > from || windowSeconds > 86_400)) {
    fail('strategy-bounds-invalid', 'an overnight window ends on the next day at or before start.time');
  }
  if (windowSeconds * 1000 < durationMs) {
    fail('strategy-bounds-invalid', 'the daily window is shorter than slotDuration, so no slot fits');
  }
}

function validateBounds(start: StrategyBound, end: StrategyBound, limits: GenerationLimits): void {
  const from = start.dateTime != null ? instant(start.dateTime, 'start.dateTime') : undefined;
  const until = end.dateTime != null ? instant(end.dateTime, 'end.dateTime') : undefined;
  if (from !== undefined && until !== undefined && from >= until) {
    fail('strategy-bounds-invalid', 'start.dateTime must be before end.dateTime');
  }
  const lead = start.timeOffset ? duration(start.timeOffset, 'start.timeOffset') : 0;
  const window = end.timeOffset ? duration(end.timeOffset, 'end.timeOffset') : 0;
  if (window > 0 && lead >= window) {
    fail('strategy-bounds-invalid', 'start.timeOffset (lead time) must be shorter than end.timeOffset (booking window)');
  }
  if (window > limits.maxHorizonDays * MS_PER_DAY) {
    fail('strategy-horizon-invalid', `end.timeOffset exceeds the ${limits.maxHorizonDays}-day horizon`);
  }
}

function instant(value: string, field: string): number {
  const ms = Date.parse(value);
  if (Number.isNaN(ms) || !/(Z|[+-]\d{2}:\d{2})$/.test(value)) {
    // An instant without a zone is a wall-clock time pretending to be UTC.
    fail('strategy-bounds-invalid', `${field} must be an ISO instant with Z or an offset`);
  }
  return ms;
}

function duration(value: string, field: string): number {
  try {
    return durationToMs(value);
  } catch {
    return fail('strategy-duration-invalid', `${field} "${value}" is not an ISO 8601 duration`);
  }
}

function timeToSecondsOrFail(value: string, field: string): number {
  try {
    return timeToSeconds(value);
  } catch {
    return fail('strategy-time-invalid', `${field} "${value}" is not HH:mm[:ss]`);
  }
}

function limit(value: number | null | undefined, field: string): void {
  if (value != null && (!Number.isInteger(value) || value < 0)) {
    fail('strategy-limit-invalid', `${field} must be a whole number >= 0`);
  }
}

function fail(code: string, detail: string): never {
  throw new Error(`${code}: ${detail}`);
}
```

Wrap it with the host's schema library for shape checks (enums, required fields) and keep this function
for the rules a schema cannot express.

## Checklist

- [ ] `timeZone` pinned on create from the service/project zone; never re-pinned silently on update
- [ ] `validateStrategy` called on create and update
- [ ] `dateTime` fields sent as UTC instants with `Z` or an offset
- [ ] Lunch breaks and split hours modelled as separate strategies
- [ ] `overnight` exposed in the admin form, not inferred
