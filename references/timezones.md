# Time zones

Three representations, each with one job. Mixing them up is the source of nearly every slot bug.

| Representation | Example | Used for |
|---|---|---|
| UTC instant (ISO, `Z`) | `2026-11-09T08:00:00.000Z` | Storage, comparison, the API value clients send back |
| Wall clock + IANA zone | `09:00` in `Europe/Warsaw` | Authoring: what an operator types and means |
| Local ISO with offset | `2026-11-09T09:00:00+01:00` | Display only (`dateTimeISOFrom`) |

## Rules

1. **Store UTC instants.** Bookings, blocks, manual slots and strategy `dateTime` bounds are instants.
2. **Keep wall-clock times as strings with a zone.** `start.time` / `end.time` are never converted at
   write time; they are interpreted on every generation in the strategy's pinned zone, so 09:00 stays
   09:00 through DST changes.
3. **Pin the zone per strategy.** Use `strategy.timeZone ?? service.timeZone`. The pin is what lets the
   project change zones without moving existing slots.
4. **Never depend on the process zone.** No `new Date('YYYY-MM-DDTHH:mm')` (parsed in the server's
   zone), no `getHours()`/`getDay()`, no `getUTCDay()` on a "zoned" Date from `toZonedTime` (correct only
   when the process runs in UTC). Run the tests under two `TZ` values to catch regressions.
5. **Iterate local dates as `YYYY-MM-DD` strings** with `addDays`, then convert each date + wall time to
   an instant. Never step instants by 24 hours.
6. **Months and weekdays come from the local calendar.** In Auckland (+13:00) 09:00 on Monday 1st is
   20:00 UTC on Sunday 31st. Filtering on UTC fields files it under the wrong day and month.

## DST policy

`zonedToUtc` resolves the two impossible cases the way Temporal's `'compatible'` mode does:

| Case | Example (Europe/Warsaw) | Result |
|---|---|---|
| Gap: wall time does not exist | 2027-03-28 02:30 | Shifted forward by the gap: 03:30 local (01:30Z) |
| Overlap: wall time occurs twice | 2026-10-25 02:30 | The earlier one: 02:30 CEST (00:30Z) |
| End of day | `24:00` | Next local midnight |

Consequences worth knowing:

- A RANGE window 02:00–04:00 on a spring-forward night is one hour long; on a fall-back night it is
  three. Slots are stepped in real time from the window start, so the count follows.
- A DAY_RANGE slot is 23 or 25 hours on transition days. It always runs to the next local midnight, so
  consecutive days tile without gaps or overlap.
- Zones with half-hour transitions (Australia/Lord_Howe) and DST at midnight (historically
  America/Santiago) are handled by the same code: offsets are read from Intl, never assumed to be whole
  hours.

## Helpers

Dependency-free; `Intl.DateTimeFormat` carries the tz database that ships with Node and every modern
runtime. If the host already uses `date-fns-tz` or Luxon you may swap the internals, but keep these
signatures and the policy above, and keep the tests.

```ts
// slots/timezone.ts — wall clock <-> instant conversion on Intl alone.
//
// Every function here gives the same answer whatever the process TZ is. Never
// build a Date from a local string (`new Date('2026-03-29T09:00')` is parsed in
// the *server's* zone) and never read getUTCDay() off a "zoned" Date — both
// tie the output to the machine the code happens to run on.

const MS_PER_MINUTE = 60_000;
const MS_PER_DAY = 86_400_000;

const formatters = new Map<string, Intl.DateTimeFormat>();

function formatterFor(timeZone: string): Intl.DateTimeFormat {
  let formatter = formatters.get(timeZone);
  if (!formatter) {
    // Throws RangeError for an unknown zone, which is what validation relies on.
    formatter = new Intl.DateTimeFormat('en-US', {
      timeZone,
      hourCycle: 'h23',
      year: 'numeric',
      month: '2-digit',
      day: '2-digit',
      hour: '2-digit',
      minute: '2-digit',
      second: '2-digit',
    });
    formatters.set(timeZone, formatter);
  }
  return formatter;
}

export function isValidTimeZone(timeZone: string): boolean {
  try {
    formatterFor(timeZone);
    return true;
  } catch {
    return false;
  }
}

export interface WallClock {
  /** 'YYYY-MM-DD' local calendar date. */
  date: string;
  year: number;
  /** 1–12. */
  month: number;
  day: number;
  hour: number;
  minute: number;
  second: number;
  /** 0 = Sunday. */
  weekday: number;
}

export function wallClock(instant: Date, timeZone: string): WallClock {
  const parts: Record<string, number> = {};
  for (const part of formatterFor(timeZone).formatToParts(instant)) {
    if (part.type !== 'literal') parts[part.type] = Number(part.value);
  }
  const year = parts.year ?? 1970;
  const month = parts.month ?? 1;
  const day = parts.day ?? 1;
  const date = `${String(year).padStart(4, '0')}-${pad(month)}-${pad(day)}`;
  return {
    date,
    year,
    month,
    day,
    hour: parts.hour ?? 0,
    minute: parts.minute ?? 0,
    second: parts.second ?? 0,
    weekday: new Date(Date.UTC(year, month - 1, day)).getUTCDay(),
  };
}

/** Minutes the zone is ahead of UTC at `instant` (Warsaw in July: 120). */
export function offsetMinutes(instant: Date, timeZone: string): number {
  const w = wallClock(instant, timeZone);
  const asUtc = Date.UTC(w.year, w.month - 1, w.day, w.hour, w.minute, w.second);
  const truncated = Math.floor(instant.getTime() / 1000) * 1000;
  return Math.round((asUtc - truncated) / MS_PER_MINUTE);
}

/**
 * The UTC instant at which the clock in `timeZone` reads `date` + `time`.
 *
 * DST policy (same as Temporal's 'compatible' disambiguation):
 * - Gap (the wall time does not exist, 02:30 on a spring-forward night):
 *   shift forward by the gap length — 02:30 becomes 03:30.
 * - Overlap (the wall time happens twice, 02:30 on a fall-back night):
 *   take the earlier instant.
 * `time` may be "24:00", meaning midnight at the end of `date`.
 */
export function zonedToUtc(date: string, time: string, timeZone: string): Date {
  const { year, month, day } = parseDate(date);
  const { hour, minute, second } = parseTime(time);
  const naive = Date.UTC(year, month - 1, day, hour, minute, second);

  // A transition is never closer than a day to another one, so the offsets a
  // day either side are the only two candidates.
  const before = offsetMinutes(new Date(naive - MS_PER_DAY), timeZone);
  const after = offsetMinutes(new Date(naive + MS_PER_DAY), timeZone);

  const valid: number[] = [];
  for (const offset of before === after ? [before] : [before, after]) {
    const candidate = naive - offset * MS_PER_MINUTE;
    if (wallClockMillis(new Date(candidate), timeZone) === naive) valid.push(candidate);
  }
  if (valid.length > 0) return new Date(Math.min(...valid));
  // Gap: read the wall time with the pre-transition offset, which lands after it.
  return new Date(naive - before * MS_PER_MINUTE);
}

/** "2026-11-09T09:00:00+01:00" — the instant as the local clock shows it. */
export function formatWithOffset(instant: Date, timeZone: string): string {
  const w = wallClock(instant, timeZone);
  const offset = offsetMinutes(instant, timeZone);
  const sign = offset < 0 ? '-' : '+';
  const abs = Math.abs(offset);
  return `${w.date}T${pad(w.hour)}:${pad(w.minute)}:${pad(w.second)}${sign}${pad(Math.floor(abs / 60))}:${pad(abs % 60)}`;
}

/** Calendar arithmetic on 'YYYY-MM-DD'; no zone involved. */
export function addDays(date: string, days: number): string {
  const { year, month, day } = parseDate(date);
  return new Date(Date.UTC(year, month - 1, day + days)).toISOString().slice(0, 10);
}

export function weekdayOf(date: string): number {
  const { year, month, day } = parseDate(date);
  return new Date(Date.UTC(year, month - 1, day)).getUTCDay();
}

export function daysInMonth(year: number, month: number): number {
  return new Date(Date.UTC(year, month, 0)).getUTCDate();
}

/** Accepts H, HH:mm, HH:mm:ss, HH:mm:ss.sss; 24:00 only as end of day. */
export function parseTime(time: string): { hour: number; minute: number; second: number } {
  const match = /^(\d{1,2})(?::(\d{2}))?(?::(\d{2}))?(?:\.\d{1,3})?$/.exec(time.trim());
  if (!match) throw new Error(`time-invalid: "${time}"`);
  const hour = Number(match[1]);
  const minute = Number(match[2] ?? 0);
  const second = Number(match[3] ?? 0);
  const endOfDay = hour === 24 && minute === 0 && second === 0;
  if ((hour > 23 && !endOfDay) || minute > 59 || second > 59) {
    throw new Error(`time-invalid: "${time}"`);
  }
  return { hour, minute, second };
}

/** Seconds since local midnight, for comparing wall-clock times. */
export function timeToSeconds(time: string): number {
  const { hour, minute, second } = parseTime(time);
  return hour * 3600 + minute * 60 + second;
}

function parseDate(date: string): { year: number; month: number; day: number } {
  const match = /^(\d{4})-(\d{2})-(\d{2})$/.exec(date);
  if (!match) throw new Error(`date-invalid: "${date}"`);
  return { year: Number(match[1]), month: Number(match[2]), day: Number(match[3]) };
}

function wallClockMillis(instant: Date, timeZone: string): number {
  const w = wallClock(instant, timeZone);
  return Date.UTC(w.year, w.month - 1, w.day, w.hour, w.minute, w.second);
}

function pad(value: number): string {
  return String(value).padStart(2, '0');
}
```

```ts
// slots/duration.ts — ISO 8601 durations as fixed milliseconds.
//
// Days and weeks are exact multiples of 24h here. That is what a slot length or
// a booking window means ("P30D" = 720 hours from now), and it is what slot IDs
// were built on. Years and months are refused: their length depends on the
// date they are applied to, which no caller of this module provides.

const UNIT_MS = { W: 604_800_000, D: 86_400_000, H: 3_600_000, M: 60_000, S: 1_000 } as const;

export function durationToMs(duration: string): number {
  const match =
    /^P(?:(\d+(?:\.\d+)?)W)?(?:(\d+(?:\.\d+)?)D)?(?:T(?:(\d+(?:\.\d+)?)H)?(?:(\d+(?:\.\d+)?)M)?(?:(\d+(?:\.\d+)?)S)?)?$/.exec(
      duration
    );
  if (!match || duration === 'P' || duration.endsWith('T')) {
    throw new Error(`duration-invalid: "${duration}" (use PnW, PnD, PTnHnMnS)`);
  }
  const [, w, d, h, m, s] = match;
  return Math.round(
    Number(w ?? 0) * UNIT_MS.W +
      Number(d ?? 0) * UNIT_MS.D +
      Number(h ?? 0) * UNIT_MS.H +
      Number(m ?? 0) * UNIT_MS.M +
      Number(s ?? 0) * UNIT_MS.S
  );
}
```

## Legacy storage: wall-clock-as-UTC

A common earlier convention stored `dateTime` bounds as the *wall clock* in UTC fields: `09:00Z` meaning
09:00 in the project's zone. If your data has it, mark new documents with `storageVersion: 'utc'` and
upgrade old ones at read time with the normalizer below. It pins the zone first, because that zone is what
defined the wall clock.

```ts
// slots/normalize.ts — read-time upgrade of stored strategies. Idempotent.
//
// Only needed if strategies were ever stored in the legacy convention, where a
// `dateTime` held the *wall clock* in UTC fields ("09:00Z" meaning 09:00 in the
// project's zone). New data never needs it; it is safe to run on every read.
import { zonedToUtc } from './timezone';

type Raw = Record<string, unknown>;

export function normalizeStrategy(
  raw: Raw,
  { defaultTimeZone }: { defaultTimeZone: string }
): { strategy: Raw; changed: boolean } {
  const out: Raw = { ...raw };
  let changed = false;

  // Pin the authoring zone. Without the pin, changing the project's zone
  // silently moves every existing slot.
  if (typeof out.timeZone !== 'string' || out.timeZone === '') {
    out.timeZone = defaultTimeZone;
    changed = true;
  }
  const timeZone = out.timeZone as string;

  // Legacy alias: end.timePeriod -> end.timeOffset.
  const end = out.end as Raw | undefined;
  if (end && typeof end === 'object' && end.timePeriod !== undefined) {
    if (end.timeOffset !== undefined && end.timeOffset !== end.timePeriod) {
      throw new Error(`strategy-end-conflict: end.timePeriod (${String(end.timePeriod)}) and end.timeOffset (${String(end.timeOffset)}) differ`);
    }
    const { timePeriod, ...rest } = end;
    out.end = { ...rest, timeOffset: end.timeOffset ?? timePeriod };
    changed = true;
  }

  if (out.storageVersion !== 'utc') {
    // The stored UTC fields ARE the wall clock; re-read them in the pinned zone.
    const toUtc = (value: unknown): string | undefined => {
      if (typeof value !== 'string' || Number.isNaN(Date.parse(value))) return undefined;
      const iso = new Date(value).toISOString();
      return zonedToUtc(iso.slice(0, 10), iso.slice(11, 19), timeZone).toISOString();
    };
    if (out.strategyType === 'ONCE') {
      out.start = toUtc(out.start) ?? out.start;
      out.end = toUtc(out.end) ?? out.end;
    } else {
      for (const key of ['start', 'end'] as const) {
        const bound = out[key] as Raw | undefined;
        const converted = bound && toUtc(bound.dateTime);
        if (bound && converted) out[key] = { ...bound, dateTime: converted };
      }
    }
    out.storageVersion = 'utc';
    changed = true;
  }

  return { strategy: out, changed };
}
```

To persist the upgrade, run it over every strategy once (batch, idempotent, with `dryRun`) and write back
the documents where `changed` is true. Until then, any admin screen that reads raw strategy documents
shows the legacy values; read through the normalizer there too.

Older shapes (root-level `timeFrom`/`timeTo`, `startDateTime`, `discontinueStrategy`,
`periodicityTendency`, ONCE `slotDateTime`) can be folded into `start`/`end`/`tendency` in the same
function if your data predates this model.

## Checklist

- [ ] Every stored instant is UTC; every wall-clock time is a string plus a zone
- [ ] No `new Date(localString)`, `getHours()`, `getDay()` or `getUTC*()` on zoned Dates in slot code
- [ ] Tests run under `TZ=UTC` and at least one non-UTC zone
- [ ] Output carries both the UTC value and the local-offset value
- [ ] Legacy documents normalized at read time until migrated
