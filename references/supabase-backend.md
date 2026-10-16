# Supabase / Postgres backend

`SlotStore` on supabase-js, with the schema, indexes, RLS and the capacity function.

## Posture

RLS is the primary control in Postgres, unlike Firestore's Admin SDK. Staff manage configuration through
RLS policies scoped by project membership. **Public slot listings and bookings go through the API**, which
calls the store with a server-side client after its own checks; the anon role gets nothing on these
tables. If the API uses the service-role key, it bypasses RLS entirely: resolve the service and its
`project_id` from the database before every call, never from the request body.

## Mapping from the neutral model

| Neutral | Postgres |
|---|---|
| UTC instant (ISO string) | `timestamptz` (supabase-js returns ISO strings) |
| Strategy | `slot_strategies` row: `strategy_type`, `time_zone`, `storage_version` columns + `definition jsonb` |
| Manual slot | `manual_slots`, primary key = the slot ID |
| Busy block | `busy_blocks` with `owner_type` / `owner_id` |
| Overlap query | `starts_at < :to and ends_at > :from`, exact, no lookback window |
| Host / resource membership | `uuid[]` columns with GIN indexes, queried with `&&` (`.overlaps()`) |

`definition` is jsonb because strategy shapes differ by type and are always read whole. `time_zone` is a
column because it is the one field you will query and audit ("which strategies are pinned to the old
zone?").

## Schema

Tables your app already has (`services`, `locations`, `bookings`, `project_members`) are shown for the
columns the engine reads; add those columns to yours rather than creating new tables.

```sql
-- slots schema. `projects`, `project_members`, `services`, `locations` and
-- `bookings` usually exist already: add the listed columns to yours instead
-- of creating them.

-- Host-owned, shown for the columns the engine reads.
create table if not exists public.project_members (
  project_id uuid not null,
  user_id uuid not null,
  primary key (project_id, user_id)
);

create table if not exists public.services (
  id uuid primary key default gen_random_uuid(),
  project_id uuid not null,
  time_zone text not null default 'UTC',
  host_ids uuid[] not null default '{}',
  location_id uuid,
  resource_ids uuid[] not null default '{}',
  unique (id, project_id)
);

create table if not exists public.locations (
  id uuid primary key default gen_random_uuid(),
  project_id uuid not null,
  working_hours jsonb
);

create table if not exists public.resources (
  id uuid primary key default gen_random_uuid(),
  project_id uuid not null,
  capacity integer not null check (capacity >= 0)
);

create table if not exists public.bookings (
  id uuid primary key default gen_random_uuid(),
  project_id uuid not null,
  service_id uuid not null references public.services (id),
  slot_ids text[] not null default '{}',
  host_ids uuid[] not null default '{}',
  resource_ids uuid[] not null default '{}',
  starts_at timestamptz not null,
  ends_at timestamptz not null,
  seats integer not null default 1 check (seats > 0),
  status text not null default 'PENDING',
  check (ends_at > starts_at)
);
create index if not exists bookings_service_time on public.bookings (service_id, starts_at, ends_at);
create index if not exists bookings_host_ids on public.bookings using gin (host_ids);
create index if not exists bookings_resource_ids on public.bookings using gin (resource_ids);
create index if not exists bookings_slot_ids on public.bookings using gin (slot_ids);

-- Owned by this module.
create table public.slot_strategies (
  id uuid primary key default gen_random_uuid(),
  project_id uuid not null,
  service_id uuid not null,
  strategy_type text not null check (strategy_type in ('RANGE', 'PERIODICITY', 'ONCE', 'DAY_RANGE')),
  -- Pinned authoring zone; never follows later changes to the service's zone.
  time_zone text not null,
  storage_version text not null default 'utc',
  definition jsonb not null,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  -- The composite key makes "a strategy of another project's service" unrepresentable.
  foreign key (service_id, project_id) references public.services (id, project_id) on delete cascade
);
create index slot_strategies_service on public.slot_strategies (service_id);

create table public.manual_slots (
  id text primary key, -- the slot ID clients book with
  project_id uuid not null,
  service_id uuid not null,
  starts_at timestamptz not null,
  ends_at timestamptz not null,
  duration text not null,
  quantity integer not null default 1 check (quantity >= 0),
  labels text[] not null default '{}',
  check (ends_at > starts_at),
  foreign key (service_id, project_id) references public.services (id, project_id) on delete cascade
);
create index manual_slots_service on public.manual_slots (service_id, starts_at);

create table public.busy_blocks (
  id uuid primary key default gen_random_uuid(),
  project_id uuid not null,
  owner_type text not null check (owner_type in ('service', 'host', 'location', 'resource')),
  owner_id uuid not null,
  starts_at timestamptz not null,
  ends_at timestamptz not null,
  source text not null default 'manual', -- 'manual' | 'google' | 'microsoft' | ...
  external_id text,
  description text,
  check (ends_at > starts_at),
  unique (owner_id, source, external_id)
);
create index busy_blocks_owner_time on public.busy_blocks (owner_id, starts_at, ends_at);

-- Row level security: staff of the project manage configuration. Public slot
-- listings are served by the API with a server-side client, never by exposing
-- these tables to anon.
alter table public.slot_strategies enable row level security;
alter table public.manual_slots enable row level security;
alter table public.busy_blocks enable row level security;

create policy slot_strategies_staff on public.slot_strategies
  for all to authenticated
  using (exists (select 1 from public.project_members m
                 where m.project_id = slot_strategies.project_id and m.user_id = (select auth.uid())))
  with check (exists (select 1 from public.project_members m
                      where m.project_id = slot_strategies.project_id and m.user_id = (select auth.uid())));

create policy manual_slots_staff on public.manual_slots
  for all to authenticated
  using (exists (select 1 from public.project_members m
                 where m.project_id = manual_slots.project_id and m.user_id = (select auth.uid())))
  with check (exists (select 1 from public.project_members m
                      where m.project_id = manual_slots.project_id and m.user_id = (select auth.uid())));

create policy busy_blocks_staff on public.busy_blocks
  for all to authenticated
  using (exists (select 1 from public.project_members m
                 where m.project_id = busy_blocks.project_id and m.user_id = (select auth.uid())))
  with check (exists (select 1 from public.project_members m
                      where m.project_id = busy_blocks.project_id and m.user_id = (select auth.uid())));

-- The capacity re-check that makes overbooking impossible. Called by the API
-- after prepareBookingSlots() succeeded; the advisory locks serialize
-- concurrent bookings of the same slot, the sum counts seats, not rows.
create or replace function public.book_slots(
  p_project_id uuid,
  p_service_id uuid,
  p_slots jsonb, -- [{ "slotId": "...", "capacity": 4, "from": "...Z", "to": "...Z" }]
  p_seats integer,
  p_host_ids uuid[] default '{}',
  p_resource_ids uuid[] default '{}'
) returns uuid
language plpgsql
security invoker
set search_path = ''
as $$
declare
  v_slot jsonb;
  v_taken integer;
  v_id uuid;
begin
  -- Lock in a stable order so two multi-slot bookings cannot deadlock.
  for v_slot in select value from jsonb_array_elements(p_slots) order by value->>'slotId' loop
    perform pg_advisory_xact_lock(hashtextextended(v_slot->>'slotId', 0));
    select coalesce(sum(b.seats), 0) into v_taken
      from public.bookings b
     where b.slot_ids @> array[v_slot->>'slotId']
       and b.status in ('PENDING', 'CONFIRMED');
    if v_taken + p_seats > (v_slot->>'capacity')::integer then
      raise exception 'slot-has-not-enough-capacity: %', v_slot->>'slotId';
    end if;
  end loop;

  insert into public.bookings (project_id, service_id, slot_ids, host_ids, resource_ids, starts_at, ends_at, seats)
  select p_project_id, p_service_id,
         array(select value->>'slotId' from jsonb_array_elements(p_slots)),
         p_host_ids, p_resource_ids,
         (select min((value->>'from')::timestamptz) from jsonb_array_elements(p_slots)),
         (select max((value->>'to')::timestamptz) from jsonb_array_elements(p_slots)),
         p_seats
  returning id into v_id;
  return v_id;
end;
$$;
```

What the schema enforces:

- A strategy or manual slot cannot reference another project's service: the
  `(service_id, project_id)` foreign key makes it unrepresentable. Blocks point at several owner types,
  so check the owner's project in the API before inserting one.
- `using` **and** `with check` on every policy, so a row cannot be moved out of a project by update.
- `book_slots` takes a transaction-scoped advisory lock per slot ID (in sorted order, so two multi-slot
  bookings cannot deadlock), sums **seats** of active bookings holding the slot, and inserts only if
  capacity remains. It is `security invoker` with an empty `search_path`.

## The store

```ts
// lib/slots/supabase-store.ts: SlotStore on supabase-js. Server-side only: it
// runs after the host's auth guard, with a client whose RLS role can read
// the service's rows (see supabase-backend.md for the posture).
import type { SupabaseClient } from '@supabase/supabase-js';

import type { BusyInterval, Interval, ResourceCap, WorkingHours } from './availability';
import { normalizeStrategy } from './normalize';
import type { BookableService, SlotStore } from './service';
import type { GeneratedSlot, SlotStrategy } from './types';
import { validateStrategy } from './validate';

const ACTIVE_STATUSES = ['PENDING', 'CONFIRMED'];

interface StrategyRow {
  id: string;
  definition: Record<string, unknown>;
  time_zone: string;
  storage_version: string | null;
}

interface IntervalRow {
  starts_at: string;
  ends_at: string;
}

export function createSupabaseSlotStore(supabase: SupabaseClient): SlotStore {
  return {
    async listStrategies(serviceId) {
      const { data, error } = await supabase
        .from('slot_strategies')
        .select('id, definition, time_zone, storage_version')
        .eq('service_id', serviceId)
        .returns<StrategyRow[]>();
      if (error) throw error;
      return data.map(row => {
        const { strategy } = normalizeStrategy(
          { ...row.definition, timeZone: row.time_zone, storageVersion: row.storage_version ?? undefined },
          { defaultTimeZone: row.time_zone }
        );
        return { ...strategy, strategyId: row.id } as unknown as SlotStrategy;
      });
    },

    async listManualSlots(serviceId) {
      const { data, error } = await supabase
        .from('manual_slots')
        .select('id, starts_at, ends_at, duration, quantity, labels')
        .eq('service_id', serviceId)
        .returns<(IntervalRow & { id: string; duration: string; quantity: number; labels: string[] })[]>();
      if (error) throw error;
      return data.map(
        row =>
          ({
            slotId: row.id,
            strategyId: '',
            from: new Date(row.starts_at),
            to: new Date(row.ends_at),
            duration: row.duration,
            capacity: row.quantity,
            quantity: row.quantity,
            labels: row.labels,
          }) satisfies GeneratedSlot
      );
    },

    async listBusy(service: BookableService, range: Interval) {
      const from = range.from.toISOString();
      const to = range.to.toISOString();

      // Overlap is two comparisons on two indexed columns: starts before the
      // range ends, ends after it starts. No lookback window needed in SQL.
      const [own, hosts, blocks] = await Promise.all([
        supabase
          .from('bookings')
          .select('starts_at, ends_at, seats')
          .eq('service_id', service.serviceId)
          .in('status', ACTIVE_STATUSES)
          .lt('starts_at', to)
          .gt('ends_at', from)
          .returns<(IntervalRow & { seats: number | null })[]>(),
        service.hostIds.length
          ? supabase
              .from('bookings')
              .select('starts_at, ends_at')
              .overlaps('host_ids', service.hostIds)
              .neq('service_id', service.serviceId) // same service is already a BOOKING
              .in('status', ACTIVE_STATUSES)
              .lt('starts_at', to)
              .gt('ends_at', from)
              .returns<IntervalRow[]>()
          : Promise.resolve({ data: [] as IntervalRow[], error: null }),
        supabase
          .from('busy_blocks')
          .select('starts_at, ends_at')
          .in('owner_id', [service.serviceId, ...service.hostIds, ...(service.locationId ? [service.locationId] : [])])
          .lt('starts_at', to)
          .gt('ends_at', from)
          .returns<IntervalRow[]>(),
      ]);
      for (const result of [own, hosts, blocks]) if (result.error) throw result.error;

      return [
        ...(own.data ?? []).map(row => ({ kind: 'BOOKING' as const, ...interval(row), seats: row.seats })),
        ...(hosts.data ?? []).map(row => ({ kind: 'BLOCK' as const, ...interval(row) })),
        ...(blocks.data ?? []).map(row => ({ kind: 'BLOCK' as const, ...interval(row) })),
      ] satisfies BusyInterval[];
    },

    async listResources(resourceIds, range) {
      const [resources, bookings] = await Promise.all([
        supabase.from('resources').select('id, capacity').in('id', resourceIds).returns<{ id: string; capacity: number }[]>(),
        supabase
          .from('bookings')
          .select('starts_at, ends_at, resource_ids')
          .overlaps('resource_ids', resourceIds)
          .in('status', ACTIVE_STATUSES)
          .lt('starts_at', range.to.toISOString())
          .gt('ends_at', range.from.toISOString())
          .returns<(IntervalRow & { resource_ids: string[] })[]>(),
      ]);
      if (resources.error) throw resources.error;
      if (bookings.error) throw bookings.error;
      return resources.data.map(
        resource =>
          ({
            resourceId: resource.id,
            capacity: resource.capacity,
            bookings: bookings.data.filter(row => row.resource_ids.includes(resource.id)).map(interval),
          }) satisfies ResourceCap
      );
    },

    async getWorkingHours(locationId) {
      const { data, error } = await supabase
        .from('locations')
        .select('working_hours')
        .eq('id', locationId)
        .maybeSingle<{ working_hours: WorkingHours | null }>();
      if (error) throw error;
      return data?.working_hours ?? null;
    },
  };
}

/**
 * Create or replace a strategy. The definition is jsonb; the zone and version
 * are columns. `service` comes from your own lookup of the stored row, never
 * from the request body: the (service_id, project_id) foreign key then refuses
 * a strategy filed under another project's service.
 */
export async function saveStrategy(
  supabase: SupabaseClient,
  service: { serviceId: string; projectId: string; timeZone: string },
  strategy: SlotStrategy
): Promise<void> {
  validateStrategy(strategy);
  const { strategyId, timeZone, ...definition } = strategy;
  const { error } = await supabase.from('slot_strategies').upsert({
    id: strategyId,
    project_id: service.projectId,
    service_id: service.serviceId,
    strategy_type: strategy.strategyType,
    time_zone: timeZone ?? service.timeZone,
    storage_version: 'utc',
    definition,
    updated_at: new Date().toISOString(),
  });
  if (error) throw error;
}

function interval(row: IntervalRow): Interval {
  return { from: new Date(row.starts_at), to: new Date(row.ends_at) };
}
```

## Booking write

```ts
const slots = await prepareBookingSlots(store, service, slotIds, seats, new Date());
const { data: bookingId, error } = await supabase.rpc('book_slots', {
  p_project_id: projectId,
  p_service_id: service.serviceId,
  p_slots: slots.map(s => ({ slotId: s.slotId, capacity: s.capacity, from: s.from.toISOString(), to: s.to.toISOString() })),
  p_seats: seats,
  p_host_ids: service.hostIds,
  p_resource_ids: service.resourceIds,
});
if (error?.message.startsWith('slot-has-not-enough-capacity')) {
  // 409: someone took the last seat between listing and booking
}
```

Extend `book_slots` with the host's own booking columns (guest, price, status) rather than inserting
separately: the capacity check and the insert must share one transaction.

## Realtime vs polling

Poll the listing (or refetch on focus) by default; slots change when bookings change, and a stale view
is caught by the capacity check anyway. Subscribe to `postgres_changes` on `bookings` filtered by
`service_id` only when a screen must update within seconds (a kiosk, a front desk), and note it needs a
Supabase JWT on the client.

## Checklist

- [ ] RLS enabled on every module table, policies with `using` and `with check`
- [ ] Anon has no grants on these tables; listings go through the API
- [ ] GIN indexes on `host_ids`, `resource_ids`, `slot_ids`
- [ ] Bookings written only through `book_slots` (or a function built on it)
- [ ] Pooler connection string for serverless hosts
