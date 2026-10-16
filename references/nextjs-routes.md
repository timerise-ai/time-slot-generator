# Next.js routes

How the engine is served from a Next.js App Router app: one route lists slots, one books them, both on the
Supabase store with a server-side client. The engine files in `lib/slots/` are plain TypeScript and do not
import Next.js; only the files in this reference do. On Firestore, swap the client and the store for
`createFirestoreSlotStore` and the transaction in [firestore-backend.md](firestore-backend.md); the routes
keep their shape.

## Runtime

- **Node.js runtime.** `slot-id.ts` hashes with `node:crypto`, which the Edge runtime does not have. Route
  handlers run on Node.js by default; do not set `runtime = 'edge'` on these routes without swapping in a
  pure md5 (see the seam contract in [adaptation.md](adaptation.md)).
- **Credentials at request time.** The client is created on the first request, not at import, so
  `next build` passes without `SUPABASE_URL` or `SUPABASE_SERVICE_ROLE_KEY` set.
- **The service comes from storage.** The routes take a `serviceId` and read the zone, hosts, location,
  resources and project from the `services` row. Nothing that scopes a query is read from the request.
- **Errors keep their codes.** Engine errors are `Error('<code>: <detail>')`; the routes answer with the
  code, never the detail, and map it to a status in one place.
- **Nothing to add.** Route handlers that read the request are dynamic and on Node.js already, so
  `runtime` and `dynamic` exports change nothing; the listing already defaults and bounds its viewport. The
  one documented variant is a `serviceId` format check, below.

## Server client and service lookup

```ts
// lib/slots/server.ts: the Supabase client, the store and the service lookup the routes share.
import { createClient, type SupabaseClient } from '@supabase/supabase-js';

import type { BookableService, SlotStore } from './service';
import { createSupabaseSlotStore } from './supabase-store';

let client: SupabaseClient | null = null;

/**
 * Server-side only. The service-role key bypasses RLS, so every query the
 * store makes is scoped by a service the route resolved from the database.
 */
export function getServiceClient(): SupabaseClient {
  if (client) return client;
  const url = process.env.SUPABASE_URL;
  const key = process.env.SUPABASE_SERVICE_ROLE_KEY;
  if (!url || !key) throw new Error('slots-not-configured: SUPABASE_URL and SUPABASE_SERVICE_ROLE_KEY are required');
  client = createClient(url, key, { auth: { persistSession: false, autoRefreshToken: false } });
  return client;
}

export function getSlotStore(): SlotStore {
  return createSupabaseSlotStore(getServiceClient());
}

export interface StoredService extends BookableService {
  projectId: string;
}

interface ServiceRow {
  id: string;
  project_id: string;
  time_zone: string;
  host_ids: string[];
  location_id: string | null;
  resource_ids: string[];
}

export async function loadBookableService(supabase: SupabaseClient, serviceId: string): Promise<StoredService> {
  const { data, error } = await supabase
    .from('services')
    .select('id, project_id, time_zone, host_ids, location_id, resource_ids')
    .eq('id', serviceId)
    .maybeSingle<ServiceRow>();
  if (error) throw error;
  if (!data) throw new Error(`service-not-found: ${serviceId}`);
  return {
    serviceId: data.id,
    projectId: data.project_id,
    timeZone: data.time_zone,
    hostIds: data.host_ids,
    locationId: data.location_id,
    resourceIds: data.resource_ids,
  };
}

const STATUS_BY_CODE: Record<string, number> = {
  'request-invalid': 400,
  'service-not-found': 404,
  'slot-no-longer-available': 409,
  'slot-has-not-enough-capacity': 409,
};

/** Maps an engine or store error to a response that carries the code only. */
export function errorResponse(error: unknown): Response {
  const message = error instanceof Error ? error.message : '';
  const code = /^([a-z]+(?:-[a-z]+)+):/.exec(message)?.[1];
  const status = code ? STATUS_BY_CODE[code] : undefined;
  if (code && status) return Response.json({ error: code }, { status });
  console.error(error);
  return Response.json({ error: 'internal-error' }, { status: 500 });
}
```

The `services` columns this reads are the ones [supabase-backend.md](supabase-backend.md) lists. A UUID
that is not well formed reaches Postgres and fails there, which `errorResponse` answers with 500; validate
`serviceId` against the host's ID format if it wants a 400 instead.

## Listing

`GET /api/slots?serviceId=<id>&from=<ISO>&to=<ISO>`. `from` and `to` are the viewport; the answer does not
depend on them beyond which slots are in it, because limits count from the strategy start
([generators.md](generators.md)). `from` defaults to now and `to` to `MAX_VIEWPORT_DAYS` after `from`, so a
request without them is bounded too; the engine alone would list every slot up to the horizon.

```ts
// app/api/slots/route.ts: the listing. Public, so it reads only what a guest may see.
import type { NextRequest } from 'next/server';

import { errorResponse, getServiceClient, getSlotStore, loadBookableService } from '@/lib/slots/server';
import { listServiceSlots } from '@/lib/slots/service';

const MAX_VIEWPORT_DAYS = 62;

function instant(value: string | null, name: string): Date | undefined {
  if (value === null) return undefined;
  const date = new Date(value);
  // Only full instants: a bare date or a local time is read in the server's zone.
  if (!/(Z|[+-]\d\d:\d\d)$/.test(value) || Number.isNaN(date.getTime())) {
    throw new Error(`request-invalid: ${name} must be an ISO instant with an offset`);
  }
  return date;
}

export async function GET(request: NextRequest): Promise<Response> {
  try {
    const params = request.nextUrl.searchParams;
    const serviceId = params.get('serviceId');
    if (!serviceId) throw new Error('request-invalid: serviceId is required');
    // Always a bounded viewport: without `to` the engine would list up to the horizon.
    const now = new Date();
    const from = instant(params.get('from'), 'from') ?? now;
    const to = instant(params.get('to'), 'to') ?? new Date(from.getTime() + MAX_VIEWPORT_DAYS * 86_400_000);
    if (to <= from || to.getTime() - from.getTime() > MAX_VIEWPORT_DAYS * 86_400_000) {
      throw new Error(`request-invalid: to must follow from by at most ${MAX_VIEWPORT_DAYS} days`);
    }

    const service = await loadBookableService(getServiceClient(), serviceId);
    const result = await listServiceSlots(getSlotStore(), service, { now, from, to });
    return Response.json(
      { timeZone: service.timeZone, slots: result.slots, truncated: result.truncated },
      { headers: { 'Cache-Control': 'no-store' } }
    );
  } catch (error) {
    return errorResponse(error);
  }
}
```

`failed` is left out of the public answer: it names strategies and their errors, which are for staff. Log
it, or return it from a staff route.

## Booking

`POST /api/bookings` with `{ "serviceId": "...", "slotIds": ["..."], "seats": 1 }`. One generation, then the
capacity re-check inside `book_slots` ([availability.md](availability.md), the booking path).

```ts
// app/api/bookings/route.ts: the booking path. The host's guard runs first.
import { errorResponse, getServiceClient, getSlotStore, loadBookableService } from '@/lib/slots/server';
import { prepareBookingSlots } from '@/lib/slots/service';

const MAX_SLOTS_PER_BOOKING = 50;

interface BookingRequest {
  serviceId: string;
  slotIds: string[];
  seats: number;
}

function parseBody(body: unknown): BookingRequest {
  const value = (body ?? {}) as Partial<Record<keyof BookingRequest, unknown>>;
  const { serviceId, slotIds, seats = 1 } = value;
  if (typeof serviceId !== 'string' || serviceId === '') throw new Error('request-invalid: serviceId');
  if (
    !Array.isArray(slotIds) ||
    slotIds.length === 0 ||
    slotIds.length > MAX_SLOTS_PER_BOOKING ||
    !slotIds.every((id): id is string => typeof id === 'string')
  ) {
    throw new Error('request-invalid: slotIds');
  }
  if (typeof seats !== 'number' || !Number.isInteger(seats) || seats < 1) throw new Error('request-invalid: seats');
  return { serviceId, slotIds: [...new Set(slotIds)], seats };
}

export async function POST(request: Request): Promise<Response> {
  try {
    // The host's auth guard goes here when bookings need a signed-in guest.
    const input = parseBody(await request.json().catch(() => null));
    const supabase = getServiceClient();
    const { projectId, ...service } = await loadBookableService(supabase, input.serviceId);

    const slots = await prepareBookingSlots(getSlotStore(), service, input.slotIds, input.seats, new Date());
    const { data: bookingId, error } = await supabase.rpc('book_slots', {
      p_project_id: projectId,
      p_service_id: service.serviceId,
      p_slots: slots.map(slot => ({
        slotId: slot.slotId,
        capacity: slot.capacity,
        strategyId: slot.strategyId,
        duration: slot.duration,
        from: slot.from.toISOString(),
        to: slot.to.toISOString(),
      })),
      p_seats: input.seats,
      p_host_ids: service.hostIds,
      p_resource_ids: service.resourceIds,
    });
    if (error) throw new Error(error.message);

    return Response.json(
      {
        bookingId: bookingId as string,
        slots: slots.map(slot => ({ slotId: slot.slotId, dateTimeFrom: slot.from.toISOString(), dateTimeTo: slot.to.toISOString() })),
      },
      { status: 201 }
    );
  } catch (error) {
    return errorResponse(error);
  }
}
```

`book_slots` raises `slot-has-not-enough-capacity: <slotId>` when the last seat went between listing and
booking; `errorResponse` reads the code from the message and answers 409, the same as when
`prepareBookingSlots` finds the slot already full.

## Environment

| Variable | Where | Notes |
|---|---|---|
| `SUPABASE_URL` | Server only | The project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Server only | Never `NEXT_PUBLIC_`; it bypasses RLS |

## Checklist

- [ ] Both routes on the Node.js runtime, or a pure md5 behind `SlotIdHasher`
- [ ] No credential read at import; `next build` passes with none set
- [ ] Service, project, hosts and resources read from the `services` row, never from the request
- [ ] `from` and `to` accepted only as instants with an offset, the viewport bounded
- [ ] Responses carry error codes only; `failed` stays off the public listing
- [ ] Bookings written only through `book_slots`
