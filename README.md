# time-slot-generator

[![Agent Skills](https://img.shields.io/badge/Agent_Skills-open_format-059669)](https://agentskills.io)
[![skills.sh](https://img.shields.io/badge/skills.sh-npx_skills_add-059669)](https://www.skills.sh)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-059669)](https://docs.claude.com/en/docs/claude-code/skills)
[![Codex CLI](https://img.shields.io/badge/Codex_CLI-compatible-059669)](https://developers.openai.com/codex/skills)
[![Gemini CLI](https://img.shields.io/badge/Gemini_CLI-compatible-059669)](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/skills.md)
[![Eval claude-opus-5-5](https://img.shields.io/badge/eval_claude--opus--5--5-pass_8%2F8-059669)](evals/2026-10-16-claude-code-p1-5.md)
[![Eval gpt-6.1-sol](https://img.shields.io/badge/eval_gpt--6.1--sol-pass_8%2F8-059669)](evals/2026-10-16-codex-p1-5.md)
[![Eval gemini-3.8-flash](https://img.shields.io/badge/eval_gemini--3.8--flash-pass_8%2F8-059669)](evals/2026-10-16-gemini-cli-p1-5.md)

An [Agent Skill](https://agentskills.io) that teaches an agent to build a **strategy-driven time-slot
generator with availability** in a **Next.js App Router** app: slots generated from strategies (daily
ranges, Nth weekday of the month, one-off slots, whole days), authored in local wall-clock time and stored as
UTC instants, with seats, bookings, blackouts, hosts' other commitments, shared resources and working hours
subtracted, served by a listing route and a booking route. Firestore and Supabase/Postgres are both carried.

Slots are never stored. Strategies are, slots are regenerated on every read, and a booking finds its slot
again by a deterministic ID. So the generator is **a pure function whose output does not depend on the
viewport, the server's time zone or the time of year**, and the listing and the booking read **one
availability map**, with seats **re-counted inside the booking transaction**.

This skill was written by the engineer who builds the slot engine of a booking platform, and audited
against the earlier implementation of that engine. The templates hold the properties a slot engine has to
hold: every weekday of a monthly rule appears in any viewport; local days survive both DST changes; limits
count from the strategy, not the requested week; output is the same under any process time zone; nothing
the listing hides can be booked by ID; and on ordinary windows the slot IDs are the ones existing bookings
already hold. A 23-test suite run under four process time zones states each one, and
[`references/provenance.md`](references/provenance.md) has the record.

## Install

One command, via the [skills.sh](https://www.skills.sh) CLI, which installs the skill into every
skills-compatible agent it detects, including Claude Code, Codex CLI and Gemini CLI:

```bash
npx skills add timerise-ai/time-slot-generator
```

Name the agents instead with `-a`, for example
`npx skills add timerise-ai/time-slot-generator -a claude-code -a codex`.

### Manual install

Nothing here is Claude-specific: the skill is a plain [Agent Skills](https://agentskills.io) folder,
`SKILL.md` plus markdown references with no file that calls a model, so cloning it into an agent's skills
directory is all an install is. For Claude Code:

```bash
git clone https://github.com/timerise-ai/time-slot-generator.git ~/.claude/skills/time-slot-generator
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. For another
agent, clone into that agent's skills directory, or symlink the Claude Code copy so one `git pull` updates
every agent:

```bash
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/time-slot-generator ~/.agents/skills/time-slot-generator
```

Update with `git pull` in the clone. The current release is **0.1.3**. See [CHANGELOG.md](CHANGELOG.md).

## Activation

The skill activates on tasks such as "generate bookable slots from our opening hours", "offer the first
Friday of every month", "our slots are an hour off after the clock change", or "two people booked the last
seat". Each host matches skills its own way, so on a first run invoke it explicitly: `/time-slot-generator`
in Claude Code, `$time-slot-generator` in Codex CLI, and through `/skills` in Gemini CLI.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: trigger, architecture, adaptation contract, critical facts, hard rules, quick start, reference directory |
| `references/adaptation.md` | Host prerequisites, seam contract, rename table, integration points, order of work, the non-negotiables |
| `references/strategy-model.md` | Strategy types, bounds, limits, types, validation |
| `references/timezones.md` | UTC and wall clock, the DST policy, Intl helpers, the legacy normalizer |
| `references/generators.md` | The engine, slot identity, horizon, cost |
| `references/availability.md` | Bookings, blocks, resources, working hours, the service entry point, the booking path |
| `references/supabase-backend.md` | Schema, RLS, the store, `book_slots` |
| `references/firestore-backend.md` | Collections, indexes, the store, the transaction capacity check |
| `references/nextjs-routes.md` | The server client, the listing and booking route handlers, error codes, environment |
| `references/testing.md` | The suite, running it under four time zones, the differential check |
| `references/provenance.md` | The audit ledger: fixed, kept deliberately, designed here |
| `evals/` | `prompts.md` with what an operator types, and one file per agent eval run |
| `.github/workflows/agent-eval.yml` | Runs the agent evals on every published release |

The seam is the `SlotStore` interface and the `BookableService` the routes read from the host's own
`services` record. They bound everything the host app supplies: its database, its tenancy and its auth
guard. The engine files import neither Next.js nor a database SDK.

## The five non-negotiables

1. **Never change the slot ID input.** `md5("<utcStartISO>;<duration>;<strategyId>")` is the identity every
   client and every booking holds; a different input orphans them all. A test pins the format.
2. **Never infer "overnight" from end before start.** A reversed range is far more often a typo than a night
   shift, so an overnight window needs `overnight: true`. The validation test refuses 17:00 to 09:00 and
   accepts the explicit overnight window.
3. **Never generate without a horizon.** `maxHorizonDays` and `maxSlots` cap every strategy, since one with
   no end would otherwise generate until 2038. A test stops a strategy at a 10-day horizon, then at
   `maxSlots`, and expects `truncated` only in the second case.
4. **Never filter availability in the listing only.** The listing and the booking path read one
   `buildAvailability` map, so an ID cannot book what the listing hides. Both entry points in `service.ts`
   call the same `computeAvailability`, and a test resolves booking IDs against that map.
5. **Never count a host's booking of this service as host busy time.** It already subtracts seats as a
   `BOOKING`; blocking on it as well hides a slot that still has room. Both stores exclude the service from
   the host query, and `SlotStore.listBusy` states the rule.

Everything else, the words, the screens, the guard and the store, is the host app's.

## Requirements

Next.js App Router with route handlers on the Node.js runtime, TypeScript, and `@supabase/supabase-js` or
`firebase-admin`. No date library: the engine runs on `Intl` alone. Vitest, or Node 22.18 or later, for the
suite.

## Verification

The TypeScript blocks compile as one project under `strict` and `noUncheckedIndexedAccess`, the routes build
with `next build` in an empty Next.js app with no credentials set, and the suite in
[testing.md](references/testing.md) passes under `TZ=UTC`, `Europe/Warsaw`, `Pacific/Auckland` and
`America/Los_Angeles`. The schema was applied to a scratch Postgres with Supabase's default grants, where
the anon role read no service and was refused `book_slots`, and `book_slots` refused a second booking of a
full slot, and of an overlapping slot with another ID, with `slot-has-not-enough-capacity`.

## Not this

| Problem | Go to |
|---|---|
| Booking screens, a walk-up terminal | `booking-kiosk` |
| Public scheduling links per host, calendar sync, e-mail invites | `calendar-scheduling-links` |
| Ticketed events on one date, deposits, no-show holds | `bookable-events` |
| Invoices for bookings | `invoicing` |
| Calendar recurrence (RRULE, iCalendar event series) | `rrule` or the calendar provider's API |

## Contributing

Issues and pull requests are welcome here. Pure markdown, with no build step, but the code blocks are checked:
every block names its destination on the first line, every TypeScript block is written to compile as one
project under `strict` and `noUncheckedIndexedAccess`, and the suite passes under four process time zones.
Claims in this skill are meant to be verifiable: if you change a factual claim, say how you verified it,
whether against the ECMAScript `Intl` specification, the IANA time zone database, the Firestore or Supabase
documentation, the Next.js documentation, or a reproduction.

Adding, removing or renaming a file in `references/` means updating the quick start and the reference
directory table in `SKILL.md`, the file table above, and any relative cross-links. Every odd-looking part of
the templates is there for a reason, and `references/provenance.md` is the ledger that must stay truthful:
read it before simplifying anything, and add an entry for anything you change. Commits follow Conventional
Commits and releases follow [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) in the
index; `CLAUDE.md` carries the full editing conventions.

## Part of the Timerise Skills

This is one of the [Timerise Skills](https://github.com/timerise-ai/skills): modules for **Next.js App
Router** apps written by our own senior engineers from the modules they have shipped, not synthetic, each
published as its own repository and indexed there. They share one layout, so an agent that has read one knows
how to read the next: a `SKILL.md` entry point, `references/` loaded on demand, and a seam contract carrying
the module's non-negotiables.

## Author

Built and maintained by [Timerise](https://timerise.ai).

## License

MIT. See [LICENSE](LICENSE).
