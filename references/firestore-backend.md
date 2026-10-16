# Firestore backend

`SlotStore` on the Firebase Admin SDK. All access goes through the server; clients get no direct
rules-based access to these collections (absence of a rule is the policy: default deny).

## Collections

| Path | Fields the engine reads |
|---|---|
| `services/{serviceId}` | `projectId`, `timeZone`, `hostIds[]`, `locationId`, `resourceIds[]` |
| `services/{serviceId}/strategies/{strategyId}` | Strategy fields; `start.dateTime`/`end.dateTime` (and ONCE `start`/`end`) as `Timestamp`; `timeZone`, `storageVersion: 'utc'`, `projectId` |
| `services/{serviceId}/slots/{slotId}` | Manual slots: `from`, `to` (`Timestamp`), `duration`, `quantity`, `labels[]` |
| `busyBlocks/{id}` | `projectId`, `ownerType`, `ownerId`, `from`, `to`, `source`, `externalId` |
| `bookings/{id}` | `projectId`, `serviceId`, `status`, `from`, `to` (span), `seats`, `slotIds[]`, `hostIds[]`, `resourceIds[]`, `slots[]` (copies) |
| `resources/{id}` | `capacity` |
| `locations/{id}` | `workingHours` |

`busyBlocks` is one top-level collection rather than an `exceptions` subcollection per owner, so one
`ownerId in [...]` query covers the service, its location, its hosts and its resources.

## Composite indexes

| Collection | Fields |
|---|---|
| `bookings` | `serviceId` ASC, `from` ASC |
| `bookings` | `hostIds` ARRAY_CONTAINS, `from` ASC |
| `bookings` | `resourceIds` ARRAY_CONTAINS, `from` ASC |
| `busyBlocks` | `ownerId` ASC, `from` ASC |

The transaction check reads `serviceId` and `from`, the first index above.

## Range queries

Firestore here allows range filters on one field per query, so "overlaps [from, to)" is approximated on
`from`: starts before `range.to` and no earlier than `range.from - MAX_INTERVAL_DAYS`, then filtered
exactly on `to` in memory. **`MAX_INTERVAL_DAYS` must be at least the longest booking or block you
allow**: a 30-day block that started 20 days ago is otherwise never read. Enforce that maximum when
writing blocks, or split longer ones.

## Limits that bite

- `in` and `array-contains-any` take at most 30 values; the store chunks host and owner lists.
- `undefined` is rejected on write; `toStored` drops it.
- Equality filters skip documents missing the field: write `status` on every booking at create.

## Caching and listeners

Reading strategies, blocks and bookings on every listing is the simple, correct default. If you add
`onSnapshot` listeners per service as a cache, bound them: cap the number of live entries (LRU) **and**
give each an absolute age limit. An idle-timeout alone never fires under steady traffic, because every hit
resets it, and listeners plus per-entry timers accumulate until the process runs out of memory. Never
cache inside the booking transaction.

## The store

```ts
// lib/slots/firestore-store.ts: SlotStore on the Firebase Admin SDK.
import {
  FieldValue,
  Timestamp,
  type DocumentData,
  type Firestore,
  type Query,
  type Transaction,
} from 'firebase-admin/firestore';

import type { BusyInterval, Interval, ResourceCap, WorkingHours } from './availability';
import { normalizeStrategy } from './normalize';
import type { BookableService, SlotStore } from './service';
import type { GeneratedSlot, SlotStrategy } from './types';
import { validateStrategy } from './validate';

/** Must be >= the longest booking or block you allow; see "Range queries" in firestore-backend.md. */
const MAX_INTERVAL_DAYS = 14;
const MS_PER_DAY = 86_400_000;
/** `in` and `array-contains-any` accept at most 30 values. */
const IN_LIMIT = 30;
const INACTIVE = new Set(['CANCELED', 'REJECTED']);

export function createFirestoreSlotStore(db: Firestore): SlotStore {
  return {
    async listStrategies(serviceId) {
      const service = await db.collection('services').doc(serviceId).get();
      const defaultTimeZone = (service.get('timeZone') as string | undefined) ?? 'UTC';
      const snapshot = await db.collection('services').doc(serviceId).collection('strategies').get();
      return snapshot.docs.map(doc => {
        // Timestamps -> ISO strings first; nothing past this line knows Firestore.
        const { strategy } = normalizeStrategy(toPlain(doc.data()) as Record<string, unknown>, { defaultTimeZone });
        return { ...strategy, strategyId: doc.id } as unknown as SlotStrategy;
      });
    },

    async listManualSlots(serviceId) {
      const snapshot = await db.collection('services').doc(serviceId).collection('slots').get();
      return snapshot.docs.map(doc => {
        const data = doc.data();
        const quantity = typeof data.quantity === 'number' ? data.quantity : 1;
        return {
          slotId: doc.id,
          strategyId: '',
          from: (data.from as Timestamp).toDate(),
          to: (data.to as Timestamp).toDate(),
          duration: data.duration as string,
          capacity: quantity,
          quantity,
          labels: (data.labels as string[] | undefined) ?? [],
        } satisfies GeneratedSlot;
      });
    },

    async listBusy(service: BookableService, range: Interval) {
      const busy: BusyInterval[] = [];

      const own = await overlapping(db.collection('bookings').where('serviceId', '==', service.serviceId), range);
      for (const data of own) {
        if (INACTIVE.has(data.status as string)) continue;
        busy.push({ kind: 'BOOKING', ...interval(data), seats: data.seats as number | undefined });
      }

      for (const hostIds of chunk(service.hostIds, IN_LIMIT)) {
        const hostBookings = await overlapping(
          db.collection('bookings').where('hostIds', 'array-contains-any', hostIds),
          range
        );
        for (const data of hostBookings) {
          if (INACTIVE.has(data.status as string)) continue;
          // Same service = already a BOOKING above; another service = the host is busy.
          if (data.serviceId === service.serviceId) continue;
          busy.push({ kind: 'BLOCK', ...interval(data) });
        }
      }

      const owners = [
        service.serviceId,
        ...service.hostIds,
        ...service.resourceIds,
        ...(service.locationId ? [service.locationId] : []),
      ];
      for (const ownerIds of chunk(owners, IN_LIMIT)) {
        const blocks = await overlapping(db.collection('busyBlocks').where('ownerId', 'in', ownerIds), range);
        for (const data of blocks) busy.push({ kind: 'BLOCK', ...interval(data) });
      }
      return busy;
    },

    async listResources(resourceIds, range) {
      const caps: ResourceCap[] = [];
      for (const resourceId of resourceIds) {
        const [resource, bookings] = await Promise.all([
          db.collection('resources').doc(resourceId).get(),
          overlapping(db.collection('bookings').where('resourceIds', 'array-contains', resourceId), range),
        ]);
        // A deleted resource caps nothing; skipping beats throwing for every slot.
        if (!resource.exists) continue;
        caps.push({
          resourceId,
          capacity: (resource.get('capacity') as number | undefined) ?? 0,
          bookings: bookings.filter(data => !INACTIVE.has(data.status as string)).map(interval),
        });
      }
      return caps;
    },

    async getWorkingHours(locationId) {
      const location = await db.collection('locations').doc(locationId).get();
      return (location.get('workingHours') as WorkingHours | undefined) ?? null;
    },
  };
}

/**
 * Create or replace a strategy. Validates, pins the zone, marks the storage
 * convention, and converts ISO instants to Timestamps on the way in.
 */
export async function saveStrategy(
  db: Firestore,
  serviceId: string,
  strategy: SlotStrategy,
  projectId: string
): Promise<void> {
  validateStrategy(strategy);
  const service = await db.collection('services').doc(serviceId).get();
  // The tenant comes from the stored service, never from the request body.
  if (!service.exists || service.get('projectId') !== projectId) throw new Error('service-not-found');
  const { strategyId, ...fields } = strategy;
  await db
    .collection('services')
    .doc(serviceId)
    .collection('strategies')
    .doc(strategyId)
    .set({
      ...(toStored(fields) as Record<string, unknown>),
      projectId,
      timeZone: strategy.timeZone ?? (service.get('timeZone') as string),
      storageVersion: 'utc',
      updatedAt: FieldValue.serverTimestamp(),
    });
}

/**
 * The capacity re-check, inside the booking transaction. Generation ran before
 * the transaction and may be seconds stale; two guests can both see the last
 * seat. Counting seats of this service's bookings that overlap each slot,
 * inside the transaction, closes that race: Firestore retries the transaction
 * when a booking it read changes, and a newly added booking makes the retry
 * see it. Overlap, not slot ID, the same rule as the listing: two strategies
 * covering one hour share its capacity.
 */
export async function assertCapacityInTransaction(
  tx: Transaction,
  db: Firestore,
  serviceId: string,
  slots: Pick<GeneratedSlot, 'slotId' | 'capacity' | 'from' | 'to'>[],
  seats: number
): Promise<void> {
  for (const slot of slots) {
    const candidates = await tx.get(
      db
        .collection('bookings')
        .where('serviceId', '==', serviceId)
        .where('from', '>=', Timestamp.fromMillis(slot.from.getTime() - MAX_INTERVAL_DAYS * MS_PER_DAY))
        .where('from', '<', Timestamp.fromDate(slot.to))
    );
    const taken = candidates.docs
      .map(doc => doc.data())
      .filter(data => !INACTIVE.has(data.status as string))
      .filter(data => (data.to as Timestamp).toMillis() > slot.from.getTime())
      .reduce((sum, data) => sum + (typeof data.seats === 'number' ? data.seats : 1), 0);
    // Seats, not documents: one booking of four people takes four.
    if (taken + seats > slot.capacity) throw new Error(`slot-has-not-enough-capacity: ${slot.slotId}`);
  }
}

/** ISO instants -> Timestamps; drops undefined, which Firestore rejects. */
function toStored(value: unknown): unknown {
  if (typeof value === 'string' && /^\d{4}-\d{2}-\d{2}T.*(Z|[+-]\d{2}:\d{2})$/.test(value)) {
    return Timestamp.fromDate(new Date(value));
  }
  if (Array.isArray(value)) return value.map(toStored);
  if (value && typeof value === 'object') {
    return Object.fromEntries(
      Object.entries(value)
        .filter(([, inner]) => inner !== undefined)
        .map(([key, inner]) => [key, toStored(inner)])
    );
  }
  return value;
}

/**
 * Firestore allows range filters on one field per query here, so "overlaps
 * [from, to)" is approximated on `from` alone: anything starting before
 * range.to and no earlier than range.from - MAX_INTERVAL_DAYS, then filtered
 * exactly in memory. Index: (<equality field>, from ASC).
 */
async function overlapping(
  query: Query,
  range: Interval
): Promise<DocumentData[]> {
  const snapshot = await query
    .where('from', '>=', Timestamp.fromMillis(range.from.getTime() - MAX_INTERVAL_DAYS * MS_PER_DAY))
    .where('from', '<', Timestamp.fromDate(range.to))
    .get();
  return snapshot.docs
    .map(doc => doc.data())
    .filter(data => (data.to as Timestamp).toMillis() > range.from.getTime());
}

function interval(data: DocumentData): Interval {
  return { from: (data.from as Timestamp).toDate(), to: (data.to as Timestamp).toDate() };
}

/** Deep-convert Timestamps to ISO strings. */
function toPlain(value: unknown): unknown {
  if (value instanceof Timestamp) return value.toDate().toISOString();
  if (Array.isArray(value)) return value.map(toPlain);
  if (value && typeof value === 'object') {
    return Object.fromEntries(Object.entries(value).map(([key, inner]) => [key, toPlain(inner)]));
  }
  return value;
}

function chunk<T>(items: T[], size: number): T[][] {
  const chunks: T[][] = [];
  for (let i = 0; i < items.length; i += size) chunks.push(items.slice(i, i + size));
  return chunks;
}
```

## Booking write, end to end

```ts
const slots = await prepareBookingSlots(store, service, slotIds, seats, new Date());
await db.runTransaction(async tx => {
  await assertCapacityInTransaction(tx, db, service.serviceId, slots, seats); // reads first
  tx.set(db.collection('bookings').doc(), {
    projectId, serviceId: service.serviceId, status: 'PENDING', seats,
    slotIds: slots.map(s => s.slotId),
    hostIds: service.hostIds, resourceIds: service.resourceIds,
    from: Timestamp.fromDate(slots[0]!.from), to: Timestamp.fromDate(slots.at(-1)!.to),
    slots: slots.map(s => ({ slotId: s.slotId, strategyId: s.strategyId, duration: s.duration,
      from: Timestamp.fromDate(s.from), to: Timestamp.fromDate(s.to) })),
    createdAt: FieldValue.serverTimestamp(),
  });
});
```

Firestore transactions require every read before the first write; the capacity check is all reads.

## Checklist

- [ ] Composite indexes deployed with the queries
- [ ] `MAX_INTERVAL_DAYS` >= longest booking/block, enforced on write
- [ ] Strategy writes go through `saveStrategy` (validation, zone pin, Timestamps)
- [ ] No client access to these collections in security rules
- [ ] Capacity re-checked in `runTransaction`
