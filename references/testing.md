# Testing

The engine is pure, so it is tested without a database, a clock or a mock. The suite below covers every
defect listed in [provenance.md](provenance.md) and the behaviours the other references promise. Put it
next to the engine files and keep it passing.

## Run it under several process zones

The point of half these tests is that the answer must not change with the machine. Run the suite under at
least `UTC` and one zone with DST; CI should do all four:

```bash
for tz in UTC Europe/Warsaw Pacific/Auckland America/Los_Angeles; do
  TZ=$tz npx vitest run slots/engine.test.ts   # or: TZ=$tz node --test slots/engine.test.ts
done
```

The file uses `node:test` and `node:assert` so it runs with no dependency on Node 22.18+ (`node --test`
strips the types). Under vitest, replace the `node:test` import with `import { describe, it } from
'vitest'`; nothing else changes. If your imports need extensions (`./generators.js`), add them.

## What each group pins

| Group | Pins |
|---|---|
| PERIODICITY | A week view returns all seven days; `slotsLimit` and the booking window cut chronologically; FIRST/LAST/FIFTH; local months at +13:00 |
| RANGE across DST | The 23-hour day survives "now = 23:30 the evening before"; the 25-hour day yields once; weekdays read locally in Auckland and Honolulu |
| DAY_RANGE | Midnight-to-midnight slots, 24/25/24 hours, tiling without gaps |
| Limits and viewports | `slotsLimit` independent of the viewport; lead time and booking window; horizon cap and `truncated`; one bad strategy isolated |
| Time zones | Pinned zone wins; gap forward, overlap earlier, Lord Howe's 30-minute shift, `24:00`; offsets including +05:30 |
| Overnight windows | 22:00–02:00, including yesterday's window continuing after midnight |
| Availability | Seats subtracted, blocks remove, resources cap; working hours reject a slot past midnight; booking lookup errors |
| Identity, validation, normalization | Slot ID format; authoring mistakes refused; legacy upgrade correct and idempotent |

## The suite

```ts
// slots/engine.test.ts — node:test, no dependencies. Run it under several
// process time zones; the results must not change:
//   for tz in UTC Europe/Warsaw Pacific/Auckland America/Los_Angeles; do TZ=$tz node --test; done
import assert from 'node:assert/strict';
import { createHash } from 'node:crypto';
import { describe, it } from 'node:test';

import { buildAvailability, isWithinWorkingHours, resolveSlotsForBooking, toSlotOutput } from './availability';
import { generateSlots } from './generators';
import { normalizeStrategy } from './normalize';
import { formatWithOffset, zonedToUtc } from './timezone';
import type { DayRangeStrategy, GeneratedSlot, PeriodicityStrategy, RangeStrategy } from './types';
import { validateStrategy } from './validate';

const WAW = 'Europe/Warsaw';
const iso = (slots: { from: Date }[]): string[] => slots.map(s => s.from.toISOString());

const range = (over: Partial<RangeStrategy> = {}): RangeStrategy => ({
  strategyId: 'r1',
  strategyType: 'RANGE',
  slotQuantity: 2,
  slotDuration: 'PT1H',
  slotInterval: 'PT1H',
  start: { time: '09:00' },
  end: { time: '10:00' },
  ...over,
});

const periodicity = (over: Partial<PeriodicityStrategy> = {}): PeriodicityStrategy => ({
  strategyId: 'p1',
  strategyType: 'PERIODICITY',
  slotQuantity: 1,
  slotDuration: 'PT1H',
  slotInterval: 'PT1H',
  start: { time: '09:00' },
  end: { time: '10:00' },
  ...over,
});

const dayRange = (over: Partial<DayRangeStrategy> = {}): DayRangeStrategy => ({
  strategyId: 'd1',
  strategyType: 'DAY_RANGE',
  slotQuantity: 1,
  start: {},
  end: {},
  ...over,
});

describe('PERIODICITY', () => {
  it('returns every weekday of a one-week viewport, not just the first weekday walked', () => {
    const { slots } = generateSlots([periodicity()], {
      now: new Date('2026-11-01T00:00:00Z'),
      from: new Date('2026-11-09T00:00:00Z'),
      to: new Date('2026-11-15T23:59:59Z'),
      timeZone: WAW,
    });
    assert.equal(slots.length, 7);
  });

  it('counts slotsLimit chronologically', () => {
    const { slots } = generateSlots([periodicity({ slotsLimit: 3 })], {
      now: new Date('2026-11-01T00:00:00Z'),
      timeZone: WAW,
    });
    assert.deepEqual(iso(slots), ['2026-11-01T08:00:00.000Z', '2026-11-02T08:00:00.000Z', '2026-11-03T08:00:00.000Z']);
  });

  it('stops at the booking window without dropping earlier weekdays', () => {
    const { slots } = generateSlots([periodicity({ end: { time: '10:00', timeOffset: 'P3D' } })], {
      now: new Date('2026-11-01T00:00:00Z'),
      timeZone: WAW,
    });
    assert.equal(slots.length, 3);
  });

  it('picks FIRST and LAST Friday, and FIFTH only where it exists', () => {
    const window = { now: new Date('2026-01-01T00:00:00Z'), to: new Date('2026-03-31T23:00:00Z'), timeZone: WAW };
    const firstLast = generateSlots([periodicity({ daysOfWeek: ['FRIDAY'], tendency: ['FIRST', 'LAST'] })], window);
    assert.deepEqual(
      firstLast.slots.map(s => formatWithOffset(s.from, WAW).slice(0, 10)),
      ['2026-01-02', '2026-01-30', '2026-02-06', '2026-02-27', '2026-03-06', '2026-03-27']
    );
    const fifth = generateSlots([periodicity({ daysOfWeek: ['FRIDAY'], tendency: ['FIFTH'] })], window);
    assert.deepEqual(fifth.slots.map(s => formatWithOffset(s.from, WAW).slice(0, 10)), ['2026-01-30']);
  });

  it('files dates under the local month in a +13:00 zone', () => {
    const { slots } = generateSlots(
      [periodicity({ daysOfWeek: ['SUNDAY'], tendency: ['FIRST'], timeZone: 'Pacific/Auckland' })],
      { now: new Date('2026-10-30T00:00:00Z'), to: new Date('2026-11-30T00:00:00Z'), timeZone: WAW }
    );
    assert.deepEqual(slots.map(s => formatWithOffset(s.from, 'Pacific/Auckland')), ['2026-11-01T09:00:00+13:00']);
  });
});

describe('RANGE across DST', () => {
  it('keeps the 23-hour spring-forward day when "now" is late the evening before', () => {
    const { slots } = generateSlots([range()], {
      now: new Date('2027-03-27T22:30:00Z'), // 23:30 local
      to: new Date('2027-03-31T00:00:00Z'),
      timeZone: WAW,
    });
    assert.deepEqual(iso(slots), ['2027-03-28T07:00:00.000Z', '2027-03-29T07:00:00.000Z', '2027-03-30T07:00:00.000Z']);
  });

  it('yields the 25-hour fall-back day once, so limits count days not duplicates', () => {
    const { slots } = generateSlots([range({ slotsLimit: 2 })], {
      now: new Date('2026-10-24T22:30:00Z'), // 00:30 local on the 25-hour day
      timeZone: WAW,
    });
    assert.deepEqual(iso(slots), ['2026-10-25T08:00:00.000Z', '2026-10-26T08:00:00.000Z']);
  });

  it('reads weekdays in the local calendar far from UTC', () => {
    const monday = (timeZone: string) =>
      generateSlots([range({ daysOfWeek: ['MONDAY'], timeZone })], {
        now: new Date('2026-11-01T00:00:00Z'),
        to: new Date('2026-11-12T00:00:00Z'),
        timeZone: WAW,
      }).slots.map(s => formatWithOffset(s.from, timeZone));
    assert.deepEqual(monday('Pacific/Auckland'), ['2026-11-02T09:00:00+13:00', '2026-11-09T09:00:00+13:00']);
    assert.deepEqual(monday('Pacific/Honolulu'), ['2026-11-02T09:00:00-10:00', '2026-11-09T09:00:00-10:00']);
  });
});

describe('DAY_RANGE', () => {
  it('runs midnight to the next local midnight, 23h and 25h on DST days', () => {
    const { slots } = generateSlots([dayRange()], {
      now: new Date('2026-10-23T10:00:00Z'),
      to: new Date('2026-10-26T00:00:00Z'),
      timeZone: WAW,
    });
    const hours = slots.map(s => (s.to.getTime() - s.from.getTime()) / 3_600_000);
    assert.deepEqual(hours, [24, 25, 24]);
    for (let i = 1; i < slots.length; i++) assert.equal(slots[i]?.from.getTime(), slots[i - 1]?.to.getTime());
  });
});

describe('limits and viewports', () => {
  it('counts slotsLimit from the first bookable slot, not from the viewport', () => {
    const now = new Date('2026-11-01T00:00:00Z');
    const later = generateSlots([range({ slotsLimit: 2 })], { now, from: new Date('2026-11-10T00:00:00Z'), timeZone: WAW });
    assert.deepEqual(later.slots, []);
    const all = generateSlots([range({ slotsLimit: 2 })], { now, timeZone: WAW });
    assert.deepEqual(iso(all.slots), ['2026-11-01T08:00:00.000Z', '2026-11-02T08:00:00.000Z']);
  });

  it('honours lead time and booking window', () => {
    const { slots } = generateSlots([range({ start: { time: '09:00', timeOffset: 'P1D' }, end: { time: '10:00', timeOffset: 'P3D' } })], {
      now: new Date('2026-11-01T07:00:00Z'),
      timeZone: WAW,
    });
    assert.deepEqual(iso(slots), ['2026-11-02T08:00:00.000Z', '2026-11-03T08:00:00.000Z']);
  });

  it('never generates past the horizon, and flags a truncated result', () => {
    const now = new Date('2026-11-01T00:00:00Z');
    const capped = generateSlots([range()], { now, timeZone: WAW }, { limits: { maxHorizonDays: 10, maxSlots: 1000 } });
    assert.equal(capped.slots.length, 10);
    assert.equal(capped.truncated, false);
    const truncated = generateSlots([range()], { now, timeZone: WAW }, { limits: { maxHorizonDays: 10, maxSlots: 4 } });
    assert.equal(truncated.slots.length, 4);
    assert.equal(truncated.truncated, true);
  });

  it('isolates a malformed strategy', () => {
    const result = generateSlots([range({ strategyId: 'bad', slotDuration: 'one hour' }), range()], {
      now: new Date('2026-11-01T00:00:00Z'),
      to: new Date('2026-11-03T00:00:00Z'),
      timeZone: WAW,
    });
    assert.equal(result.slots.length, 2);
    assert.equal(result.failed[0]?.strategyId, 'bad');
  });
});

describe('time zones', () => {
  it('uses the pinned strategy zone over the default', () => {
    const { slots } = generateSlots([range({ timeZone: 'America/New_York' })], {
      now: new Date('2026-11-02T00:00:00Z'),
      to: new Date('2026-11-03T00:00:00Z'),
      timeZone: WAW,
    });
    assert.deepEqual(iso(slots), ['2026-11-02T14:00:00.000Z']);
  });

  it('resolves gaps forward and overlaps to the earlier instant', () => {
    assert.equal(zonedToUtc('2027-03-28', '02:30', WAW).toISOString(), '2027-03-28T01:30:00.000Z');
    assert.equal(zonedToUtc('2026-10-25', '02:30', WAW).toISOString(), '2026-10-25T00:30:00.000Z');
    // Lord Howe moves by 30 minutes: 02:15 does not exist on 2026-10-04.
    assert.equal(zonedToUtc('2026-10-04', '02:15', 'Australia/Lord_Howe').toISOString(), '2026-10-03T15:45:00.000Z');
    assert.equal(zonedToUtc('2026-11-09', '24:00', WAW).toISOString(), '2026-11-09T23:00:00.000Z');
  });

  it('formats the local offset', () => {
    assert.equal(formatWithOffset(new Date('2026-11-09T08:00:00Z'), WAW), '2026-11-09T09:00:00+01:00');
    assert.equal(formatWithOffset(new Date('2026-11-09T08:00:00Z'), 'Asia/Kolkata'), '2026-11-09T13:30:00+05:30');
  });
});

describe('overnight windows', () => {
  it('runs 22:00–02:00 and continues yesterday’s window after midnight', () => {
    const { slots } = generateSlots(
      [range({ overnight: true, start: { time: '22:00' }, end: { time: '02:00' } })],
      { now: new Date('2026-11-09T23:30:00Z'), to: new Date('2026-11-10T23:59:00Z'), timeZone: WAW } // 00:30 local
    );
    assert.deepEqual(
      slots.map(s => formatWithOffset(s.from, WAW)),
      ['2026-11-10T01:00:00+01:00', '2026-11-10T22:00:00+01:00', '2026-11-10T23:00:00+01:00', '2026-11-11T00:00:00+01:00']
    );
  });
});

describe('availability', () => {
  const window = { now: new Date('2026-11-09T00:00:00Z'), to: new Date('2026-11-09T23:00:00Z'), timeZone: WAW };
  const generated = generateSlots([range({ start: { time: '09:00' }, end: { time: '12:00' } })], window).slots;
  const at = (h: number) => new Date(`2026-11-09T${String(h).padStart(2, '0')}:00:00Z`);

  it('subtracts seats, removes blocked slots, caps by resources', () => {
    const map = buildAvailability({
      generated,
      busy: [
        { kind: 'BOOKING', from: at(8), to: at(9), seats: 2 },
        { kind: 'BLOCK', from: at(9), to: at(10) },
      ],
      resources: [{ resourceId: 'court', capacity: 1, bookings: [{ from: at(10), to: at(11) }] }],
      timeZone: WAW,
    });
    const out = toSlotOutput(map.values(), WAW);
    assert.deepEqual(
      out.map(s => [s.dateTimeISOFrom, s.quantity]),
      [
        ['2026-11-09T09:00:00+01:00', 0],
        ['2026-11-09T11:00:00+01:00', 0],
      ]
    );
  });

  it('rejects a slot that ends after midnight against 08:00–17:00 hours', () => {
    const late = { from: new Date('2026-11-09T22:00:00Z'), to: new Date('2026-11-09T23:00:00Z') }; // 23:00–00:00 local
    assert.equal(isWithinWorkingHours(late, WAW, { MONDAY: { timeFrom: '08:00', timeTo: '17:00' } }), false);
    assert.equal(isWithinWorkingHours(late, WAW, { MONDAY: { timeFrom: '08:00' } }), true);
  });

  it('resolves booking IDs against one map and reports why one fails', () => {
    const map = buildAvailability({ generated, busy: [], timeZone: WAW });
    const first = generated[0] as GeneratedSlot;
    assert.equal(resolveSlotsForBooking(map, [first.slotId], 2).length, 1);
    assert.throws(() => resolveSlotsForBooking(map, [first.slotId], 3), /slot-has-not-enough-capacity/);
    assert.throws(() => resolveSlotsForBooking(map, ['nope'], 1), /slot-no-longer-available/);
  });
});

describe('identity, validation, normalization', () => {
  it('keeps the slot ID format md5(utcStartISO;duration;strategyId)', () => {
    const { slots } = generateSlots([range()], { now: new Date('2026-11-09T00:00:00Z'), to: new Date('2026-11-09T12:00:00Z'), timeZone: WAW });
    const expected = createHash('md5').update('2026-11-09T08:00:00.000Z;PT1H;r1').digest('hex');
    assert.equal(slots[0]?.slotId, expected);
  });

  it('rejects the common authoring mistakes', () => {
    assert.throws(() => validateStrategy(range({ start: { time: '17:00' }, end: { time: '09:00' } })), /strategy-bounds-invalid/);
    assert.throws(() => validateStrategy(range({ slotInterval: 'PT30M' })), /strategy-interval-invalid/);
    assert.throws(() => validateStrategy(range({ timeZone: 'Europe/Warsw' })), /strategy-timezone-invalid/);
    assert.throws(() => validateStrategy(range({ end: { time: '10:00', timeOffset: 'P5000D' } })), /strategy-horizon-invalid/);
    assert.throws(() => validateStrategy(range({ start: { dateTime: '2026-11-09T09:00:00', time: '09:00' } })), /strategy-bounds-invalid/);
    assert.doesNotThrow(() => validateStrategy(range({ overnight: true, start: { time: '22:00' }, end: { time: '02:00' } })));
  });

  it('upgrades legacy wall-clock-as-UTC strategies once', () => {
    const legacy = { strategyType: 'RANGE', start: { dateTime: '2026-07-01T09:00:00.000Z', time: '09:00' }, end: { timePeriod: 'P30D' } };
    const first = normalizeStrategy(legacy, { defaultTimeZone: WAW });
    assert.deepEqual(first.strategy.start, { dateTime: '2026-07-01T07:00:00.000Z', time: '09:00' });
    assert.deepEqual(first.strategy.end, { timeOffset: 'P30D' });
    const second = normalizeStrategy(first.strategy, { defaultTimeZone: 'UTC' });
    assert.equal(second.changed, false);
    assert.deepEqual(second.strategy, first.strategy);
  });
});
```

## Porting an existing engine: a differential check

Before switching, run the old and new generators side by side over an ordinary window (no DST change,
no viewport cut mid-month) and compare `slotId + from + to` sets. They must be identical — that proves
existing slot IDs and bookings still resolve. Then run both over the DST and week-view cases above: the
differences you see are the fixes.

```ts
const window = { now: new Date('2026-11-02T06:00:00Z'), to: new Date('2026-12-20T00:00:00Z'), timeZone: 'Europe/Warsaw' };
const key = (s: { slotId: string; from: Date; to: Date }) => s.slotId + s.from.toISOString() + s.to.toISOString();
const next = generateSlots([strategy], window).slots.map(key).sort();
const prev = oldGenerator(strategy, window).map(key).sort(); // adapt to the old signature
assert.deepEqual(next, prev);
```

## Checklist

- [ ] Suite passes under `UTC` and a DST zone (ideally all four)
- [ ] Differential check identical on an ordinary window before cut-over
- [ ] New strategy features land with a test in the matching group
