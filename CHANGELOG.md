# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.2] - 2026-10-16

Fix release, from scoring the dispatched prompt-1 agent eval runs against 0.1.1.

### Fixed

- The public listing had no viewport by default: `MAX_VIEWPORT_DAYS` applied only when both `from` and `to`
  were sent, and a request with neither listed every slot to the two-year horizon. `from` now defaults to
  now and `to` to `MAX_VIEWPORT_DAYS` after it. Apps built from earlier versions copy the listing route in
  (`references/nextjs-routes.md`).
- Supabase bookings kept no copy of their slots, which the booking path promises. `bookings` gains a `slots`
  jsonb column, `book_slots` fills it, and the booking route passes each slot's `strategyId` and `duration`.
  Apps built from earlier versions add the column and replace the function and the route
  (`references/supabase-backend.md`, `references/nextjs-routes.md`).

### Changed

- `SKILL.md` says to write each block to its path with a script rather than retype it, and the routes
  reference says `runtime` and `dynamic` exports add nothing.
- Two ledger entries for the eval findings (`references/provenance.md`, now 32 entries).

## [0.1.1] - 2026-10-16

Security fix release, from scoring the prompt-1 agent eval runs against 0.1.0.

### Security

- The anon key could book any number of seats. Postgres lets PUBLIC execute a new function and Supabase grants
  anon on every table in `public`, so `book_slots` was callable with a forged capacity and the host tables the
  schema creates were readable and writable with the public key. The schema now enables RLS on every table it
  creates and grants `book_slots` to `service_role` only. Apps built from 0.1.0 on Supabase apply the new
  `alter table ... enable row level security` lines and the `revoke` and `grant` on `book_slots` to their
  database (`references/supabase-backend.md`).

### Fixed

- The capacity re-check counted bookings by slot ID while the listing counts by overlap, so overlapping slots
  of two strategies could both sell the last seat. `book_slots` takes one lock per service and sums the seats
  of overlapping bookings; `assertCapacityInTransaction` takes the service ID and does the same. Apps built
  from 0.1.0 copy both in (`references/supabase-backend.md`, `references/firestore-backend.md`).
- Blackouts on resources were never read. Both stores include the service's resources among the block owners.

### Changed

- The quick start says to copy every block as written, to install Vitest and the backend SDK from the
  package registry, to keep tests of one's own apart from the shipped suite, and lists what the handover
  carries (`SKILL.md`, `references/testing.md`, `references/adaptation.md`).
- Three ledger entries for the eval findings (`references/provenance.md`, now 30 entries).

## [0.1.0] - 2026-10-16

First release: the slot engine of a booking platform, written by the engineer who builds it and audited
against the earlier implementation.

### Added

- Strategy model (`RANGE`, `PERIODICITY`, `ONCE`, `DAY_RANGE`) with `start` and `end` bounds combining
  absolute instants, daily wall-clock times and offsets from now; `daysOfWeek`, `months`, `tendency`,
  `labels`, `slotsLimit`, `daysLimit`, explicit `overnight` windows and a pinned `timeZone`.
- Dependency-free Intl time helpers with a stated DST gap and overlap policy, independent of the
  process time zone.
- Generators with chronological emission, local-date iteration, midnight-to-midnight day slots,
  viewport-independent limits, a generation horizon, per-strategy caps and failure isolation.
- Availability: seats, blocks, host busy time, shared resources and date-aware working hours, one map
  for listing and booking, and slot IDs compatible with existing data.
- Firestore and Supabase/Postgres stores, the Postgres schema with RLS, and capacity re-checks inside
  the booking transaction for both.
- Next.js App Router route handlers for the listing and the booking on Supabase, with credentials read at
  request time and errors answered by code (`references/nextjs-routes.md`).
- The seam contract, rename table and order of work (`references/adaptation.md`).
- Validation with stable error codes and a read-time normalizer for wall-clock-as-UTC storage.
- A 23-test `node:test` suite run under four process time zones, wired to `npm test` under Vitest in the
  app.
