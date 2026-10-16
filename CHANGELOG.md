# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
