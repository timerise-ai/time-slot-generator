# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-10-16

First release, extracted from a production booking API's slot engine and hardened.

### Added

- Strategy model (`RANGE`, `PERIODICITY`, `ONCE`, `DAY_RANGE`) with `start`/`end` bounds combining
  absolute instants, daily wall-clock times and offsets from now; `daysOfWeek`, `months`, `tendency`,
  `labels`, `slotsLimit`, `daysLimit`, explicit `overnight` windows and a pinned `timeZone`.
- Dependency-free Intl time helpers with a stated DST gap/overlap policy, independent of the process TZ.
- Generators with chronological emission, local-date iteration, midnight-to-midnight day slots,
  viewport-independent limits, a generation horizon, per-strategy caps and failure isolation.
- Availability: seats, blocks, host busy time, shared resources and date-aware working hours, one map
  for listing and booking, and slot IDs compatible with existing data.
- Firestore and Supabase/Postgres stores, the Postgres schema with RLS, and capacity re-checks inside
  the booking transaction for both.
- Validation with stable error codes and a read-time normalizer for wall-clock-as-UTC storage.
- A 23-test `node:test` suite run under four process time zones.
